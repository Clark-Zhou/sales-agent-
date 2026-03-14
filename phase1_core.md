# Phase1 核心收口文档（兼 Phase1.5 基线）

## 1. 目标与定位
- 本文件对应 [三人协作执行方案](../三人协作执行方案.md) 中的完整 `Phase1`，用于确认 `UserState 并发写丢失` 这一问题域已经满足收尾条件。
- 本文件同时作为 `Phase1.5` 的唯一基线/交接文档，后续 `BM25` 冷启动竞态修复以本文和 Phase0.5 材料为输入继续推进。
- 当前结论口径为：`Phase1` 从工程实现、测试和文档证据上已满足收尾条件，且已合入本地 `main`；正式关闭仍以后续远端提交和签字为准。
- 本文聚焦阶段收口与交接，不替代后续 PR、merge 记录或正式签字页。

## 2. Phase0.5 输入与 Phase1 问题定义
- `Phase1` 的直接输入来自：
  - [三人协作执行方案](../三人协作执行方案.md)
  - [Phase0.5 核心收口文档](../phase0.5/phase0.5-core.md)
  - [SOP 场景矩阵](../phase0.5/sop_scene_matrix_phase0.md)
  - [Phase0.5 日志字段字典](../log-field-dictionary-phase0.5.md)
  - [Phase1 OpenSpec 提案](../../../openspec/changes/fix-userstate-concurrency-phase1/proposal.md)
  - [Phase1 OpenSpec 任务清单](../../../openspec/changes/fix-userstate-concurrency-phase1/tasks.md)
- Phase0.5 已明确 Phase1 的核心问题：
  1. 主业务链路大量存在 `get_or_create() -> 改字段 -> save()` 直写模式，同一用户并发请求时容易出现“后写覆盖前写”。
  2. 重点风险字段是 `tags / milestones / quoted / session_id`，其中 `quoted` 和关键 `milestones` 一旦回退，会直接影响 SOP 触发和后续决策。
  3. `wrapper` 在 SDK 返回后补写 `session_id`，如果仍走整对象覆盖写，会把工具刚更新的状态抹掉。
  4. Phase0.5 还登记了观测缺口和异常语义不统一问题，要求 Phase1 完成后至少具备 `trace_id/request_id/user_id/goal_id/error_code` 的一致日志口径。
- Phase0.5 交接给 Phase1 的主验收口径是 `S07-S10`：
  - 同用户并发写 `tags` 与 `quoted` 不丢失。
  - 同用户并发写多个 `milestones` 不回退。
  - `session_id` 更新不得覆盖 `tags/milestones/quoted`。
  - 10 worker 并发混合写后关键字段全部保留。

## 3. Phase1 本轮落地内容

### 3.1 Dev A：状态事务层与主链路写入口收口
- 在 `backend/agent/state.py` 新增 `StateManager.mutate(...)`，作为同用户状态写入的唯一事务入口。
- 写入路径固定为：用户级 `.json.lock` 文件锁、锁内重读最新 state、执行 mutator、校验并发合同、原子写盘、释放锁。
- 并发合同已编码化：
  - `tags.needs/barriers/behaviors` 只允许并集去重，不允许把已有值写丢。
  - `tags.stage` 允许覆盖。
  - `milestones` 做键级 merge，布尔值默认不允许 `True -> False`。
  - `quoted` 为 sticky true，不允许并发写回 `False`。
  - `session_id` 可更新，但不得回退 `tags/milestones/quoted`。
- 新增并落地 4 类状态异常：
  - `StateConflictError`
  - `StateLockTimeoutError`
  - `StateValidationError`
  - `StatePersistenceError`
- 主业务写入口已统一收口到事务层：
  - `wrapper` 的 goal 初始化、trigger flag 写入、SDK 返回后的 `session_id` 保存
  - `scheduler` 的状态写回与 pending followup 领取
  - 写 state 的 tools 和 `api/users`

### 3.2 Dev B：观测字段统一与异常语义收口
- `/api/*` 请求已统一透传或生成 `X-Request-ID`，并回写到响应头。
- 全局日志字段已统一为：
  - `trace_id`
  - `request_id`
  - `user_id`
  - `goal_id`
  - `error_code`
