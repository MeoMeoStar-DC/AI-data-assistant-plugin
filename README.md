# AI 数据助手远程 Plugin

本仓库是 `https://github.com/MeoMeoStar-DC/DCP-Data-Assident` 的确定性分发产物，包含 Codex 桌面版和 Codex CLI 所需的 Plugin manifest、远程 MCP 配置、两套业务 Skill、一套连接排障 Skill 和自主分析的三份专题参考。

所有文件均由主仓库的 `scripts/generate_ai_data_assistant_plugin_distribution.py` 生成。禁止在本仓库手工修改；修改会在下一次生成时被拒绝或覆盖。

## 安装

本仓库公开可读，使用 HTTPS 安装，不需要 GitHub 账号、PAT 或 SSH key。

```bash
codex plugin marketplace add https://github.com/MeoMeoStar-DC/AI-data-assistant-plugin.git --ref stable
codex plugin add ai-data-assistant@ai-data-assistant-team
```

重启 Codex 桌面版，或结束 Codex CLI 旧会话并新开会话，在新会话中使用“AI 数据助手”。桌面版在插件详情的 MCP 设置中进行身份验证；Codex CLI 先确认 Plugin 已提供 `ai-data-assistant`，再运行：

```bash
codex mcp login ai-data-assistant
```

在本机浏览器完成该次 CLI 原生 OAuth。CLI 与浏览器不在同一台机器时，先在浏览器所在电脑建立仅监听回环地址的 SSH 转发：

```text
ssh -i "<SSH私钥路径>" -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -N -L 127.0.0.1:14550:127.0.0.1:14550 <远程用户名>@<远程主机地址>
```

保持登录进程和转发运行，先用不带授权参数的 `http://127.0.0.1:14550/` 做实际 HTTP 连通性检查；超时或拒绝连接时先修复转发。不要手工修改授权 URL、resource、回调、PKCE 或 Client Secret，也不要因为 Plugin 服务器未显示就运行 `codex mcp add` 新增同名 MCP。登录命令提示 `Successfully logged in to MCP server 'ai-data-assistant'.` 后，再检查 `codex mcp list` 显示 `OAuth`，最后关闭临时转发并删除包含授权链接的临时日志。

## 更新

```bash
codex plugin marketplace upgrade ai-data-assistant-team
codex plugin add ai-data-assistant@ai-data-assistant-team
```

更新后重启 Codex 桌面版，或结束 CLI 旧会话并新建会话。普通成员默认使用 `analyst` 权限；`governance` 仅限经批准人员。

## 安全边界

- MCP 固定连接 `https://mcp.aidamms.com/mcp`。
- 本仓库只包含可公开的预注册 OAuth Client ID 和回调地址；不包含 Client Secret、token、其他凭据、本地 Node 启动脚本、服务器代码、业务日志或原始 metadata。
- `distribution-manifest.json` 记录生成所用的主仓库提交和源文件哈希。
