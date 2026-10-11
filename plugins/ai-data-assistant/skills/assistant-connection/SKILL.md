---
name: assistant-connection
description: 用于 Codex 桌面版或 Codex CLI 中已安装 AI 数据助手的首次授权、重新授权、登录失败、complete 后仍不可用、授权入口缺失、回调超时或工具不可见；正常问数不触发重复连接检查。
---

# AI 数据助手连接排障

目标是恢复现有插件入口，并以一次 DAU 正式定义读取确认可用。本 Skill 随插件本地安装，排障起步不依赖远程 MCP 工具；不要先要求调用尚不可用的远程工具来认证。

## 定位现有入口

- 核对实际登录客户端的版本、插件来源、启用状态与同名用户级 MCP。桌面版核对匹配后端；Codex CLI 核对实际执行版本。只读取必要配置字段，不读取凭据文件，不输出完整配置。
- Windows CLI `0.144.4` 曾出现错误回调端口和 resource，不用于登录。桌面后端 `0.154.0-alpha.6.2` 曾通过验证；Ubuntu Codex CLI `0.162.1` 与插件 `0.2.5` 已完成纯插件 OAuth 和 DAU 验证。不保证其他版本兼容，不猜路径或自动升级。
- 来源应为 `ai-data-assistant@ai-data-assistant-team`，地址为 `https://mcp.aidamms.com/mcp`。CLI `0.162.1` 的 `codex mcp list` / `get` 可显示插件服务器，这不等于重复配置；列表为空也不能单独证明未加载。结合用户配置判断混合入口，不删除未知配置或新增 MCP。

## 区分状态并恢复

- OAuth 未完成、登录入口消失，或网页显示 `complete` 但仍不可用：通过宿主实际提供的 OAuth 功能为已有入口重新发起登录并打开浏览器。用户已请求恢复连接时可直接执行；只有缓存阻止重新授权时，才退出该入口的 OAuth 后再登录。不得触碰其他连接。
- 桌面版：设置 → 插件 → AI 数据助手 → 打开 MCP 设置 → `ai-data-assistant` → 进行身份验证。Codex CLI 不需要桌面端，按下节执行。账号选择、注册和授权由用户本人完成，保持当前登录与回调进程运行。
- 实际客户端没有可用登录能力或未识别服务器时，报告版本、入口和脱敏错误；不虚构功能，不因为没有桌面界面就把可用的 CLI 登录判为不可用。
- 已认证但出现 `403` / `insufficient_scope`：让管理员核对实际账号角色及客户端允许的权限。角色更新后才重新授权；不自行扩大 scope 或修改 Auth0。缺少 raw 工具不能单独判定登录失败，普通成员通常没有 `metadata.raw.read`。
- 已认证且工具可用，但旧会话找不到工具：先新开会话加载插件，不反复退出登录。
- 一次重新授权后仍失败，停止同条件重试，报告脱敏错误、版本与入口来源。`too_many_entities` 应立即停止重试，不动态注册更多客户端。

## Codex CLI 原生授权

1. 核对 `codex plugin list --json`、marketplace 来源及 `codex mcp get ai-data-assistant --json`，确认服务器来自现有插件，不运行 `codex mcp add`。核对实际插件 `.mcp.json` 的公共 OAuth 回调配置，本次验证端口为 `14550`。
2. CLI 与浏览器在同一机器时直接登录；不同机器时，在浏览器所在电脑启动 SSH 转发并保持运行：

   ```text
   ssh -i "<SSH私钥路径>" -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -N -L 127.0.0.1:14550:127.0.0.1:14550 <远程用户名>@<远程主机地址>
   ```

   转发必须处于浏览器可访问的本机网络环境，不能只凭端口监听判断成功。仅绑定回环地址，不开放公网端口。Windows 后台启动用 `Start-Process -WindowStyle Hidden`，记录专用进程 PID。
3. 在运行 CLI 的机器执行 `codex mcp login ai-data-assistant`，保留该进程。无图形环境的浏览器启动失败不等于授权失败。使用该次客户端原生链接在本机浏览器打开；不追加 resource、scopes 或 DCR 策略，不修改回调、Client ID 或 PKCE 参数。
4. 不同机器时先用无授权参数的 `http://127.0.0.1:14550/` 测试两端 HTTP；本次返回 `400`，其他版本可能为 `404`，表示可达而非已授权。超时、`000` 或拒绝连接先修复转发；不使用含 code / state 的回调 URL 探测。
5. 浏览器回调 `ERR_CONNECTION_TIMED_OUT` 时排查转发，不能重装插件或修改 Auth0。修复后若原页面和登录仍有效，可刷新尚未送达的回调；页面关闭、登录过期或状态失效时，只结束本次旧登录，再生成一次新授权，不复用失效链接。同条件未修复不反复登录。
6. 等 CLI 提示登录成功，并确认新的 `codex mcp list` 显示 `OAuth`；凭据保存在 CLI 所在机器的 Codex 用户环境。随后执行最小验证，关闭本次专用 SSH 转发并删除临时登录日志；不退出刚保存的 OAuth。

## 最小验证与结果

仅在工具可用后调用一次 `get_metric_reference`，查询 `dau`，使用 `detail=summary`、`limit=1`。不调用 `prepare_analysis`，不执行 Tableau、Redshift 业务查询或 SQL，不调用 raw 工具。

只有调用成功，正式指标 ID 为 `dau` 且来源为 `formal_metric_dictionary` 时，报告正式参考连接可用；这不证明仓库查询或 governance 权限已验收。按钮消失、网页 complete、插件启用均不能替代真实调用。

只汇报入口、实际版本、认证状态、验证结果和下一步。不读取凭据文件，不索取或输出密码、token、授权码或完整授权 URL。可在宿主内部将客户端原生链接传给浏览器启动功能，不在聊天、共享日志、截图或错误输出中暴露；临时登录日志限制为 `600` 并在完成后删除。不新增、删除或重装插件/MCP，不修改 Auth0。