- `wrapper` 的 SSE `error` 事件已补充 `error_code`，成功事件结构保持不变。
- Dev B 在三人方案里负责的“调用链重试与异常语义对齐”，在当前主线方案中的最终落地方式是：
  - 保留状态层内部重试与锁退避，不额外引入业务侧 `save()+retry`
  - 保留 `error_code` 观测和对上层可见的错误语义
  - 不吸收 `backend/tools/retry.py` 这一套旧路径实现
- 混合失败语义已定稿：
  - `generate_order_link`：状态写失败时显式报错，避免“已生成链接但系统未记住”
  - `send_message`：保持业务成功优先，避免重复发送
  - `transfer_to_human`：保持业务成功优先，避免重复转接
  - `query_youzan`：保持查询成功优先，状态写失败只做降级日志

### 3.3 Dev C：并发回归与 Gate2 验收
- `Dev C` 已补齐 `S07-S10` 到 `backend/tests/integration/test_concurrent_state.py`，覆盖并发写合同的 4 个核心场景。
- 状态层单测已补齐，覆盖锁超时、非法回退、持久化失败和日志字段。
- 旧的并发边界测试口径已更新，不再接受 `Last-Write-Wins` 作为正确结果。
- 受影响的工具、调度器和 `wrapper` 测试已同步对齐到 `mutate()` 调用方式和当前错误语义。

## 4. 测试与 Gate2 验收结果

### 4.1 Gate2 主验收场景
`S07-S10` 已自动化并通过：

| 场景 | 验收点 | 结果 |
|---|---|---|
| S07 | 同用户并发写 `tags` 与 `quoted` 不丢失 | 已通过 |
| S08 | 同用户并发写多个 `milestones` 不回退 | 已通过 |
| S09 | `session_id` 更新不覆盖 `tags/milestones/quoted` | 已通过 |
| S10 | 10 worker 混合并发写关键字段全部保留 | 已通过 |

### 4.2 已记录的回归证据
- Gate2 组合回归：

```bash
backend/.venv/bin/pytest -q -o addopts='' \
  backend/tests/integration/test_concurrent_state.py \
  backend/tests/integration/test_sop_phase0_idempotency.py \
  backend/tests/integration/test_sop_pipeline.py
```

- 实际结果：连续运行 3 轮，均为 `71 passed`。
- 受影响面定向回归结果：`82 passed`。
- 观测字段与剩余合并后的定向回归结果：`131 passed`。
- 更宽一轮的合并后回归结果：`142 passed`。
- 已覆盖的证据组包括：
  - `test_concurrent_state.py + test_sop_phase0_idempotency.py + test_sop_pipeline.py`
  - `test_contextvars_isolation.py + test_chat_api.py`
  - `test_order_link_tool.py + test_send_message.py + test_transfer_human.py + test_youzan.py`
- OpenSpec 状态：
  - [tasks.md](../../../openspec/changes/fix-userstate-concurrency-phase1/tasks.md) 已全部打勾
  - `openspec validate fix-userstate-concurrency-phase1 --strict` 已通过

### 4.3 手工验证与 API 验证补充
- 手工流式验证已完成：
  - 使用 `curl` 调用 `/api/chat/stream`，响应为 `200 OK`
  - 响应头正确返回 `X-Trace-ID` 与 `X-Request-ID`
  - `SSE` 事件链完整，包含 `mode / step / tool_call / state_update / done`
  - 验证过程中未出现 `error` 事件
- 同一用户双并发流式验证已完成：
  - 两个并发请求均正常结束
  - 未出现 `500`、超时、卡死或明显死锁
  - 该手测主要用于确认主链路稳定性，不替代 `S07-S10` 的自动化并发合同验证
- 最新一轮本地验证结果：
  - `backend/tests/api/test_chat_api.py`：`12 passed`
  - `test_contextvars_isolation.py + test_concurrent_state.py + test_sop_phase0_idempotency.py + test_sop_pipeline.py`：`80 passed`
  - `test_tags.py + test_order_link_tool.py + test_sop_state_tool.py + test_send_message.py + test_transfer_human.py + test_youzan.py`：`50 passed`

