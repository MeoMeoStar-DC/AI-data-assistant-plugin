# 分析结论与交付验证

用于多阶段调查结束或复核已有报告。模型自查是分析步骤，不等于服务端已验证或业务验收通过。

## 对照原请求与实际证据

将用户要求的指标、比较侧、维度、人群、时间范围和分析轴逐项对应到实际结果及 evidence 引用。继承的会话记忆、参考元数据和未执行 SQL 不能补齐业务事实。只选择真正支撑结论的证据调用入口规定的 `finalize_analysis`，不为辅助复核重复准备合同。

若收口返回缺失义务和 `query_coverage.recovery_actions`，先检查其他已有证据能否补齐，再按原合同的首选等价来源或逐侧期间恢复；新增有效证据后重新收口。不要重复相同证据的无变化收口，也不要仅把缺口写进限制便跳过仍可执行的恢复。两天合并查询的扫描期间不等于两个单日义务分别取得期间证明；派生计算和行内日期不能替代该证明。恢复仍无可信路径或预算耗尽时交付已有可信部分，并保留实际未完成状态。

## 复核重要计算

- 核对合计、差值、变化率、单位、精度和零分母。由实际行集或受控查询复算，不以重复生成同一句答案作为独立验证。
- 比率按正式分子分母重算；跨日用户数不能相加；分组有重叠时不得当作互斥构成。总计不能与其明细重复计数。
- 区分占总体比例与占变化比例。总变化接近零、贡献相互抵消或样本被截断时，选择有业务意义的绝对量并解释限制。
- 只有执行过的统计方法才能报告置信区间或显著性；披露独立样本单位、抽样/分组方式和方法假设。探索性切分未经验证不能当作确认性结论。

发现错误优先重算或修正对应结论；无法验证时清楚标记该项，保留已有权威结果。不能把模型生成的辅助分析表当作权威主表，也不能用辅助验证状态改写原始查询 coverage。

## 检查用户真正收到的内容

主结论回应业务问题并包含量化证据；查询步骤、来源说明和口径放在对应详情。变化贡献、相关关系、因果与建议明确区分。

核对可见表格、图表和附件的期间、行列、单位、排序、精度与正文一致。附件或图表尚未生成时不能声明已交付；不能检查实际渲染时记录这一限制，不伪称视觉验收通过。没有 Notebook 能力时保留现有 SQL/查询合同附录，不创建假的 Notebook 引用。

最后给出可信结论、证据与适用范围、仍缺少的关键事实和具体下一步。部分成功按实际 OutcomeEnvelope 交付，不能将核心义务缺失描述成完整回答。业务决策提示：数据仅供参考，需要人工核对。

## 参考来源与本地适配

方法参考：[OpenAI validate-data](https://github.com/openai/role-specific-plugins/blob/fe5608d2512a7d6a7b9821ce8a88c48464ecd6e4/plugins/data-analytics/skills/validate-data/SKILL.md)，固定提交 `fe5608d2512a7d6a7b9821ce8a88c48464ecd6e4`；该提交仓库许可证为 MIT。本文为独立中文指引，未复制上游正文；以实际工具证据收口，不将模型自查当作宿主证明，不增加无恢复路径的整题门控。

## 结构化分析要求与算术复核

仅在当前服务 schema 提供 `analysis_requirements` 时使用。每项包含唯一 `requirement_id`、`operation` 和绑定基础指标原词的 `metric_mention`。`difference`、`relative_change`、`share_change` 还须指定已存在的 `comparison_side_id`；`share`、`share_change` 指定合同内 `dimension` 和 `denominator_scope=within_each_period_and_filters`。它们不会新增原始查询指标，也不能免除未知指标义务。

计算发现用 `requirement_id` 关联要求。需要宿主复核数值时，同时返回 `value` 和 `calculation={operands: {...}, decimal_places: 6}`。每个操作数引用本次数据集表的 `evidence_id`、已选择的 `field` 及从零开始的 `row_index`；对完整表显式求和可用 `aggregate: 'sum'` 替代行号。不要使用模型抄写的常量作为原始操作数。

| 操作 | operands 字段 | 输出单位 |
| --- | --- | --- |
| difference | current、previous | 原指标单位 |
| relative_change | current、previous | 百分数，按 `(current-previous)/previous*100` |
| share | numerator、denominator | 百分数 |
| share_change | current、current_total、previous、previous_total | 百分点 |

小数位允许 0–12，默认 6，按 Decimal 默认四舍五入复核。零分母返回 `value: null` 和 `value_status: 'undefined_zero_denominator'`，不能写成零。人数、比例及分类维度的列选择必须符合实际含义；`sum` 只在正式可加性及分组互斥已经确认时使用，跨日去重人数不得求和。

`requirement_results` 将未记录执行、输入覆盖不完整、执行已记录分开。`numeric_validation=passed` 只证明所引用值的算术一致性，不证明分母口径、分群完整性、统计显著性或因果。`method_review_required` 仍须披露；算术失败时检查引用和公式并在预算内修复，已有查询结果继续交付，不能把图表或执行成功当作方法验证。
