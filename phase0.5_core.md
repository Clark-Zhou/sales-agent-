# Phase0.5 核心收口文档（Gate0 基线）

## 1. 目标与边界
- 目标：冻结 SOP 基线，明确“触发-执行-落盘-观测”链路，作为 Phase1 并发改造唯一输入。
- 边界：Phase0.5 仅提交文档与验收材料，不修改业务逻辑，不改外部 API。
- 适用范围：`guanzhan` 品牌默认业务线。

## 2. 基线入口与链路
- 触发规则入口：`backend/scheduler/trigger.py`（`get_trigger_goal/get_trigger_result`）。
- 对话执行入口：`backend/agent/wrapper.py`（`process_message()`）。
- 流程定义：`backend/config/brands/guanzhan/lines/default/sop/stage_flows.yaml`。
- 规则定义：`backend/config/brands/guanzhan/rules.yaml`。

端到端冻结链路：
1. `scheduler` 按 `rules.yaml` 判定 `goal`（Demo 模式可跳过时间条件）。
2. `wrapper.process_message(goal=...)` 加载用户状态。
3. 若 `goal` 属于核心 goal（`welcome/taste_feedback/coupon_20/coupon_50_final`）：
   - 已有 `*_triggered=true`：跳过重复初始化。
   - 否则解析 `goal -> flow` 并初始化 `flow/stage`。
4. `scheduler_goal_mapping` 写入状态字段（核心 goal 仅在初始化路径成功后写 `*_triggered`）。
5. SDK 会话完成后，`wrapper` 二次加载 state 并写入 `session_id`。

## 3. 核心 Goal 四元组（进入条件 + 初始化动作 + 状态写入字段 + 重复触发行为）

| Goal | 进入条件（规则侧） | 初始化动作（wrapper） | 状态写入字段 | 重复触发行为 |
|---|---|---|---|---|
| `welcome` | `stage=new` 且 `welcome_done=false` | 解析 `guanzhan_day1_onboarding`，`enter_flow` + `enter_stage(首阶段)` | `welcome_triggered=true`（仅初始化路径成功后） | 若已 `welcome_triggered=true`，跳过 flow/stage 重复初始化 |
| `taste_feedback` | `welcome_done=true`、`order_collected=false`、`taste_asked=false`、`days_since_created>=2` | 解析 `guanzhan_day3_taste_upgrade`，初始化 flow/stage | `taste_feedback_triggered=true` | 若已 `taste_feedback_triggered=true`，跳过重复初始化 |
| `coupon_20` | `quoted=true`、`order_collected=false`、`coupon_20_sent=false`、`days_since_created>=4` | 解析 `guanzhan_day5_coupon_20`，初始化 flow/stage | `coupon_20_triggered=true` | 若已 `coupon_20_triggered=true`，跳过重复初始化 |
| `coupon_50_final` | `coupon_20_sent=true`、`order_collected=false`、`coupon_50_sent=false`、`days_since_created>=6` | 解析 `guanzhan_day7_coupon_50`，初始化 flow/stage | `coupon_50_final_triggered=true` | 若已 `coupon_50_final_triggered=true`，跳过重复初始化 |

非核心 goal（保持现状，不在 0.5 改动）：
- `long_term_followup_1 -> followup_14_done`
- `long_term_followup_2 -> followup_21_done`
- `long_term_followup_final -> followup_30_done`

## 4. 幂等点与失败回退点
- 幂等点 A：核心 goal 的 `*_triggered` 检查（避免重复初始化同一 flow）。
- 幂等点 B：若当前已在目标 `flow`，视为初始化路径可达，不重复 `enter_flow`。
- 回退点 A：`goal -> flow` 解析失败时，不写 `*_triggered`。
- 回退点 B：品牌上下文缺失时直接报错退出，不进入执行链。

## 5. 日志字段契约（含字段字典）

### 5.1 Gate0 必备字段（用于场景回放与故障定位）
- `trace_id`
- `request_id`
- `user_id`
- `brand_id`
- `goal_id`
- `rule_name`
- `flow_id`
- `stage_id`
- `session_id`
- `error_code`

### 5.2 扩展字段（建议）
- `line_id`
- `event`
- `status`

### 5.3 字段规则（冻结）
| 字段 | 约束 |
|---|---|
| `goal_id` | 仅记录真实业务目标；无目标统一写 `-`；不得写 `passive_response` |
| `trace_id` | 来自 `X-Trace-ID` 或中间件生成，缺失时回退 `-` |
| `session_id` | 可更新，但不得导致 `tags/milestones/quoted` 回退 |
| `error_code` | 保留原始错误类型，禁止统一吞并成泛化错误 |