## 5. Phase1 收尾判断
- 按 [三人协作执行方案](../三人协作执行方案.md) 的 Gate 定义，`Phase1` 需要满足两类条件：
  1. Gate1：并发合同冻结，三方对异常名、冲突语义、重试口径无歧义。
  2. Gate2：同用户并发写 0 丢失，关键回归全绿，并有验收材料。
- 当前判断：
  - Gate1 已具备证据：见 [phase0.5-core.md](../phase0.5/phase0.5-core.md) 第 7 节并发合同、[proposal.md](../../../openspec/changes/fix-userstate-concurrency-phase1/proposal.md) 和当前实现。
  - Gate2 已具备证据：`S07-S10` 自动化、Gate2 组合回归 3 轮、工具与观测链路回归均已通过。
- 当前工程状态补充如下：
  - `Phase1` 相关提交已合入本地 `main`
  - OpenSpec 变更目录仍保留在 `openspec/changes/` 下，作为当前阶段证据的一部分
- 因此，当前未完成的只剩流程动作，而不是工程缺口：
  - 远端提交
  - 签字
  - 后续按节奏进入 `Phase1.5`
- 结论：`Phase1` 已满足收尾条件，并已完成本地收尾。

## 6. Phase1.5 基线与交接输入
- `Phase1.5` 的输入来源固定为：
  - [三人协作执行方案](../三人协作执行方案.md)
  - [phase0.5-core.md](../phase0.5/phase0.5-core.md)
  - 本文当前版本
- `Phase1.5` 的问题域固定为：`BM25 冷启动竞态`。
- `Phase1.5` 的主验收目标固定为 `S11-S13` 与 Gate3：
  - S11：冷启动并发查询每 worker 加载受控
  - S12：冷启动加载失败走降级且可观测
  - S13：知识库 reload 与查询并发不崩溃
- 当前已经具备的前提：
  - `request_id/error_code` 观测字段已统一，可用于 Gate3 排障
  - `Phase1` 的并发主风险已收口，不再与 BM25 冷启动问题混叠
  - 主链路错误语义已定稿，后续 Phase1.5 只需在 BM25 范围内补齐
- 当前尚未完成、应由 `Phase1.5` 承接的事项：
  - `backend/tests/integration/test_bm25_cold_start.py`
  - 冷启动压测报告
  - 多 worker 首波加载次数验证
  - reload/query 并发安全验证
  - 失败降级路径与日志摘要

## 7. 当前剩余边界
- `BM25` 冷启动修复本身尚未开始，属于 `Phase1.5`，不作为本轮 Phase1 收尾阻塞项。
- `Phase3` 灰度、告警与复盘闭环尚未开始，不在本文完成范围内。
- `backend/api/debug.py` 与 `backend/benchmark/runner.py` 仍保留非主链路 `save()` 使用，但不属于本轮主业务入口，也不作为本轮收尾阻塞项。
- `.env` 中 `SESSION_STORAGE_PATH` 的路径风险本轮仅登记，不写成本轮已修复项。
- 本轮未引入 Redis/DB，也未改变外部 API 结构；实现范围仍限定在内部一致性、测试验收与可观测性收口。

## 8. 结论
- `Dev A` 的状态事务层与并发合同已落地，主业务写路径已统一收口到 `mutate()`。
- `Dev C` 的并发回归与 Gate2 验收已落地，`S07-S10` 已补齐并通过验证。
- `Dev B` 在 Phase1 范围内需要落到主线的观测字段统一与异常语义收口也已完成，并已按当前主线架构吸收。
- 因此，按三人协作方案的完整 `Phase1` 口径，当前状态应表述为：`已满足收尾条件，已合入本地 main，待远端提交和签字即可正式关闭`。
- 后续若继续推进，`Phase1.5` 应直接以本文为基线，进入 `BM25` 冷启动竞态修复与 Gate3 验收。
