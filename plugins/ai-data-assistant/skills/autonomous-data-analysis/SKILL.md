---
name: autonomous-data-analysis
description: 使用 AI 数据助手 MCP 自主调查业务问题、发现 Redshift 表与字段、预览 EXPLAIN 成本并执行参数化只读 SQL。适用于用户要求问数、自由下钻、排行、对比、归因、异常调查或验证数据假设，同时允许按需参考正式指标、Tableau 和 ETL 证据。
---

# 自主数据分析

主动完成合法的只读业务分析。正式参考用于提高准确率，但参考缺失、过期、失败或与 Redshift 不一致时，继续使用可验证的只读路径，并披露差异。

## 按任务读取分析方法

普通取数直接进入下方工作流；深度分析按需读取对应参考，不为简单问题加载全部材料，也不重复建立分析合同。

- 解释变化、异常或人群结构差异：读取 [指标诊断](references/metric-diagnostics.md)，在同一合同内核实前提、选择有信息增益的下钻并验证候选解释。
- 缺失、重复、关联放大、来源冲突可能改变结论：读取 [数据质量](references/data-quality.md)，只检查影响当前结论的风险。
- 多阶段分析收口或用户要求复核报告：读取 [分析验证](references/analysis-validation.md)，对照原问题检查重要数值、证据和实际交付。

计算能力以当前实际暴露的工具为准；没有计算工具时继续用已有可信查询结果交付。不得调用未暴露的工具或将业务数据上传到未经授权的计算环境。

## 可选统计计算

只有当前服务暴露 `prepare_analysis_dataset` 和 `start_analysis_compute` 时才进入此流程。大规模聚合仍优先在受治理 SQL 层完成；计算仅用于已有结果上的分布、分位数、变化贡献、置信区间和异常复核。

1. 向 `prepare_analysis_dataset` 提交本合同的 `evidence_ids` 和所需 `fields`；输入必须来自服务端记录，不能上传自行重写的表格。检查返回的行数、截断及字段，数据不完整时限定结论范围。
2. 向 `start_analysis_compute` 提交数据集引用、Python 代码、方法说明、参数和固定 `seed`。代码读取 `dataset['tables']`（各表含 `rows`、期间、人群筛选和来源信息）及 `parameters`；最终赋值 `result={'findings': [...], 'limitations': [...]}`。每项发现写明量化结果、方法、适用人群与期间；图表可保存为当前目录的 `chart.png`。不要从网络补数据、修改原始输入或读取凭据。
3. 用 `get_analysis_compute_status` 读取同一任务，取消时调用 `cancel_analysis_compute` 并核对终态；错误、超时或取消失败要明确披露。已有可信查询结果先交付，辅助计算失败不触发整题拒答。
4. 对实际使用的产物按 `read_analysis_artifact` 的 `next_offset` 完整读取，核对整体哈希后再交付 Notebook、结构化结果、输入、参数、执行环境及可选图表。产物不可用不能声称附件已交付；`historical=true` 只能作为历史记录。
5. 收口时把实际使用的 `derived_id` 放入 `finalize_analysis.selected_derived_ids`，原始 `evidence_id` 仍单独提交。`analysis_validation` 与查询 coverage 分别解释；`execution_recorded` 只证明记录了执行，`method_review_required=true` 表示仍需复核统计方法、分母、样本、期间和结论强度，不能当作业务准确性通过。

置信区间明确抽样单位、假设、方法和置信水平，短样本披露不确定性；变化贡献不是因果解释。比率重算、跨日去重与关联防重继续遵循原合同，不通过 Python 绕过口径。

## 工作流

游戏指标参数约定：对游戏次数、游戏用户数、投注金额、赢分、RTP、GGR、NGR，使用 `game_metric_requests` 将 `mention`（须与metric_mentions中的原词一致）、`concept` 和 `currency_scope` 分开传入。范围枚举cash/coin/split/combined分别表示Cash、Coin、分开展示、明确合并；不能因限定词位置变化丢失范围，也不能把用户未要求的混合金额标成美元。若返回 `game_scope_requires_structure`，尚未建立有效合同，须补全game_metric_requests并重新准备；与game_mode筛选冲突时先核对完整请求并修正同一准备参数；不要原样重试或删除限定。

