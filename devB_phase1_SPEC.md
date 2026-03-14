# DevB Phase1 需求文档（AI Agent 执行版）

## 1. 任务目标
在 Phase1 中完成 DevB 职责：对齐调用链重试与异常语义，使 UserState 并发冲突在业务链路中可重试、可观测、可验收。

## 2. 输入基线（必须先读）
1. `docs/service-transformation/phase0.5/phase0.5-core.md`
2. `docs/service-transformation/三人协作执行方案.md`
3. `docs/service-transformation/devs.md`
4. `docs/service-transformation/log-field-dictionary-phase0.5.md`

## 3. 强制流程约束（必须执行）
1. 在开始任何代码修改前，先输出：
   - 你对本任务的理解
   - 你的改动计划（文件清单 + 预期改动点 + 测试计划）
   并等待我确认；未确认不得修改代码。
2. 本次改动完成后，必须更新项目根目录 `version.md`：
   - 若 `version.md` 不存在则新建。
   - 记录本次变更内容、影响范围、测试结果、日期。

## 4. DevB 责任边界

### 4.1 In Scope（必须做）
1. 为 wrapper + 关键写状态工具统一接入并发重试策略。
2. 统一并发异常语义与错误码透传，禁止静默吞错。
3. 输出可回放日志（`trace_id/user_id/goal_id/error_code`）。
4. 形成 Gate1 可评审材料（合同条款 + 日志样例 + 初版测试清单）。

### 4.2 Out of Scope（不做）
1. 不实现底层锁/CAS 机制（DevA 主责）。
2. 不负责 Gate2 最终签字（DevC 主责，DevB 配合）。
3. 不做 DB/Redis/MQ 等架构升级。

## 5. 并发合同口径（冻结）
1. 最大重试：3 次。
2. 退避：`20ms -> 50ms -> 100ms`，抖动 ±20%。
3. 重试耗尽：返回明确冲突错误，不得伪装成功。
4. 异常类型：`StateConflictError`、`StateLockTimeoutError`、`StateValidationError`、`StatePersistenceError`。
5. `session_id` 更新不得导致 `tags/milestones/quoted` 回退。

## 6. 代码改动任务（执行清单）

### 6.1 公共重试能力
1. 在 `backend/tools/__init__.py` 或新增公共模块（建议 `backend/tools/retry.py`）实现统一重试执行器/装饰器。
2. 只捕获并发相关异常；参数错误和业务错误不进入重试。

### 6.2 wrapper 链路
1. 修改 `backend/agent/wrapper.py`：
   - SDK 结束后 state 写入链路接入统一重试。
   - 区分可重试冲突异常和不可重试异常。
   - 失败时保留原始 `error_code` 和关键上下文字段。

### 6.3 工具链路（7 个）
为以下文件中“写 state”的关键路径接入统一重试：
1. `backend/tools/tags.py`
2. `backend/tools/order_link.py`
3. `backend/tools/sop_state.py`
4. `backend/tools/send_message.py`
5. `backend/tools/transfer_human.py`
6. `backend/tools/scheduler.py`
7. `backend/tools/youzan.py`

## 7. 日志要求
1. 每次重试记录 warning：`attempt/max_attempts/backoff_ms/error_code`。
2. 重试成功记录 info：至少包含 `attempt` 与 `user_id`。
3. 重试耗尽记录 error：至少包含 `error_code` 与请求上下文。
4. 日志字段契约遵循 `log-field-dictionary-phase0.5.md`。

## 8. 协作接口（与 DevA / DevC）
1. DevA 先给出并发异常抛出边界与类名。
2. DevB 只消费 DevA 的异常，不自定义冲突判定。
3. DevC 基于相同异常语义做回归断言，DevB提供失败日志样例。

## 9. 验收标准

### 9.1 Gate1（Phase1 Day1）
1. 并发合同条款已对齐且无歧义。
2. 重试/异常语义已有统一入口。
3. 可演示至少 1 条“冲突 -> 重试 -> 成功”样例链路。

### 9.2 Gate2（Phase1 收尾，DevB 配合）
1. `backend/tests/integration/test_concurrent_state.py` 连跑 3 轮通过。
2. 同用户并发写入 0 丢失（以测试报告为准）。
3. 失败链路可通过日志完整回放。

## 10. 执行后必须提交的产物
1. 代码改动（重试能力 + wrapper/工具接入 + 异常映射）。
2. 测试结果（命令 + 摘要）。
3. 关键日志样例（成功重试、重试耗尽各至少 1 条）。
4. `version.md` 更新记录。
5. 一段简短中文提交说明（1-3 行）。

## 11. 建议测试命令
1. `python -m pytest backend/tests/integration/test_concurrent_state.py -q`
2. `python -m pytest backend/tests/integration/test_error_handling.py -q`
3. `python -m pytest backend/tests/test_contextvars_isolation.py -q`

## 12. 执行 checklist
- [ ] 已在改代码前提交“理解 + 计划”并获得确认
- [ ] 已完成统一重试入口与 wrapper 接入
- [ ] 已完成 7 个关键工具接入
- [ ] 已对齐并发异常语义并保留原始 error_code
- [ ] 已产出 Gate1 评审材料与日志样例
- [ ] 已完成测试并记录结果
- [ ] 已更新 `version.md`


