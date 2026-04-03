# any-agent

A Claude Code plugin marketplace — a collection of practical skills to enhance your Claude Code experience.

[中文文档](README_CN.md)

## Plugins

### agent-sdk-guide

Skills for understanding and building with `@anthropic-ai/claude-agent-sdk`.

| Skill | Description |
|---|---|
| **claude-agent-sdk** | Entry point — covers `query()`, message types, common patterns, and routes to specialized skills |
| **agent-sdk-custom-tools** | Adding custom tools via `createSdkMcpServer()` + `tool()`, external MCP server configs |
| **agent-sdk-sessions** | Session management, multi-turn conversations, V2 API, query control methods |
| **agent-sdk-advanced** | Permissions, hooks, subagents, structured output, sandbox, budget controls, thinking/effort |

### session-harvest

Capture knowledge from your Claude Code sessions and turn them into reusable skills.

| Skill | Description |
|---|---|
| **session-harvest** | Review conversation history, extract valuable workflows and patterns, and invoke skill-creator to generate formal skills |

### claude-plugin-marketplace-setup

Guide for creating and organizing Claude Code plugin marketplaces.

| Skill | Description |
|---|---|
| **claude-plugin-marketplace-setup** | Covers directory structure, marketplace.json/plugin.json schema, multi-plugin organization, source types, and troubleshooting |

## Installation

Add this marketplace to Claude Code:

```bash
/plugin marketplace add https://github.com/yuebanhome/any-agent.git
```


## How it works

These are **model-invoked skills** — Claude automatically activates them when your conversation matches their trigger conditions. No manual commands needed for most skills. For example:

- Asking "how do I use the Claude Agent SDK?" activates the `claude-agent-sdk` skill
- Running `/session-harvest` at the end of a session captures what you learned

## Related

- [Official Agent SDK docs](https://docs.claude.com/en/api/agent-sdk/overview)
- [agent-sdk-dev plugin](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/agent-sdk-dev) — scaffolding commands and verification agents

## Friendly Links

- [linux.do Community Help](https://linux.do/) — Community resources and support

## Built for Claude Code

[![Claude Code](https://img.shields.io/badge/Claude-Code-00A4EF?logo=anthropic)](https://claude.ai/)