## 6. 失败路径清单（Pre / In / Post）

### 6.1 触发前（Pre-Trigger）
| ID | 症状 | 触发条件 | 定位日志字段 | 责任人 | 临时止损 |
|---|---|---|---|---|---|
| PRE-01 | 请求直接报错，SOP 不进入 | `brand_id` 缺失或上下文未设置 | `request_id,user_id,brand_id,error_code` | Dev B | API 层强制 `brand_id` 入参校验 |
| PRE-02 | 规则不命中，用户应触发但未触发 | `rules.yaml` 条件字段与真实 state 字段不一致 | `rule_name,user_id,brand_id,error_code` | Dev A | 回滚到上个已验证规则版本 |
| PRE-03 | 命中规则但无法进入流程 | `goal` 在规则存在但 `stage_flows` 无对应 flow | `goal_id,flow_id,error_code,brand_id` | Dev A | 暂停该 goal 自动触发，转人工 |
| PRE-04 | 时间条件判断异常 | 时区/时间基准偏差导致 `days_since_created` 误判 | `rule_name,user_id,brand_id,trace_id` | Dev B | 以 Demo 模式复核并冻结变更 |

### 6.2 触发中（In-Trigger）
| ID | 症状 | 触发条件 | 定位日志字段 | 责任人 | 临时止损 |
|---|---|---|---|---|---|
| IN-01 | 同一核心 goal 重复初始化 flow | `*_triggered` 写入或读取异常 | `goal_id,flow_id,stage_id,user_id,trace_id` | Dev A | 单用户补写 `*_triggered=true` |
| IN-02 | 流程卡在 stage | 未调用 `update_sop_state` 或参数无效 | `goal_id,stage_id,error_code,user_id` | Dev A + Dev C | 人工补写 stage/step |
| IN-03 | `tags` 更新后丢失 | 并发完整对象覆盖写 | `user_id,request_id,trace_id,error_code` | Dev A | 同用户入口限流 |
| IN-04 | `quoted=True` 回退为 False | 并发写冲突，后写覆盖前写 | `user_id,goal_id,session_id,error_code` | Dev A | 成交字段启用人工兜底 |
| IN-05 | `session_id` 写入后覆盖其他字段 | SDK 二次保存与工具保存竞争 | `session_id,user_id,trace_id,error_code` | Dev A + Dev B | 异常时停用自动恢复会话 |

### 6.3 触发后（Post-Trigger）
| ID | 症状 | 触发条件 | 定位日志字段 | 责任人 | 临时止损 |
|---|---|---|---|---|---|
| POST-01 | 对账失败，无法复盘触发链路 | 缺失关键日志字段 | `trace_id,request_id,goal_id,rule_name` | Dev B | 强制结构化日志模板 |
| POST-02 | 用例通过但线上仍复现 | 场景矩阵未覆盖并发/竞态窗口 | `trace_id,user_id,goal_id` | Dev C | 增补回放脚本并重跑演练 |
| POST-03 | 冷启动首波延迟抖动 | BM25 首次加载并发重复 load | `trace_id,request_id,error_code` | Dev B | 低峰预热与降级观测 |
| POST-04 | 材料齐但不可执行 | 文档与代码版本偏离 | `request_id,trace_id` | Dev A | 锁定基线 commit 重新核对 |

统一升级规则：
1. 同一问题 24 小时内复现 >= 2 次，升级为 P0 候选。
2. 涉及 `tags/milestones/quoted/session_id` 任一字段回退，进入并发问题池。
3. 无法定位到 `goal_id/rule_name/flow_id` 的故障按“观测缺陷”处理并阻断放量。

## 7. 并发合同 v1（Phase1 输入）

### 7.1 覆盖字段
- `tags`
- `milestones`
- `quoted`
- `session_id`

### 7.2 冲突语义（冻结）
- `tags.needs/barriers/behaviors`：集合并集（去重，不丢历史值）。
- `tags.stage`：允许覆盖，最后成功提交值（LWW）。
- `milestones`：键级 merge；布尔字段默认只允许 `False -> True`；`True -> False` 仅显式 reset 工具可做。
- `quoted`：粘滞位，一旦 `True` 不可被并发写回 `False`。
- `session_id`：允许覆盖，但更新不得回退 `tags/milestones/quoted`。

