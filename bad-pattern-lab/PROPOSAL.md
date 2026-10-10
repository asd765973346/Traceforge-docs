# Bad Pattern Lab：单轨迹诊断与跨轨迹聚类

版本：traceforge-v2。本文是当前架构和实施契约的唯一权威说明；旧的分流→批量归因→独立 Review 三阶段方案由本方案取代。

## 本次改动

每条完整 StandardTrace 只启动一个诊断主会话：判断失败来源；如果是 model_failure，继续分析错误事件、因果机制、提取单条候选 Pattern，并在同一父会话内审查与修正。非模型问题或无法判断时，写出判断依据、证据缺口，结束该条分析。随后另开一个聚类会话，读取全部已完成诊断产物，合并机制并关联来源轨迹。

不使用 resume；不设置 max_turns、max_budget_usd 或固定 Review 迭代次数。代码不按步数/字符/token 截断原轨迹。聚类读取诊断档案而非重新预填全部原轨迹，完整轨迹以只读文件引用保留供回查。

## 会话与职责

- Diagnosis：一次 query(prompt, options)，主模型默认 glm-5.3。完整轨迹、明确输出 schema 与 TRACE_ANALYSIS_SKILL 一次输入。可调用 evidence_analyst 和 pattern_reviewer 两个只读 sub-agent，默认 glm-5.2，由 --sub-model 配置。主 agent 可以自审或委派审查，但必须记录实际审查方式、问题、修订与最终裁决。sub-agent 有自己的上下文，委派时必须传入候选和完整原轨迹文件路径。主会话只有在证据足够或明确无法补齐时自行结束，不以固定轮数结束。
- Clustering：全部单条任务结束后一次新 query，读取所有成功诊断、单条 Pattern、审查记录和完整原文路径。比较触发条件、错误机制和证据，不能只按描述措辞相似合并。所有可聚类候选都必须被分配或明确排除；不复用单条诊断聊天。
- 异常：SDK/解析/契约错误记为 error，不冒充 unknown。unknown 是模型给出的有效诊断。默认一次任务尝试，失败项下次执行重做；单条报错不丢掉其他成功结果，聚类明确报告输入覆盖不完整。

## 产物契约（实现以 analysis_contracts.py 为准）

Diagnosis 顶层包含 task_summary、outcome_summary、classification、errors、first_error_id、critical_error_id、patterns、limitations。

classification 包含 primary_cause（五类）、rationale、evidence[]、alternative_explanations[]、missing_information[]。每条 evidence 固定为 step_id、field、quote、supports；field 是某步内的 JSON Pointer（例如 /message 或 /observation/results/0/content）。程序确认步骤存在、field 指向文本、quote 是该原文或其严格 JSON 解码后字符串叶子的非空子串。工具观察常把 JSON 对象序列化为 content 字符串，因此允许这种无损解码；不做 HTML 解码、任意 Unicode 转换、空白或缩进折叠，也不自动修正错误步骤和 field。

errors[] 每项有 error_id、step_id、category、available_information、expected_action、actual_action、causal_explanation、recovery、evidence[]。recovery 明确 recovered/unrecovered/unknown、rationale、evidence[]。程序确保事件步骤有引用依据。first_error_id 指向时间上首个可确认错误，critical_error_id 指向对失败有决定影响的关键错误；model_failure 时两者必须引用 errors 中存在的事件，可以指向同一事件。

patterns[] 每项有 pattern_id、description、trigger、mechanism、consequence、error_ids、evidence[]、counterevidence[]、review。pattern_id 是单轨迹内局部 ID，聚类用 (trajectory_id, pattern_id) 唯一定位。review 包含 mode（self/sub_agent）、verdict（supported/needs_evidence/rejected）、rationale、challenges[]、revisions[]、downstream_use、remaining_gaps[]。challenge 含 claim、finding、evidence[]、resolution。supported 仅表示单条轨迹证据支持，不表示跨任务复现或可直接导出。非 model_failure 的 errors/patterns 必须为空；model_failure 必须记录错误，若无可成立 Pattern，limitations 必须解释原因。supported 不可带未解决 remaining_gaps。

Cluster 结果包含 clusters[]、excluded_patterns[]、summary。每簇有 cluster_id、description、trigger、mechanism、consequence、members[]、merge_rationale、limitations[]。每个 member 指定 trajectory_id、pattern_id、rationale，只允许引用审查 supported 的候选。needs_evidence/rejected 产物仍提供给聚类用于反证和边界，不能算作正向支撑。每个 supported 候选恰好出现在一个簇或一个明确排除项；非 supported 候选不进入分配集合。excluded_patterns 每项为 trajectory_id、pattern_id、reason。程序重算 frequency（唯一轨迹数）、supporting_trace_ids、independent_task_count（bench+task_id 去重；未知 task_id 不推定独立），不允许模型自行报大数字。不把 frequency 当作总体发生概率。

