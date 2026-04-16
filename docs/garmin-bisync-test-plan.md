# Garmin CN / Global 双向同步测试设计（单测 + 集成）

## 1. 测试目标
- 验证双向同步正确性：CN/Global 两侧最终一致。
- 验证幂等：重复运行不重复上传。
- 验证并发安全：并发拉取/上传下状态文件不损坏、任务不重复分发。
- 验证可恢复：异常退出后可基于文件状态恢复。

## 2. 测试分层

### 2.1 单元测试（Unit Test）
**目标**：隔离模块行为，快速定位逻辑缺陷。  
**覆盖对象**：
1. `fingerprinter`：指纹稳定性、容差匹配。
2. `normalizer`：不同来源活动字段归一化。
3. `diff_engine`：`cn_only` / `global_only` 差集计算。
4. `file_state_store`：原子写、锁、JSONL 读写、compact。
5. `retry_policy`：指数退避、重试上限、到期判定。
6. `task_claim`：并发下 `inflight(fp, direction)` 唯一性。

### 2.2 集成测试（Integration Test）
**目标**：验证 `sync_once` 端到端流程编排。  
**覆盖对象**：
1. 并发拉取阶段（双 connector）。
2. 计算阶段（normalize/fingerprint/diff）。
3. 并发上传阶段（双向 + worker pool）。
4. 失败重试阶段。
5. 最终提交（`state.json` 原子更新 + mapping/fail queue 更新）。

### 2.3 合同测试（Contract Test，可选）
对接 `running_page` 接口适配层：
- 模拟 `garmin_sync.py` 返回结构，校验适配器不因字段变化崩溃。
- 若上游 API 字段变更，测试应第一时间失败并提示映射点。

## 3. 单元测试设计明细

## 3.1 fingerprinter

### 用例 U-FP-01（稳定性）
- 输入：同一活动对象重复计算 100 次。
- 断言：输出 `fingerprint` 完全一致。

### 用例 U-FP-02（容差内等价）
- 输入：同一活动，时间偏移 90 秒、距离偏移 1%、时长偏移 3%。
- 断言：判定为同一活动（等价匹配）。

### 用例 U-FP-03（容差外不等价）
- 输入：时间偏移 300 秒或距离偏移 10%。
- 断言：判定为不同活动。

## 3.2 diff_engine

### 用例 U-DF-01（基础差集）
- 输入：
  - CN: {A, B, C}
  - Global: {B, C, D}
- 断言：
  - `cn_only={A}`
  - `global_only={D}`

### 用例 U-DF-02（空集边界）
- 输入：两侧均空。
- 断言：差集均空，不报错。

## 3.3 file_state_store

### 用例 U-FS-01（原子写）
- 场景：写 `state.json` 时中断（模拟写 tmp 后 crash）。
- 断言：重启后要么保留旧文件，要么完整新文件，不出现半文件。

### 用例 U-FS-02（并发提交）
- 场景：多个线程提交 mapping/failure 批次。
- 断言：最终文件行数正确、无截断、无 JSON 损坏。

### 用例 U-FS-03（锁机制）
- 场景：两个进程同时启动。
- 断言：仅一个获得 `.lock`，另一个退出并返回明确错误码。

## 3.4 retry_policy

### 用例 U-RT-01（指数退避）
- 输入：重试次数 0..5。
- 断言：`next_retry_at` 递增并符合配置数组。

### 用例 U-RT-02（上限熔断）
- 输入：失败次数超过 `retry_max`。
- 断言：任务转入 dead-letter（或终态失败）且不再调度。

## 3.5 task_claim

### 用例 U-CL-01（同轮去重）
- 场景：并发提交同一 `(fp,direction)`。
- 断言：仅 1 个任务成功 claim。

### 用例 U-CL-02（轮次释放）
- 场景：轮次结束后清理 inflight。
- 断言：下一轮允许重新调度失败任务。

## 4. 集成测试场景设计

## 4.1 I-E2E-01 双向补齐
- 初始：CN={A,B}, Global={B,C}
- 执行：`sync_once`
- 期望：执行后两侧均包含 {A,B,C}（考虑上传异步完成后在下轮收敛也可接受，需在测试中明确等待策略）。

## 4.2 I-E2E-02 幂等验证
- 初始：两侧已一致。
- 执行：连续运行 3 轮。
- 期望：上传调用次数为 0（或仅校验调用被短路）。

## 4.3 I-E2E-03 并发上传冲突
- 初始：构造多个近似活动 + 并发 worker。
- 执行：高并发上传。
- 期望：无重复 mapping、无重复上传、状态文件可解析。

## 4.4 I-E2E-04 失败重试收敛
- 初始：注入可恢复错误（前 2 次失败，第 3 次成功）。
- 执行：多轮调度。
- 期望：任务先进入 `failed_queue`，后续轮次成功并从队列移除。

## 4.5 I-E2E-05 崩溃恢复
- 初始：同步进行到“上传成功但尚未提交 state”。
- 执行：模拟进程崩溃并重启。
- 期望：不出现批量重复上传，最终收敛一致。

## 5. 测试夹具（Fixtures）与 Mock 策略

