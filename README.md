# agent-sdk-guide

A Claude Code plugin providing model-invoked skills for understanding and building with `@anthropic-ai/claude-agent-sdk`.

## Skills

| Skill | Description |
|---|---|
| **claude-agent-sdk** | Entry point — covers `query()`, message types, common patterns, and routes to specialized skills |
| **agent-sdk-custom-tools** | Adding custom tools via `createSdkMcpServer()` + `tool()`, external MCP server configs |
| **agent-sdk-sessions** | Session management, multi-turn conversations, V2 API, query control methods |
| **agent-sdk-advanced** | Permissions, hooks, subagents, structured output, sandbox, budget controls, thinking/effort |

## Installation

Add this plugin to Claude Code:

```bash
# From the Claude Code CLI
/plugin /path/to/this/repo
```

Or add it to your project's `.claude/settings.json`:

```json
{
  "plugins": ["/path/to/this/repo"]
}
```

## How it works

These are **model-invoked skills** — Claude automatically activates them when your conversation matches their trigger conditions. For example, asking "how do I use the Claude Agent SDK?" will load the main `claude-agent-sdk` skill, which then routes to specialized skills as needed.

## Related

- [Official Agent SDK docs](https://docs.claude.com/en/api/agent-sdk/overview)
- [agent-sdk-dev plugin](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/agent-sdk-dev) — scaffolding commands and verification agents (complements this plugin)