## 文件与恢复

run.py 写出 run_config.json、traces/<安全哈希>.json、diagnoses/<安全哈希>.json 与同源 Markdown、analysis_checkpoint.jsonl、analysis_results.jsonl、patterns.json、patterns.md、run_record.json。原始 ID 不用于拼文件路径。JSON 是事实来源，Markdown 用确定性渲染，不额外调用模型。SDK 元数据包括 session_id、model_usage、Agent 调用角色记录；不声称保存所有分析对话。

配置指纹包含 schema/runner/skill 版本、模型、benchmark、输入内容、凭据端点（不保存 key）。已完成且输入/契约匹配的成功项可复用；失败重做。输出目录不兼容时要求使用新目录；旧 triage checkpoint 不作为新诊断结果。聚类失败先落盘成功诊断与 run_record 的失败状态，移除本次可能遗留的旧聚类结果，避免旧结果被误当成新结果。没有 supported 候选则程序写空聚类，不调用模型。

## 文档维护

权威 Markdown 是 docs/bad-pattern-lab/PROPOSAL.md；交互 index.html 必须由当前契约与 skills 生成。README 仅放用途、安装运行和权威说明链接。旧 bpl-execution-guide 与 HANDOFF 只保留指向此处的迁移入口，不留复制的 prompt/schema/source 快照。原 data-infra 文档入口与 Traceforge 代码仓库使用同一份受维护内容，同步时检查一致性。

## 验收

通过 mock SDK 的工作流验证一条轨迹只有一次 query、非模型问题早结束、错误引用原文校验、审查结果传播、所有诊断进入聚类输入、supported 分配完整性、唯一轨迹/独立任务计数、失败状态、checkpoint 失效、中文报告保留完整产物。既有凭据隔离和数据过滤测试保留。真实 GLM 端点行为与节省 token 的幅度需另行验证，离线测试不代表真实质量评测。

## 公网文档发布

公网地址：https://asd765973346.github.io/Traceforge-docs/bad-pattern-lab/ 。站点由单独的公开文档仓库 Traceforge-docs 发布到 GitHub Pages，代码仓库保持私有。生成页面后执行 `python tools/publish_docs.py`：先检查文档与源码一致，再只同步 index.html 和 PROPOSAL.md，推送后由文档仓库 Actions 部署。公开文件是从权威正文生成的发布产物，不在公开仓库独立编辑；不发布凭据、真实轨迹或运行结果。

部分兼容模型通过父会话 StructuredOutput 工具块返回 JSON，而 ResultMessage 的 structured_output 为空。runner 仅在对应父会话工具调用收到成功 tool_result 确认后使用该回退路径，之后仍执行完整 schema 与原文证据校验；未通过校验的模型产物仍记为 error。

## 单条会话：输入、每一步工作与结束条件

诊断的 user prompt 包含一条完整 StandardTrace JSON 和该轨迹的只读文件路径。system prompt 来自 TRACE_ANALYSIS_SKILL，存在对应 benchmark skill 时追加它；JSON Schema 通过 SDK 的 output_format 约束最终返回。模型看到的是完整轨迹，不只是最终低分。真实固定 prompt、完整 skill 与 schema 可在同目录交互页的“Prompt / Schema 原文”查看，它们从源码生成。

### 1. 判断失败来源

先读任务目标、评分证据、工具返回与终态，区分 model_failure、task_defect、env_failure、judge_failure、unknown。classification.rationale 写清决定性证据与判断范围，alternative_explanations 记录其他解释，missing_information 记录缺失材料。只有零分或本地测试通过不足以推出模型有错或评分器有错。

不是模型问题，或者只能判 unknown 时，当前会话直接输出诊断档案，errors 与 patterns 为空；这些档案依然会被保存并提供给聚类作背景。程序错误或请求失败会另记 error，不能伪装为证据不足的 unknown。

### 2. 定位错误和因果关系

model_failure 才继续。逐条错误记录当时可用的信息、预期动作、实际动作、因果解释与恢复情况。引用必须定位到一个存在的 step_id、步内 JSON Pointer 和逐字原文。把“模型本该知道”的证据与“事后评审才知道”的信息分开；把已恢复的早期错误与导致最终失败的关键错误分开。

例如：步骤 8 的 /observation/results/0/content 明确给出库存已预留，但步骤 9 的动作按总库存承诺交付。档案应引用步骤 8 的原文和步骤 9 的实际动作，解释预留信息为何在当时可用，后续是否修正，以及这个动作如何导致任务失败。该例仅解释字段含义，不是真实运行结果。