相对期间参数约定：单一窗口优先填写 `relative_period` 的 `unit`（day/week/month/year）、`count`、`offset`（向历史偏移的单位数）、`alignment`（rolling/complete_calendar）和 `mature`，并用 `period_expression` 保留原文证据。宿主计算实际日期，调用方不得自算相对日期或传成熟滞后天数。单个relative_period只表示一个窗口；多窗口用period_sides逐侧填写side_id、role及相对期间或绝对日期，唯一current侧与各comparison侧保留独立范围。不得再传顶层日期覆盖各侧。Tableau逐侧使用对应business_period查询，所有指标×侧义务均有实际证据后才可完整交付；不要把单侧证据声明为覆盖全部侧。Redshift会核对已声明主来源的实际date字段及单来源日期谓词，以period_evidence报告证明状态；未改变范围的单来源CTE投影/聚合可沿来源链继承证明；其他子查询、联接、外层范围修改及日期条件聚合等尚需进一步证明。未验证时保留已有结果，按诊断重规划成可验证的逐侧查询或使用等价受治理来源；不能仅凭coverage_obligation_ids宣称多侧完整。若旧入口返回 `period_expression_requires_structure`，表示尚未建立有效合同；重新理解完整范围后修正参数再准备，不受有效合同禁止重复准备的限制。不得原样重试或把工具参数修复转嫁成无必要的用户澄清；真正存在范围歧义时才一次性澄清。

1. 从问题提取指标原词、期间原文、时区、时间桶、维度、筛选、人群、比较侧和输出要求，同一有效请求合同内只调用一次 `prepare_analysis` 固定 Runtime 快照、正式义务和 `plan_signature`。同一请求即使仍在等待后续规划，也不得用相同参数重复调用；直接复用首次返回的不可变合同。用户改变期间、维度、筛选或目标，或服务端明确提示合同失效时，重新准备合同；旧 evidence、preview token 和 job 不自动绑定新合同，重新验证适用性，按新合同重新预览必要查询。`metric_mentions` 保留用户的 Cash/真金等限定，不要先改写成可能丢失范围的指标 ID。用户使用“昨天/前一天”“最近 N 个 UTC 业务日”“最近 N 个已成熟 cohort”“最近 N 个完整周”等相对期间时，将包含比较侧的完整原文放入 `period_expression` 且不自行填入日期；宿主按 Runtime 业务日、指标成熟滞后和 UTC 自然周确定性解析。用户明确绝对日期时才传 `start_date` 和 `end_date`。不要猜测会实质改变结果的口径。
2. 只为未解析义务或必要的公式、维度、字段和血缘调用参考工具。`get_metric_reference` 默认使用 `detail=summary` 和小 `limit`；精确命中后停止扩大候选，只有确需公式或来源合同时才读取 `full`。
3. 后续 Tableau 和 Redshift 查询都传入 `prepare_analysis` 返回的 `analysis_id`。UTC 合同下，单指标可调用 `query_tableau_reference_metric`；兼容的 2 至 8 个 Tableau 指标优先调用一次 `query_tableau_reference_metrics`；正式来源、期间、筛选及 coverage 完整且 `evidence_role=authoritative_result` 时直接使用结果，不追加无信息增益的 Redshift 验证。显式非 UTC 合同不得调用 Tableau。
4. 正式资料不足时调用 `search_redshift_tables` 和 `describe_redshift_table` 验证实际结构。ETL 精确目标表已返回完整声明列且 `requires_live_describe=false` 时不重复 describe。只使用允许 schema，优先 DWS 和最低必要明细层。
5. 使用命名参数和显式列投影编写 `SELECT` 或 `WITH`。同一 SQL 结构仅参数不同的单元先改写成一次分组查询；无法证明等价时才使用 batch，普通分析软预算为 8 个执行单元。
6. 单计划调用 `preview_redshift_readonly`，多个不可合并计划调用 `preview_redshift_readonly_batch`。用于回答结论的执行单元传 `coverage_mode=result`，并传入它实际产出的非空 `coverage_obligation_ids`；多义务分析不得省略，也不得把未由该 SQL 产出的义务一并声明。仅做结构验证、对账或来源可用性检查的执行单元传 `coverage_mode=validation` 和空 coverage；其 `validation_result` 证据不得用于完成正式义务。宿主会在 EXPLAIN 前拒绝缺失、空或未知的结论义务 ID，修正同一计划后可继续，不得因此关闭已有结果。preview 成功后、任何 `start` 或兼容 `execute` 前，先在 commentary 或等价用户可见消息中按执行单元展示：完整 `sql` 代码块、命名参数与非敏感实际值、敏感值脱敏说明、目标表、实际日期、时区、结果粒度、筛选、测试用户策略、预计结构和 `row_limit`。最终结果、validation、跨表、用户去重、比率重算、非 UTC 和自定义指标 SQL 必须逐段解释主要 CTE、关联键、分区条件、聚合公式、分子分母及防重方法；metadata SQL 可简述但不能省略完整 SQL。
7. 展示后调用 `record_sql_disclosure`，传回 preview 的 `analysis_id`、`disclosure_id` 和 `sql_digest`。SQL 展示不是用户审批门禁，登记后继续执行；遗漏时先补展示，不能因此丢弃可信事实或直接失败关闭。默认使用 `start_redshift_readonly` / `start_redshift_readonly_batch`，再用 `get_redshift_execution_status` 和 `get_redshift_execution_result` 分页取得完整行。同步 `execute` 仅为明确暴露该能力的兼容入口。只传回未过期 token，不在执行阶段替换 SQL、参数或限制。
8. 用户取消、请求事务失效或结果已不再需要时，对仍为 `queued` / `running` 的 job 调用 `cancel_redshift_execution`。只有返回 `cancelled` 才视为仓库取消已确认；返回 `completed` 时继续使用已完成结果，返回 `failed` 时披露取消失败，不重复无界取消。
9. 成本失败先合并等价计划、利用原请求已有分区/筛选或改用等价汇总层；能力允许时按原期间分块并校验完整覆盖，不通过分块规避总体成本限制。不得默默缩短期间或增加改变人群的筛选。无法覆盖全请求时，只交付明确标注实际范围和缺失义务的部分结果，或向用户提出具体的缩小范围选择，不能宣称完整回答。同一个 `failure_signature` 没有新表、分区、连接或谓词证据时不得重试；保留其他成功子计划。
10. 使用实际 row set 收口，并在最终回答前调用一次 `finalize_analysis`，只选择实际支撑结论的 `evidence_id`，如实传入限制。以服务端 OutcomeEnvelope 的 coverage、状态、事实哈希和 disclosure 状态为准；工具不可用时可兼容交付旧结果，但必须明确 `outcome_not_recorded`。完整时直接交付，部分成功或外部能力缺失时使用 `completed_with_limitations` 交付可信部分，不猜字段、不补零、不反复扫描明细层。

