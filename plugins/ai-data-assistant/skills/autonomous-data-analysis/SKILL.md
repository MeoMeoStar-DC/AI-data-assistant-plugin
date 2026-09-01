---
name: autonomous-data-analysis
description: 使用 AI 数据助手 MCP 自主调查业务问题、发现 Redshift 表与字段、预览 EXPLAIN 成本并执行参数化只读 SQL。适用于用户要求问数、自由下钻、排行、对比、归因、异常调查或验证数据假设，同时允许按需参考正式指标、Tableau 和 ETL 证据。
---

# 自主数据分析

主动完成合法的只读业务分析。正式参考用于提高准确率，但参考缺失、过期、失败或与 Redshift 不一致时，继续使用可验证的只读路径，并披露差异。

## 工作流

1. 从问题提取指标原词、期间原文、时区、时间桶、维度、筛选、人群、比较侧和输出要求，只调用一次 `prepare_analysis` 固定 Runtime 快照、正式义务和 `plan_signature`。同一请求即使仍在等待后续规划，也不得用相同参数重复调用；直接复用首次返回的不可变合同。`metric_mentions` 保留用户的 Cash/真金等限定，不要先改写成可能丢失范围的指标 ID。用户使用“昨天/前一天”“最近 N 个 UTC 业务日”“最近 N 个已成熟 cohort”“最近 N 个完整周”等相对期间时，将包含比较侧的完整原文放入 `period_expression` 且不自行填入日期；宿主按 Runtime 业务日、指标成熟滞后和 UTC 自然周确定性解析。用户明确绝对日期时才传 `start_date` 和 `end_date`。不要猜测会实质改变结果的口径。
2. 只为未解析义务或必要的公式、维度、字段和血缘调用参考工具。`get_metric_reference` 默认使用 `detail=summary` 和小 `limit`；精确命中后停止扩大候选，只有确需公式或来源合同时才读取 `full`。
3. 后续 Tableau 和 Redshift 查询都传入 `prepare_analysis` 返回的 `analysis_id`。UTC 合同下，兼容的 2 至 8 个 Tableau 指标优先调用一次 `query_tableau_reference_metrics`；正式来源、期间、筛选及 coverage 完整且 `evidence_role=authoritative_result` 时直接使用结果，不追加无信息增益的 Redshift 验证。显式非 UTC 合同不得调用 Tableau。
4. 正式资料不足时调用 `search_redshift_tables` 和 `describe_redshift_table` 验证实际结构。ETL 精确目标表已返回完整声明列且 `requires_live_describe=false` 时不重复 describe。只使用允许 schema，优先 DWS 和最低必要明细层。
5. 使用命名参数和显式列投影编写 `SELECT` 或 `WITH`。同一 SQL 结构仅参数不同的单元先改写成一次分组查询；无法证明等价时才使用 batch，普通分析软预算为 8 个执行单元。
6. 单计划调用 `preview_redshift_readonly`，多个不可合并计划调用 `preview_redshift_readonly_batch`。用于回答结论的执行单元传 `coverage_mode=result`，并传入它实际产出的非空 `coverage_obligation_ids`；多义务分析不得省略，也不得把未由该 SQL 产出的义务一并声明。仅做结构验证、对账或来源可用性检查的执行单元传 `coverage_mode=validation` 和空 coverage；其 `validation_result` 证据不得用于完成正式义务。宿主会在 EXPLAIN 前拒绝缺失、空或未知的结论义务 ID，修正同一计划后可继续，不得因此关闭已有结果。preview 成功后、任何 `start` 或兼容 `execute` 前，先在 commentary 或等价用户可见消息中按执行单元展示：完整 `sql` 代码块、命名参数与非敏感实际值、敏感值脱敏说明、目标表、实际日期、时区、结果粒度、筛选、测试用户策略、预计结构和 `row_limit`。最终结果、validation、跨表、用户去重、比率重算、非 UTC 和自定义指标 SQL 必须逐段解释主要 CTE、关联键、分区条件、聚合公式、分子分母及防重方法；metadata SQL 可简述但不能省略完整 SQL。
7. 展示后调用 `record_sql_disclosure`，传回 preview 的 `analysis_id`、`disclosure_id` 和 `sql_digest`。SQL 展示不是用户审批门禁，登记后继续执行；遗漏时先补展示，不能因此丢弃可信事实或直接失败关闭。默认使用 `start_redshift_readonly` / `start_redshift_readonly_batch`，再用 `get_redshift_execution_status` 和 `get_redshift_execution_result` 分页取得完整行。同步 `execute` 仅为明确暴露该能力的兼容入口。只传回未过期 token，不在执行阶段替换 SQL、参数或限制。
8. 用户取消、请求事务失效或结果已不再需要时，对仍为 `queued` / `running` 的 job 调用 `cancel_redshift_execution`。只有返回 `cancelled` 才视为仓库取消已确认；返回 `completed` 时继续使用已完成结果，返回 `failed` 时披露取消失败，不重复无界取消。
9. 成本失败先按恢复建议合并计划、缩短期间、增加选择性筛选或改用汇总层。同一个 `failure_signature` 没有新表、分区、连接或谓词证据时不得重试；保留其他成功子计划。
10. 使用实际 row set 收口，并在最终回答前调用一次 `finalize_analysis`，只选择实际支撑结论的 `evidence_id`，如实传入限制。以服务端 OutcomeEnvelope 的 coverage、状态、事实哈希和 disclosure 状态为准；工具不可用时可兼容交付旧结果，但必须明确 `outcome_not_recorded`。完整时直接交付，部分成功或外部能力缺失时使用 `completed_with_limitations` 交付可信部分，不猜字段、不补零、不反复扫描明细层。

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
