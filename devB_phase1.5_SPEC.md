# DevB Phase1.5 需求文档（AI Agent 执行版）

## 1. 任务目标
在 `Phase1.5` 完成 DevB 职责：修复 `BM25 冷启动竞态`，实现冷启动并发保护与失败降级，产出 Gate3 可验收证据。

## 2. 输入基线（必须先读）
1. `docs/service-transformation/phase1/phase1-core.md`
2. `docs/service-transformation/phase0.5/phase0.5-core.md`
3. `docs/service-transformation/三人协作执行方案.md`
4. `docs/service-transformation/log-field-dictionary-phase0.5.md`
5. `docs/service-transformation/devs.md`（若存在）
6. `docs/service-transformation/完整技术方案-材料B.md`（仅 BM25/冷启动相关章节）

## 3. 强制流程约束（必须执行）
1. 在开始任何代码修改前，先输出：
   - 你对本任务的理解
   - 你的改动计划（文件清单 + 预期改动点 + 测试计划）
   并等待我确认；未确认不得修改代码。
2. 仅允许改动 DevB 边界内文件；涉及 DevA/DevC 职责时只能记录协作需求，不得代做。
3. 本次改动完成后，必须更新项目根目录 `version.md`：
   - 若 `version.md` 不存在则新建。
   - 记录本次变更内容、影响范围、测试结果、日期。
4. 若发现阻塞问题不属于 DevB 范围（例如状态并发合同变更），必须先停下并汇报，不得自行扩 scope。

## 4. DevB 责任边界

### 4.1 In Scope（必须做）
1. 实现 BM25 冷启动并发保护（同一 worker 内首次并发查询不重复加载）。
2. 实现冷启动失败降级路径，并保证可观测（可回放日志 + 错误码）。
3. 实现 `reload/query` 并发安全，保证不崩溃、不出现中间态污染。
4. 补齐 Gate3 主验收用例与执行材料：
   - `backend/tests/integration/test_bm25_cold_start.py`
   - 冷启动压测与加载次数证据
   - 降级路径日志摘要

### 4.2 Out of Scope（不做）
1. 不实现 DevA 的一致性评审结论本身（仅接入其评审结论）。
2. 不承担 DevC 的验收签字主责（DevB 只提供实现与证据）。
3. 不做 SOP/UserState 新需求开发。
4. 不做 DB/Redis/MQ 架构升级（如确需引入，必须另开变更单）。

## 5. Phase1.5 冻结口径（必须遵守）
1. 问题域仅限：`BM25 冷启动竞态`。
2. 验收目标固定：`S11/S12/S13`。
3. Gate3 通过标准固定：
   - 每 worker 冷启动加载受控（一次级别）
   - 首波延迟稳定
   - 失败可降级且可观测
4. 外部 API 默认不变；允许新增测试和观测字段。

## 6. 代码改动任务（执行清单）

### 6.1 冷启动并发保护（核心）
1. 修改 `backend/knowledge/retriever.py`：
   - 在 `_ensure_loaded()` 引入同集合并发保护（单飞/互斥）。
   - 保证“检查已加载 + 加载”是原子流程，避免重复 load。
2. 必要时修改 `backend/knowledge/bm25_engine.py`：
   - 保持加载接口幂等语义，避免并发下重复构建覆盖。

### 6.2 reload/query 并发安全
1. 修改 `backend/knowledge/retriever.py` 的 `reload()` 与 `search()/search_sync()`：
   - 保证 reload 与查询并发时不崩溃。
   - 查询要么读旧可用集合，要么读新可用集合，不读半成品状态。

### 6.3 降级与异常语义
1. 修改 `backend/tools/knowledge.py`（必要时）：
   - BM25 加载失败时走降级返回，不得导致进程异常退出。
   - 保留原始 `error_code`，并输出标准上下文字段。
2. 修改 `backend/api/knowledge.py`（必要时）：
   - reload 失败语义统一，错误码可追踪。

### 6.4 Gate3 验收测试与材料
1. 新增 `backend/tests/integration/test_bm25_cold_start.py`，覆盖：
   - S11：冷启动并发查询每 worker 加载受控
   - S12：冷启动加载失败走降级且可观测
   - S13：知识库 reload 与查询并发不崩溃
2. 新增或更新压测脚本（若仓库无现成脚本）。
3. 产出 `docs/service-transformation/phase1.5/devB-gate3-materials.md`，包含测试与压测证据摘要。

## 7. 日志与观测要求
1. 日志字段契约遵循 `log-field-dictionary-phase0.5.md`。
2. 关键日志至少包含：
   - `trace_id`, `request_id`, `user_id`, `goal_id`, `error_code`
   - `brand_id`, `collection_type`/`collection_name`
3. 冷启动事件要求：
   - 加载开始（info）
   - 并发等待/复用（warning 或 info）
   - 加载成功（info，含耗时/条数）
   - 加载失败并降级（error，含 error_code）
4. 禁止静默吞错；允许降级但必须有可检索日志证据。

## 8. 协作接口（与 DevA / DevC）
1. DevA 提供“BM25 修复一致性影响评审”结论，DevB 按结论约束实现。
2. DevB 提供可复现日志样例、加载次数统计口径给 DevC。
3. DevC 负责 Gate3 验收主测与签字，DevB 负责修复与补证据。

## 9. 验收标准

### 9.1 Gate3（Phase1.5 收尾）
1. `backend/tests/integration/test_bm25_cold_start.py` 通过。
2. 冷启动压测完成并有报告。
3. 日志可证明加载次数受控与降级路径存在。
4. 多 worker 场景首波延迟稳定，无崩溃。

### 9.2 回归基线（DevB 范围）
1. 检索主链路测试通过（Knowledge Retriever/API/Tool 相关）。
2. 不引入现有检索能力回归（结果格式与错误语义保持兼容）。

## 10. 执行后必须提交的产物
1. 代码改动（并发保护 + 降级 + reload/query 并发安全）。
2. 新增测试文件与结果摘要（命令 + 通过数）。
3. 冷启动压测报告与加载次数日志证据。
4. `docs/service-transformation/phase1.5/devB-gate3-materials.md`。
5. 根目录 `version.md` 更新记录。
6. 一段简短中文提交说明（1-3 行）。

## 11. 建议测试命令
1. `python -m pytest backend/tests/integration/test_bm25_cold_start.py -q`
2. `python -m pytest backend/tests/knowledge/test_retriever.py -q`
3. `python -m pytest backend/tests/test_knowledge_api.py -q`
4. `python -m pytest backend/tests/tools/test_knowledge.py -q`

## 12. 执行 checklist
- [ ] 已在改代码前提交“理解 + 计划”并获得确认
- [ ] 已完成 BM25 冷启动并发保护
- [ ] 已完成 reload/query 并发安全
- [ ] 已完成失败降级与 error_code 可观测
- [ ] 已新增并通过 `test_bm25_cold_start.py`
- [ ] 已产出 Gate3 材料（压测 + 日志 + 结论）
- [ ] 已更新 `version.md`