### 7.3 重试语义（冻结）
- 最大重试：3 次。
- 退避：`20ms -> 50ms -> 100ms`，抖动 ±20%。
- 重试耗尽返回冲突错误，不做静默吞错。

### 7.4 错误语义（冻结）
- `StateConflictError`
- `StateLockTimeoutError`
- `StateValidationError`
- `StatePersistenceError`

要求：保留原始 `error_code`，且记录失败前关键字段 diff。

## 8. 场景矩阵（Gate0/Gate2/Gate3）

| ID | 场景描述 | 问题域 | 优先级 | 状态 | 对应用例文件 | 责任人 |
|---|---|---|---|---|---|---|
| S01 | 核心 goal 首次触发写入 `*_triggered=true` | SOP 幂等 | P0 | 已有自动化 | `backend/tests/integration/test_sop_phase0_idempotency.py` | Dev C |
| S02 | 核心 goal 二次触发不重复初始化 flow/stage | SOP 幂等 | P0 | 已有自动化 | `backend/tests/integration/test_sop_phase0_idempotency.py` | Dev C |
| S03 | flow 解析失败时不写 `*_triggered` | SOP 幂等 | P0 | 已有自动化 | `backend/tests/integration/test_sop_phase0_idempotency.py` | Dev C |
| S04 | 非核心 goal 行为保持不变 | SOP 回归 | P0 | 已有自动化 | `backend/tests/integration/test_sop_phase0_idempotency.py` | Dev C |
| S05 | 规则链路 `state -> rule -> goal` 命中正确 | SOP 规则 | P0 | 已有自动化 | `backend/tests/integration/test_sop_pipeline.py` | Dev C |
| S06 | `goal -> flow` 映射一致（Day1/3/5/7） | SOP 流程 | P0 | 已有自动化 | `backend/tests/integration/test_sop_pipeline.py` | Dev C |
| S07 | 同用户并发写 `tags` 与 `quoted` 不丢失 | State 并发 | P0 | 待新增 | `backend/tests/integration/test_concurrent_state.py` | Dev A + Dev C |
| S08 | 同用户并发写多个 `milestones` 不回退 | State 并发 | P0 | 待新增 | `backend/tests/integration/test_concurrent_state.py` | Dev A + Dev C |
| S09 | SDK 写 `session_id` 不覆盖 `tags/milestones/quoted` | State 并发 | P0 | 待新增 | `backend/tests/integration/test_concurrent_state.py` | Dev A + Dev C |
| S10 | 10 worker 并发混合写（关键字段核对） | State 并发 | P0 | 待新增 | `backend/tests/integration/test_concurrent_state.py` | Dev A + Dev C |
| S11 | 冷启动并发查询每 worker 加载受控 | BM25 冷启动 | P0 | 待新增 | `backend/tests/integration/test_bm25_cold_start.py` | Dev B + Dev C |
| S12 | 冷启动加载失败走降级且可观测 | BM25 冷启动 | P0 | 待新增 | `backend/tests/integration/test_bm25_cold_start.py` | Dev B + Dev C |
| S13 | 知识库 reload 与查询并发不崩溃 | BM25 竞态 | P1 | 待新增 | `backend/tests/integration/test_bm25_cold_start.py` | Dev B + Dev C |
| S14 | 被动模式下不触发主动规则 | 调度边界 | P1 | 已有自动化 | `backend/tests/scheduler/test_trigger.py` | Dev C |

## 9. Gate0 验证流程与缺口

### 9.1 Gate0 验证流程
1. 按 S01-S06 回放基线场景。
2. 对照日志字段，确保可映射 `goal/rule/flow/stage/user`。
3. 在失败路径中随机抽 3 条，验证可定位性。
4. 形成“通过/不通过 + 整改项 + 责任人 + 截止日期”。

### 9.2 Gate2/Gate3 缺口（本阶段仅登记）
| 缺口 | 目标文件 | 负责人 | 计划阶段 | 备注 |
|---|---|---|---|---|
| 同用户并发写 0 丢失验收缺失 | `backend/tests/integration/test_concurrent_state.py` | Dev A + Dev C | Phase1 | Gate2 主验收 |
| BM25 冷启动并发验收缺失 | `backend/tests/integration/test_bm25_cold_start.py` | Dev B + Dev C | Phase1.5 | Gate3 主验收 |

## 10. Gate0 签字与结论