## 路由、权限与业务交付

- 只查定义时使用 `analytics-reference`，不启动业务查询；分析中查资料复用当前合同。只调用当前已授权且可用的工具，raw 元数据缺失时优先正式参考和允许的结构发现，不把辅助权限缺失升级为整题失败。整体连接异常时使用 `assistant-connection`；正常分析不重复登录检查。
- 最终回答先给出直接回应问题的业务结论，再给关键量化证据、口径与限制。比较同时说明两侧实际值、差值和有意义的变化率；零分母、不同单位或不同人群不强行计算。
- 趋势覆盖请求的整体期间和主要变化，不能以峰谷或返回行数替代分析。归因区分已计算贡献、相关现象与待验证原因；没有因果证据不声称因果关系。用户提出多个比较或分析轴时逐项回答，未覆盖部分明确披露。
- 解释展示字段的业务含义、单位和派生计算；区分空值、缺行和零，不对标识符作无意义均值或极差。部分结果保留可信结论，但不掩盖期间缺口、截断和口径差异。

## 报告附录

- 报告、复盘、调查文档、Markdown 或 Word 默认在正文末尾添加“SQL 附录索引”，列出 query/unit ID、用途、来源、preview/执行状态、实际期间、参数说明、结果粒度、SQL digest 和对应结论。
- SQL 体量适中时按实际执行顺序嵌入完整 SQL 与关键逻辑解释。单条 SQL 超过 200 行或 16 KiB，或全部 SQL 明显影响正文阅读时，生成稳定相对路径的 `*-SQL附录.sql`；正文仍保留索引、摘要和相对链接。
- 附录只能使用实际 preview 的 SQL。preview 拒绝和执行失败可以进入索引，但必须分别标记 `preview_rejected`、`failed`、`executed` 或 `used_for_conclusion`，不能把失败尝试当作结论依据。
- 只有 Tableau 结果时生成“查询合同附录”，记录指标、workbook/datasource、日期、维度、筛选和 coverage，并明确底层 SQL 未由数据源接口提供；不得猜测 SQL。
- 附录中的 Secret、token、连接串、受保护身份和隐私参数继续确定性脱敏。