## 5.0 测试数据来源（回答“单测数据从哪来”）
单测与集成测试的数据来源按优先级分 4 类：

1. **手工构造数据（默认）**  
   - 直接在测试代码内定义最小活动样本（dict/JSON），覆盖正常与边界字段。  
   - 用于 `fingerprinter`、`diff_engine`、`retry_policy` 等纯逻辑单测。  

2. **固化样例文件（推荐）**  
   - 在仓库 `tests/fixtures/` 维护脱敏后的 JSON/JSONL/FIT 样本：  
     - `activities_cn_sample.json`  
     - `activities_global_sample.json`  
     - `index_cn_sample.jsonl` / `index_global_sample.jsonl`  
   - 用于 `normalizer`、`file_state_store`、`sync_once` 集成测试。  

3. **录制回放数据（可选）**  
   - 使用一次真实调用结果进行脱敏后落盘（移除账号、设备 ID、精确地理坐标）。  
   - 用于合同测试，验证与 `running_page` 适配层的字段兼容。  

4. **性质测试生成数据（可选）**  
   - 通过随机生成器批量构造活动（时间/距离/时长扰动）验证容差与并发稳健性。  
   - 适用于并发与幂等压力测试。

> 结论：**单测首选“手工构造 + fixtures 固化样本”**，不依赖线上 Garmin 真实数据与网络环境。

## 5.1 Garmin Mock Connector
- 以本地假实现替代真实 Garmin 网络调用。
- 提供可编排行为：成功、超时、429、5xx、部分成功。
- 记录调用历史用于断言（上传次数、参数）。

## 5.2 文件系统隔离
- 每个用例使用独立临时目录（如 `tmp/test_case_xxx/sync_state`）。
- 测试结束后自动清理，避免互相污染。

## 5.3 时间控制
- 注入 `clock`（可 mock 的 now()）来稳定重试与游标断言。

## 5.4 样本数据维护规范
- 样本必须脱敏（邮箱、账号 ID、设备 ID、经纬度）。  
- 每类关键运动至少保留 1 组（跑步/骑行）。  
- 每次上游字段变更时补充对应 fixture 与回归用例。  
- fixture 文件命名需体现场景，如：  
  - `activity_running_tolerance_in.json`（容差内）  
  - `activity_running_tolerance_out.json`（容差外）  
  - `state_crash_recovery_before.json` / `state_crash_recovery_after.json`

## 6. 覆盖率与质量门禁建议
- 核心模块（fingerprinter/diff/store/retry/task_claim）行覆盖率 >= 85%。
- `sync_once` 集成路径覆盖所有阶段（拉取/计算/上传/重试/提交）。
- PR 门禁：
  1. 单测全绿
  2. 集成测试关键场景全绿
  3. 并发压力测试（轻量）通过

## 7. 建议测试命令（示例）
- 单元测试：`python -m unittest discover -s tests/unit -p 'test_*.py'`
- 集成测试：`python -m unittest discover -s tests/integration -p 'test_*.py'`
- 全量：`python -m unittest discover -s tests -p 'test_*.py'`

## 8. 分阶段落地
- T1：先落 `fingerprinter/diff/store` 单测。
- T2：补 `retry/task_claim` 并发单测。
- T3：补 `sync_once` 集成测试（mock connector）。
- T4：补崩溃恢复与并发压力测试，接入 CI。

## 9. 需求-测试覆盖矩阵（FR Traceability）

| 需求ID | 需求摘要 | 对应测试类型 | 对应用例/检查点 |
|---|---|---|---|
| FR-1 | 账号连接与凭证失效报错 | 集成 | I-E2E-01（正常连接）、新增 I-CONN-01（凭证失效报错路径） |
| FR-2 | 增量拉取 + 首次全量 + 双侧并行拉取 | 集成 | I-E2E-01、I-E2E-02；新增 I-FETCH-01（全量初始化）、I-FETCH-02（增量游标） |
| FR-3 | 标准化与指纹生成 | 单元 | U-FP-01/U-FP-02/U-FP-03；新增 U-NM-01（normalizer 字段映射） |
| FR-4 | 差集计算 | 单元 | U-DF-01/U-DF-02 |
| FR-5 | 双向上传并写状态 | 集成 | I-E2E-01、I-E2E-03（并发上传）、I-E2E-05（崩溃后状态一致） |
| FR-6 | 失败队列与指数退避重试 | 单元+集成 | U-RT-01/U-RT-02 + I-E2E-04 |
| FR-7 | 幂等控制 | 单元+集成 | U-CL-01/U-CL-02 + I-E2E-02 |
| FR-8 | 日志与审计 | 集成 | 新增 I-OBS-01（轮次指标日志）、I-OBS-02（活动级审计记录） |
| FR-9 | 并发执行控制与并发配置 | 单元+集成 | U-FS-02 + I-E2E-03 + 并发压力测试（轻量） |

结论：需求列表中的 FR-1 ~ FR-9 都有对应测试覆盖。当前文档中已定义的主用例可直接覆盖大部分需求，矩阵中补充的 `I-CONN-* / I-FETCH-* / U-NM-* / I-OBS-*` 可作为下一步新增用例编号，确保评审时可逐条对齐需求。