### 10.1 验收证据（2026-03-13）
- 自动化回归（功能口径，关闭 coverage 门槛）：
  - `./backend/.venv/bin/pytest backend/tests/integration/test_sop_phase0_idempotency.py -q --no-cov` -> `19 passed`
  - `./backend/.venv/bin/pytest backend/tests/test_contextvars_isolation.py -q --no-cov` -> `9 passed`
  - `./backend/.venv/bin/pytest backend/tests/api/test_chat_api.py -q --no-cov` -> `11 passed`
- 手测场景（`/api/chat/stream`、`/api/chat/send`、trace 透传、history、reset）可复现且可定位。
- 风险说明：长消息手测过程中出现过第三方网关鉴权/余额错误（非本仓代码回归），按“环境依赖风险”登记，不阻塞 Phase0.5 文档收口。

### 10.2 签字栏
| 角色 | 姓名 | 结论（通过/不通过） | 日期 | 备注 |
|---|---|---|---|---|
| Dev A（状态/规则） | 待填写 | 通过 | 待填写 | 已完成文档收口与基线冻结 |
| Dev C（测试/验收） | 待填写 | 通过 | 待填写 | 自动化回归通过；环境风险已登记 |

### 10.3 最终结论
- Gate0 结果：`[x] 通过` / `[ ] 不通过`
- 进入条件：Phase1 并发开发可开始，验收以本文件与对应 `commit_sha` 为准。

## 11. 基线承诺
- Phase1 开发前，协作者统一以本文件 + 对应 `commit_sha` 为唯一基线。
- 本文档仅定义行为与验收口径，不替代后续 Phase1/1.5 的实现与测试文件。

## 12. Phase1 交接清单
- Dev A：按第 7 节并发合同实现 `tags/milestones/quoted/session_id` 的冲突与重试语义。
- Dev C：按第 8 节补齐 `S07-S10` 自动化（`test_concurrent_state.py`）。
- Dev B：对齐第 5 节日志字段契约，补齐链路字段一致性与错误码透出。
- Gate1 评审输入：并发合同条款 + 关键日志样例 + 初版并发测试清单。

## 13. 附录A：SOP Scene 矩阵补充（摘要）

说明：
- 本节为 Phase1 决策输入所需的摘要版。
- 原始全量明细保留在 `docs/service-transformation/phase0.5/sop_scene_matrix_phase0.md`，用于回溯与审计，不作为主入口。

### 13.1 场景规模总览（Dev C 基线）
| 类别 | 场景数 | 自动化覆盖 | 主要测试文件 |
|---|---|---|---|
| A. 正常路径 | 4 | 是 | `test_sop_phase0_idempotency.py` |
| B. 重复触发（串行幂等） | 1 | 是 | `test_sop_phase0_idempotency.py` |
| C. 并发触发 | 12 | 是 | `test_sop_phase0_idempotency.py` |
| D. 失败路径 | 2 | 是 | `test_sop_phase0_idempotency.py` |
| E. 触发规则链 | 16 | 是 | `test_sop_pipeline.py` |
| F. 规则优先级 | 4 | 是 | `test_sop_pipeline.py` |
| G. 流程映射 | 9 | 是 | `test_sop_pipeline.py` |
| H. 配置完整性 | 8 | 是 | `test_sop_pipeline.py` |
| 合计 | 56 | 是 | 以上两类为主 |

### 13.2 关键补充结论
- Phase0.5 的 SOP 基线已具备“首次触发、重复触发、并发触发、失败路径、规则链、流程映射、配置完整性”全链路可测材料。
- 现有自动化最小闭环可归约为第 10 节的 3 组回归（`19 + 9 + 11`），用于 Gate0 与日常回归。
- 交接到 Phase1 的高优先项保持不变：并发首次触发竞态、UserState 并发写覆盖风险。

### 13.3 异常路径补充（与第 6 节一致）
| 异常路径 | 当前处理 | 阶段归属 |
|---|---|---|
| flow 解析失败 | 不写 trigger flag，保持可观测 | Phase0.5（已冻结） |
| 并发重复触发（已首次） | 幂等跳过初始化 | Phase0.5（已冻结） |
| 并发首次触发（未触发） | 可能出现初始化竞争窗口 | Phase1（Dev A 主任务） |
| UserState 并发写覆盖 | 可能丢失 `tags/milestones` | Phase1（Gate2 主验收） |

### 13.4 运行命令（补充）
```bash
cd backend
python -m pytest tests/integration/test_sop_phase0_idempotency.py \
                 tests/integration/test_sop_pipeline.py \
                 -q --no-cov
```