## 不变量

- 区分 `missing`、`null` 和 `zero`，比率先聚合分子分母再计算。
- 用户未指定时区时使用 UTC；Redshift 和 Tableau 的日期参数默认都是 UTC 日期。用户显式指定非 UTC 时区时，遵循 `source_requirements`，排除 Tableau/DWS 日期汇总，只在 ETL 或实时字段证据确认时间戳字段及源时区后查询 `dwd`/`ods`，使用参数化 `CONVERT_TIMEZONE(:source_timezone, :user_timezone, <verified_timestamp_field>)::date` 筛选。不得猜时间戳字段、源时区或静默使用本地日期。
- 游戏次数、投注次数、下注次数和 spin 次数统一按 `request_obligations[].business_resolution` 执行。Cash、真金、付费筹码对应 `game_mode=0`，金额单位 USD；Coin、免费筹码对应 `game_mode=1`，不标货币单位；未限定币种使用全币种 `bet_cnt`，但不得把 Cash 与 Coin 混合数值描述为美元。Cash 次数使用 `cash_bet_cnt`，Cash 游戏用户使用 `cash_bet_user_cnt`。Coin 次数或用户数没有正式顶层指标时，按 `source_plan` 验证 Redshift 聚合字段或 DWD 事件路径，不用全币种指标冒充。
- Redshift 按合同保留 `game_mode=0/1`；Tableau 不传 `game_mode` filter，只能使用已编码对应范围的正式指标。不存在等价正式 Tableau 指标时改走经验证的 Redshift 路径。
- `dim.game_info` 仅作为当前游戏配置维表使用。关联事实表或汇总表时，`JOIN ... ON` 必须同时且仅以 `dim.game_info.app = <fact>.app AND dim.game_info.game_id = <fact>.game_id` 两个等值键关联；不得把 `dim.game_info.game_mode` 放入 `JOIN`、`WHERE`、`HAVING`、`GROUP BY` 或币种分类表达式。Superman 的该配置字段只是统一货币体系映射（Cash=`0`、Coin=`1`、其他=`100`），Jimmy 游戏配置无此字段，因此不能用于跨产品币种判断。Cash/Coin 条件必须作用于经合同或实际结构验证的事实/汇总表 `game_mode`，或使用已编码对应范围的正式指标；无法取得等价字段时保留其他可信结果并说明限制，不得用 `dim.game_info.game_mode` 补推。
- 遵循 `calculation_contract` 区分来源：原始 `dwd.bet_detail_di` 的次数用 `COUNT(1)`，用户数按 `app + user_id` 去重并在关联注册快照前按用户周期预聚合；只有聚合数据集才对正式指标列求和。Cash 金额、输赢和 RTP 校验使用 `cash_bet_amount`、`cash_winscore`、`rtp_sc`，不得混入全币种指标。
- 所有需要去重的用户数/人数指标遵循 `user_identity_contract`：业务身份键固定为 `app + user_id`，不同 App 下相同 `user_id` 计为不同用户。已按 App 分组时可在组内按 `user_id` 去重；跨 App 汇总必须使用复合键。跨日区间去重不能用每日人数求和替代，必须在请求粒度重新去重。
- 不把 Tableau 参考失败升级为 Redshift 核心结果失败，也不把原始元数据当作正式口径。
- Tableau 单指标最多 1000 行，多指标协调结果按工具自身 limit；Redshift 最多 20000 行，不得混用来源行数合同或用 Top N 冒充全量。
- 原始 ETL/Tableau 元数据只在正式公式、字段或血缘证据不足时按当前义务小范围读取。
- 不绕过只读 AST、EXPLAIN 门禁、一次性 token、10 分钟、20000 行、64 列和 4 MiB 限制。
