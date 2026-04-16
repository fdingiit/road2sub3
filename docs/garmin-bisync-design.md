# Garmin CN / Global 双向活动同步设计文档（文件持久化版）

## 1. 设计目标
- 复用 `running_page` 的 Garmin 同步能力，最小化二次开发。
- 实现 CN 与 Global 两账号活动双向 merge + 补齐。
- 使用 daemon 常驻运行并周期调度。
- 在“仅文件持久化”约束下保证幂等、可恢复、可审计。

## 2. 总体架构

```text
+--------------------------- daemon process ----------------------------+
|                                                                       |
|  Scheduler (interval loop)                                            |
|      -> SyncEngine.sync_once()                                        |
|            -> CN Connector (running_page Garmin CN)                   |
|            -> Global Connector (running_page Garmin COM)              |
|            -> Normalizer + Fingerprinter                              |
|            -> Matcher / Diff Engine                                   |
|            -> Upload Executor (CN->Global, Global->CN)                |
|            -> FileStateStore (state/index/mapping/failures/logs)      |
|                                                                       |
+-----------------------------------------------------------------------+
```

## 3. 复用点（running_page）

- `run_page/garmin_sync.py`
  - Garmin 客户端初始化（含 CN/COM 域名支持）
  - 活动拉取、活动文件下载、活动上传能力
- `run_page/garmin_sync_cn_global.py`
  - 单向同步流程可作为双向编排模板

说明：本方案不改造 Garmin API 细节，仅新增编排与状态管理层。

## 4. 模块设计

### 4.1 Scheduler
- 采用标准库 `time.sleep()` 的间隔轮询模式。
- 每轮触发 `sync_once()`。
- 任何异常都被记录，不导致主进程退出。
- 单轮内部采用“阶段内并行、阶段间有序”的执行模型（见 6.2）。

### 4.2 Connector
- `CNConnector`: 使用 CN 域名配置登录并拉取/上传。
- `GlobalConnector`: 使用 COM 域名配置登录并拉取/上传。
- CN 与 Global 两侧拉取相互独立，可并发调用。

### 4.3 Normalizer
统一活动结构：

```json
{
  "source": "cn|global",
  "activity_id": "string",
  "sport_type": "running",
  "start_time_utc": "2026-04-16T08:30:00Z",
  "duration_sec": 3600,
  "distance_meter": 10000,
  "file_path": "cache/xxx.fit"
}
```

### 4.4 Fingerprinter
指纹计算（建议）：

`sha1(sport_type + rounded_start_time + rounded_duration + rounded_distance)`

容差建议：
- start_time ±120 秒
- duration ±5%
- distance ±2%

### 4.5 Matcher / Diff Engine
- 从 CN/Global 索引文件加载指纹集合。
- 计算：
  - `cn_only = cn_fp - global_fp`
  - `global_only = global_fp - cn_fp`
- 输出两个上传任务列表。

### 4.6 Upload Executor
- CN -> Global 与 Global -> CN 两个方向可并发执行（每方向内部可限制并发度）。
- 单条任务状态：`pending/success/failed`。
- 成功写映射，失败写重试队列。

### 4.7 FileStateStore
负责所有文件读写、原子提交、锁控制。

### 4.8 Parallel Runtime（并发运行时）
- 使用标准库 `concurrent.futures.ThreadPoolExecutor` 实现 I/O 并发。
- 推荐配置：
  - `fetch_workers=2`（CN/Global 各 1）
  - `upload_workers=4~8`（按接口限流能力调参）
  - `retry_workers=2`
- 通过有界任务队列限制内存和请求洪峰。

## 5. 持久化文件设计

根目录：`sync_state/`

```text
sync_state/
  config.json
  state.json
  index_cn.jsonl
  index_global.jsonl
  mapping.jsonl
  failed_queue.jsonl
  .lock
  logs/
```

### 5.1 config.json
运行配置：
- `interval_sec`
- `lookback_days`
- `retry_max`
- `retry_backoff_sec`
- `sync_private_activities`
- `fetch_workers`
- `upload_workers`
- `retry_workers`
- `max_inflight_tasks`

### 5.2 state.json
全局游标与轮次状态：
- `last_success_at`
- `last_cn_cursor`
- `last_global_cursor`
- `last_run_status`
- `version`