-----------------------------------
以下是agent给我的回复（我先让它检查了phase0.5是否完美实现了）。它的意思是不是phase0.5部分还有漏洞需要完善？它对phase1的任务理解又是怎么样的呢？有没有问题？如果没有，我是先让它完美补完phase0.5还是直接做phase1？
-------------------------------------
Phase0.5核查结论
  我先做了文档对照和代码/测试核查，当前结论是：功能基线基本达标，但还不能算“完美实现”。                                                                   
                                                                                                                                                          
  1. 已达标项                                                                                                                                             
  2. Gate0相关自动化在当前仓库可通过：                                                                                                                    
      - test_sop_phase0_idempotency.py：19 passed                                                                                                         
      - test_sop_pipeline.py：48 passed                                                                                                                   
      - test_contextvars_isolation.py：9 passed                                                                                                           
      - test_chat_api.py：11 passed                                                                                                                       
      - test_sop_basic.py：29 passed                                                                                                                      
  3. trace_id / user_id / goal_id链路已接通：                                                                                                             
      - 日志过滤器注入字段在 main.py:36、main.py:55                                                                                                       
      - Trace中间件透传/生成 X-Trace-ID 在 trace.py:36                                                                                                    
      - Chat入口设置 user_scope/goal_scope 在 chat.py:200                                                                                                 
      - Wrapper统一设置 goal_id 在 wrapper.py:146                                                                                                         
  4. 仍有缺口（所以不是“完美”）                                                                                                                           
  5. request_id未进入主业务日志上下文（仅在审计日志里有）：                                                                                               
      - Phase0.5要求见 phase0.5-core.md:45                                                                                                                
      - 审计里有 request_id 在 audit.py:33                                                                                                                
  6. error_code透传/日志不统一，且多处状态写失败被吞掉：                                                                                                  
      - 例如 order_link.py:95、send_message.py:397、transfer_human.py:198、youzan.py:281
  7. Phase0.5文档内部状态有矛盾：                                                                                                                         
      - 同一文档既写“待DevB”又写“日志字段完整” sop_scene_matrix_phase0.md:139、sop_scene_matrix_phase0.md:160                                             
                                                                                                                                                          
  ———                                                                                                                                                     
                                                                                                                                                          
  我对Phase1 DevB任务的理解                                                                                                                               
                                                                                                                                                          
  1. 只做 DevB 范围：统一重试入口、统一并发异常语义、日志可回放、Gate1材料。                                                                              
  2. 不做 DevA 的锁/CAS冲突判定实现，不做 DevC 验收主责，不做 BM25(Phase1.5)。                                                                            
  3. 必须满足你给的流程：先给计划，等你批准后再改代码。                                                                                                   
  4. 完成后要更新根目录 version.md（当前仓库只有 versions.md，没有 version.md）。                                                                         
                                                                                                                                                          
  ———                                                                                                                                                     
                                                                                                                                                          
  拟修改计划（等你批准后执行）                                                                                                                            
                                                                                                                                                          
  1. 新增统一重试模块（建议 backend/tools/retry.py）                                                                                                      
  2. 固化策略：3次、20/50/100ms、±20%抖动。                                                                                                               
  3. 只重试并发相关异常（按约定类名）。                                                                                                                   
  4. 输出标准日志：attempt/max_attempts/backoff_ms/error_code。                                                                                           
  5. 接入 wrapper 状态写入链路 wrapper.py                                                                                                                 
  6. 将关键 state_manager.save(...) 全部走统一重试（初始化写、milestone写、session_id写）。                                                               
  7. 区分可重试并发异常与不可重试异常。                                                                                                                   
  8. 失败时返回并保留原始 error_code 与上下文。                                                                                                           
  9. 接入 7 个工具文件（仅“写 state”路径）                                                                                                                
  10. tags.py                                                                                                                                             
  11. order_link.py                                                                                                                                       
  12. sop_state.py                                                                                                                                        
  13. send_message.py                                                                                                                                     
  14. transfer_human.py                                                                                                                                   
  15. scheduler.py                                                                                                                                        
  16. youzan.py                                                                                                                                           
  17. 去掉“状态写失败但工具仍成功返回”的静默吞错行为，统一错误码透传。                                                                                    
  18. Gate1材料与版本记录                                                                                                                                 
  19. 新增 docs/service-transformation/phase1/devB-gate1-materials.md（合同条款、日志样例、初版测试清单）。                                               
  20. 新建并更新根目录 version.md 记录本次变更、影响范围、测试结果、日期。                                                                                
  21. 测试计划（DevB范围）                                                                                                                                
  22. 先跑你要求的三条：                                                                                                                                  
      - python -m pytest backend/tests/integration/test_concurrent_state.py -q（若文件仍不存在，将明确记录为DevC待补）                                    
      - python -m pytest backend/tests/integration/test_error_handling.py -q                                                                              
      - python -m pytest backend/tests/test_contextvars_isolation.py -q                                                                                   
  23. 再跑受影响工具测试与 chat API 回归，确保无回归。                                                                                                    
                                                                                                                                                          
  如果你批准，我就按这个计划开始改代码，并严格只动 DevB 范围。                                                                                            
 --
