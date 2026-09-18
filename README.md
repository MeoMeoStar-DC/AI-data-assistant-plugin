# AI 数据助手远程 Plugin

本仓库是 `https://github.com/ddddnake/AI-intelligent-analysis-Python` 的确定性分发产物，只包含 Codex 桌面版所需的 Plugin manifest、远程 MCP 配置和两套业务 Skill 和一套连接排障 Skill。

所有文件均由主仓库的 `scripts/generate_ai_data_assistant_plugin_distribution.py` 生成。禁止在本仓库手工修改；修改会在下一次生成时被拒绝或覆盖。

## 安装

本仓库公开可读，使用 HTTPS 安装，不需要 GitHub 账号、PAT 或 SSH key。

```bash
codex plugin marketplace add https://github.com/MeoMeoStar-DC/AI-data-assistant-plugin.git --ref stable
codex plugin add ai-data-assistant@ai-data-assistant-team
```

重启 Codex 桌面版，在新会话中使用“AI 数据助手”。在插件详情或首次使用的连接提示中，通过 Auth0 使用个人身份授权。没有提示时先核对插件加载状态，不要另外添加同名 MCP。

## 更新

```bash
codex plugin marketplace upgrade ai-data-assistant-team
codex plugin add ai-data-assistant@ai-data-assistant-team
```

更新后重启 Codex 桌面版并新建会话。普通成员默认使用 `analyst` 权限；`governance` 仅限经批准人员。

## 安全边界

- MCP 固定连接 `https://mcp.aidamms.com/mcp`。
- 本仓库只包含可公开的预注册 OAuth Client ID 和回调地址；不包含 Client Secret、token、其他凭据、本地 Node 启动脚本、服务器代码、业务日志或原始 metadata。
- `distribution-manifest.json` 记录生成所用的主仓库提交和源文件哈希。