### 5.3 index_cn.jsonl / index_global.jsonl
每行一个活动索引条目，含 `fingerprint` 与核心字段。

### 5.4 mapping.jsonl
记录双向同步映射：
- `fp`
- `src_region/src_activity_id`
- `dst_region/dst_activity_id`
- `synced_at`

### 5.5 failed_queue.jsonl
失败任务队列：
- `fp`
- `direction`
- `retry_count`
- `next_retry_at`
- `last_error`

## 6. 关键流程

### 6.1 启动流程
1. 创建并校验 `sync_state/`。
2. 获取 `.lock` 文件锁（若已有锁则退出）。
3. 加载配置与状态。
4. 启动调度循环。

### 6.2 单轮同步流程（sync_once）
1. **并发拉取阶段**：并行拉取 CN 和 Global 活动（增量 + 回看窗口）。
2. **计算阶段**：标准化并生成指纹，更新两侧索引，计算双向差集。
3. **并发上传阶段**：并行执行 CN -> Global 与 Global -> CN；方向内采用 worker 池并发上传。
4. **并发重试阶段**：并行处理到期失败任务（受 `retry_workers` 限制）。
5. **提交阶段**：汇总结果，原子更新 `state.json`，输出轮次日志。

### 6.3 原子写流程
1. 写入 `target.tmp`
2. `flush + fsync`
3. `rename(target.tmp, target)`

## 7. 幂等与一致性策略
- 以 `fingerprint` 作为主幂等键。
- 上传前先查 `mapping.jsonl` 与目标索引；已存在则跳过。
- 同步失败不会回滚已成功条目，依靠后续轮次最终收敛。
- 并发写入 `mapping/failed/index` 时采用“内存聚合 + 单写线程提交”策略，避免多线程直接竞争文件句柄。
- 任务 claim 机制：上传前先在内存态标记 `inflight(fp,direction)`，防止同轮重复分发。

## 8. 异常处理策略
- API 临时失败：指数退避重试（上限可配）。
- 单条上传失败：入失败队列，不阻断整轮。
- 文件损坏：检测 JSON 解析失败并自动备份坏文件后重建。
- 进程崩溃：依赖 `state.json` + `mapping.jsonl` 恢复。
- 并发子任务异常隔离：单 worker 失败不影响其他 worker，统一在轮次汇总阶段收敛。

## 9. 日志与可观测性
- 日志级别：INFO/WARN/ERROR。
- 每轮关键指标：
  - `fetched_cn`, `fetched_global`
  - `cn_only_count`, `global_only_count`
  - `upload_success`, `upload_failed`
  - `retry_success`, `retry_failed`
  - `duration_ms`

## 10. 配置建议
- `interval_sec`: 900（15分钟）
- `lookback_days`: 30
- `retry_max`: 5
- `retry_backoff_sec`: [60, 300, 900, 1800, 3600]
- `fetch_workers`: 2
- `upload_workers`: 6
- `retry_workers`: 2
- `max_inflight_tasks`: 200

## 11. 测试与验证方案
1. **功能测试**：构造单边新增活动，验证双向补齐。
2. **幂等测试**：同一数据重复跑 3 轮，验证无重复上传。
3. **恢复测试**：在上传中途 kill 进程，重启后验证可继续。
4. **失败重试测试**：模拟接口失败，验证队列与退避策略。
5. **并发正确性测试**：开启多 worker，验证无重复上传、无状态文件损坏。
6. **性能测试**：对比串行与并发方案下轮次耗时与吞吐。
7. **详细用例清单**：见 `docs/garmin-bisync-test-plan.md`（包含单测、集成、夹具与门禁建议）。

## 12. 里程碑建议
- M1：完成 `sync_once` 双向流程 + 指纹判重（手动触发）。
- M2：完成文件状态层（mapping/failed/state）+ 幂等。
- M3：完成 daemon 调度 + 日志 + 恢复机制。
- M4：灰度运行与参数调优。

## 13. 评审决策输入
- 评审会建议使用 `docs/garmin-bisync-review-checklist.md` 逐条拍板。
- 设计中的默认参数仅作为起点，最终以评审拍板值为准。