### 3. 提取单条候选 Pattern

从错误事件提炼 description、trigger、mechanism、consequence，保留 error_ids 和证据映射。描述需要脱离偶然的公司名或具体数字，仍保留使错误发生的条件。不能用“推理不足”“工具使用差”代替机制；不能因为一条轨迹支持，就宣称该机制普遍存在。

一条轨迹可以有多个候选，也可以在模型有错的情况下没有足够清晰的 Pattern；后一种情况必须在 limitations 解释。单条局部 pattern_id 与 trajectory_id 组合后才是全局唯一候选引用。

### 4. Review、修订和停止

主 agent 选择 self 自审或调用 pattern_reviewer。子任务拿到候选描述、错误记录和完整原轨迹路径，主动寻找反证、恢复行为、其他原因与过度泛化。它有自己的子上下文，不能假设自动继承父会话全部历史。

审查意见回到当前主会话。主 agent 根据具体 challenge 回查证据、修改候选并记录 revisions，可以再次委派或自审；不新建一个顶层 Review 会话，不在 Python 外层设置固定审查轮数。mode 记录实际采用的方式；声称 sub_agent 时运行记录必须有 pattern_reviewer 的实际调用。

supported 表示当前单条证据足够；needs_evidence 表示具体缺口尚未解决；rejected 表示候选被证据否定。主 agent 在问题已经解决，或确认现有材料无法补齐并明确记录限制后停止。无法取得的新材料不是无休止重复核查的理由。supported 的 remaining_gaps 必须为空，但即便 supported，也不直接表示已完成新环境复现或下游正式准出。

## 聚类会话：怎样复用前面的工作

聚类 user prompt 包含全部成功诊断档案、完整原轨迹文件路径和失败轨迹 ID 清单。前一阶段的分类、错误分析、候选、Review 与反证都在输入里；聚类不需要从一个空 notes 重新做单条归因。主 agent 有只读工具，存在疑问时可以回查完整原文。完整原文不会再次全部内联到聚类 prompt，因此节省重复输入；模型实际是否节省 token 需要从 usage 测量。

聚类以 trigger 与 mechanism 为主比较，description 只帮助发现候选。合并必须写 merge_rationale，说明共享机制和保留的限制；相似字面描述不足以合并。未审查通过的候选可以暴露边界和反证，但不作为支撑成员。

每个 supported 候选必须出现一次：进入一个 cluster，或者进入 excluded_patterns 并解释为何排除。一个簇可以只有一条轨迹，这只表示当前尚无更多支撑；不强行设定最小样本数，也不制造复现结论。同一轨迹的多个成员只算一次 frequency。同一 bench 下相同 task_id 即使由不同 harness 执行，也只算一个已知独立任务；未知 task_id 不凭空认定独立。

所有单条诊断失败，或成功诊断中没有 supported 候选时，程序直接写空聚类，不启动无输入的模型会话。部分诊断失败时，成功档案仍然保留，结果必须声明覆盖不完整。频次是当前样本中的支撑轨迹数，不是总体失败概率。

## 一轮运行到底开多少个会话

假设输入 5 条轨迹，其中 3 条非模型问题、2 条存在模型错误并产出 supported 候选：共开启 5 个单条诊断主会话，再开启 1 个聚类主会话，总计 6 个顶层会话。Review 或证据子任务不增加顶层诊断次数，但它们有独立的子上下文和 token 用量。

如果 5 条都没有 supported 候选，只开 5 个诊断主会话，聚类由程序输出空结果。再次运行兼容的输出目录时，复用已成功完成的单条产物；错误项重新开新会话，聚类依据本次完整可用产物重新计算。没有恢复任何旧聊天上下文。

--concurrency 控制同时运行的单条主任务数量，--max-traces 选择输入样本数量。这两个参数都不限制单个 agent 的工具轮次、审查次数或金额。工具调用、子任务和主模型继续判断都发生在 SDK 会话生命周期内，直到最终 ResultMessage 或异常。

## 阅读产物

先看 run_record.json 的运行状态、错误与覆盖，再看 analysis_results.jsonl 中每条诊断；diagnoses/ 中同源 Markdown 适合逐条阅读证据和审查记录。跨轨迹结论看 patterns.json 与 patterns.md；完整原文从对应 trace_path 找到 traces/ 文件。

JSON 是事实来源；Markdown 由程序确定性渲染，不做第二次模型改写。保存的审查说明与因果依据是可复核结果，不声称包含模型全部内部推理、SDK 全部历史消息或全部子任务 transcript。旧 triage_results/triage_checkpoint 以及旧截断版本的 glm53-sdk-100 不能视为本版本诊断与聚类结果。
