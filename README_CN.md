# any-agent

Claude Code 插件市场 — 一系列实用技能，提升你的 Claude Code 使用体验。

[English](README.md)

## 插件

### agent-sdk-guide

用于理解和构建 `@anthropic-ai/claude-agent-sdk` 应用的技能集。

| 技能 | 说明 |
|---|---|
| **claude-agent-sdk** | 入口技能 — 涵盖 `query()`、消息类型、常见模式，并路由到专项技能 |
| **agent-sdk-custom-tools** | 通过 `createSdkMcpServer()` + `tool()` 添加自定义工具、外部 MCP 服务器配置 |
| **agent-sdk-sessions** | 会话管理、多轮对话、V2 API、查询控制方法 |
| **agent-sdk-advanced** | 权限、hooks、子代理、结构化输出、沙盒、预算控制、思考/effort |

### session-harvest

从 Claude Code 会话中捕获知识，将其转化为可复用的技能。

| 技能 | 说明 |
|---|---|
| **session-harvest** | 回顾对话历史，提取有价值的工作流和模式，调用 skill-creator 生成正式技能 |

### claude-plugin-marketplace-setup

创建和组织 Claude Code 插件市场的指南。

| 技能 | 说明 |
|---|---|
| **claude-plugin-marketplace-setup** | 涵盖目录结构、marketplace.json/plugin.json 模式、多插件组织、source 类型和常见问题排查 |

## 安装

在 Claude Code 中添加此插件市场：

```bash
/plugin marketplace add https://github.com/yuebanhome/any-agent.git
```


## 工作原理

这些是**模型自动调用的技能** — 当你的对话匹配触发条件时，Claude 会自动激活相应技能，大多数技能无需手动命令。例如：

- 问"如何使用 Claude Agent SDK？"会自动激活 `claude-agent-sdk` 技能
- 在会话结束时运行 `/session-harvest` 可以捕获你本次学到的知识

## 相关资源

- [Agent SDK 官方文档](https://docs.claude.com/en/api/agent-sdk/overview)
- [agent-sdk-dev 插件](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/agent-sdk-dev) — 脚手架命令和验证代理
