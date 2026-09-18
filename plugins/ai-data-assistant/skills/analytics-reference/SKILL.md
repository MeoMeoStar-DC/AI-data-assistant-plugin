---
name: analytics-reference
description: 检索 AI 数据助手的正式指标、维度、Runtime、Tableau、ETL 参考，以及 maintain-metric-dictionary 下载的原始 ETL catalog、Tableau Cloud 清单和 TWB/TWBX XML 元数据。适用于核实公式、字段、血缘、看板对象、连接结构或治理快照，不用于修改正式字典。
---

# 分析资料检索

根据问题选择最小必要证据，区分正式参考、解析目录和原始下载资料。

## 正式参考

- 使用 `get_metric_reference` 查询正式公式、单位、依赖、来源和允许维度。默认读取 `summary` 和少量精确候选；只有确认需要完整公式或依赖时才使用 `detail=full`，大结果按 cursor 继续。
- 使用 `get_dimension_reference` 查询维度定义和同义词；只有问题需要枚举值时才设置 `include_observed_values=true`。
- 使用 `get_runtime_reference_status` 检查 snapshot、业务日和 READY；异常只作为 freshness warning。
- 使用 `search_tableau_catalog` 与 `search_etl_reference` 检索经过解析或发布的参考目录。
- 仅查定义、公式、字段或血缘时，不调用 `prepare_analysis`、Tableau 数值查询或 `finalize_analysis`。用户需要业务数值时，由同包 `autonomous-data-analysis` 接管完整查询和收口流程；已有有效 `analysis_id` 时传递该合同与资料证据，不重复准备。若宿主没有加载该 Skill，先读取同包 `../autonomous-data-analysis/SKILL.md`，不要另建简化业务查询流程。

## 权限与交付

按当前实际可用工具选择路径，不为每次参考查询重复检查登录。原始资料仅在已授权 `metadata.raw.read` 且工具可用时读取；账号角色名称不能替代实际权限。raw 工具不可用时，继续使用正式指标、维度和允许的解析目录，说明未验证的原始血缘，不反复调用越权工具或判定登录失败。只有整体连接异常时使用同包 `assistant-connection`。

交付先给出定义或核对结论，再列单位、公式、适用范围和证据来源；区分正式口径与原始观测差异，不把目录字段或原始资料当成业务数值结果。未取得证据的字段明确标为未验证，不补造公式。

## 原始资料

1. 使用 `list_raw_metadata_snapshots` 获取完整、非 staging 的 run；省略 run ID 时读取最新完整 run。
2. 使用 `read_raw_etl_catalog` 按 JSON Pointer、目标表、任务、字段或血缘读取原始 ETL 对象，并用 cursor 继续分页。
3. 使用 `read_raw_tableau_metadata` 读取 `tableau-cloud.json`、解析 catalog 对象、TWB XML 或 TWBX 内的 TWB XML。按 workbook、datasource、worksheet、dashboard、field、formula 或 `xml_path` 缩小范围。
4. 保留响应中的 run ID、源文件、SHA-256、总字节数和分页信息，引用证据时说明来自原始观测层。

只读取当前分析义务所需的最小对象，不下载与问题无关的整份指标字典、ETL catalog 或 Tableau XML。大型结果使用 cursor 分页并在证据足够后停止。

原始资料不经过正式指标或 Runtime 摘要，但接口仍会确定性脱敏连接凭据。不要寻找或恢复 Secret、PAT、token、password、Access Key、身份隐私字段或 Extract 数据；不要用原始资料自动写入正式指标字典、catalog 或 Runtime。
