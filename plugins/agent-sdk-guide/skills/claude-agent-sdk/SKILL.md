---
name: claude-agent-sdk
description: "Use when the user asks about the Claude Agent SDK, mentions @anthropic-ai/claude-agent-sdk, wants to build a programmatic agent with Claude Code, asks about query(), discusses embedding Claude Code in an application, or wants to understand how to use the Agent SDK. This is the entry point for Agent SDK development."
---

# Claude Agent SDK

## What is the Agent SDK?

The Claude Agent SDK (`@anthropic-ai/claude-agent-sdk`) wraps Claude Code as a managed subprocess and exposes it through a single `query()` function that returns a `Query` object — an `AsyncGenerator<SDKMessage, void>` of typed messages. You iterate with `for await` to receive assistant responses, tool use events, status updates, and a final result. The SDK requires Node.js >= 18, depends on `@anthropic-ai/sdk` (>= 0.74.0) and `@modelcontextprotocol/sdk` (>= 1.27.1), and has a peer dependency on `zod ^4.0.0`.

---

## Quick Start

Install the package:

```bash
npm install @anthropic-ai/claude-agent-sdk
```

### Single-shot example

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const conversation = query({
  prompt: "Explain what package.json does in three sentences.",
  options: {
    cwd: process.cwd(),
    permissionMode: "plan",       // read-only, no tool execution
    maxTurns: 1,
  },
});

for await (const message of conversation) {
  if (message.type === "assistant") {
    // message.message is a BetaMessage from @anthropic-ai/sdk
    for (const block of message.message.content) {
      if (block.type === "text") {
        console.log(block.text);
      }
    }
  }
}
```

### Checking the result

```typescript
import { query, type SDKMessage } from "@anthropic-ai/claude-agent-sdk";

const conversation = query({
  prompt: "List the files in the current directory.",
  options: {
    cwd: "/tmp/my-project",
    permissionMode: "bypassPermissions",
    allowDangerouslySkipPermissions: true,
    maxTurns: 3,
  },
});

let resultText = "";

for await (const message of conversation) {
  if (message.type === "result") {
    if (message.subtype === "success") {
      resultText = message.result;
      console.log("Cost:", message.total_cost_usd, "USD");
      console.log("Turns:", message.num_turns);
    } else {
      console.error("Error subtype:", message.subtype);
      console.error("Errors:", message.errors);
    }
  }
}

console.log("Final result:", resultText);
```

---

## Decision tree — specialized skills

Depending on what you need, load the appropriate specialized skill:

| Need | Skill to load |
|---|---|
| Custom tools (MCP servers, in-process tools via `createSdkMcpServer`) | `agent-sdk-custom-tools` |
| Session management, multi-turn conversations, resume/fork | `agent-sdk-sessions` |
| Permissions, hooks, subagents, sandbox configuration | `agent-sdk-advanced` |

---

## Core `query()` pattern

The `query()` function is the primary entry point. It accepts a prompt (string or async iterable of `SDKUserMessage`) and an `Options` object:

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const conversation = query({
  prompt: "Refactor the auth module to use JWT.",
  options: {
    cwd: "/path/to/project",
    model: "claude-sonnet-4-6",
    permissionMode: "acceptEdits",
    maxTurns: 10,
    maxBudgetUsd: 0.50,
    thinking: { type: "adaptive" },
    effort: "high",
  },
});
```

The returned `Query` object is an `AsyncGenerator<SDKMessage, void>` with additional control methods: `interrupt()`, `setModel()`, `setPermissionMode()`, `close()`, `rewindFiles()`, `setMcpServers()`, and more.

### Key Options fields

| Field | Type | Purpose |
|---|---|---|
| `cwd` | `string` | Working directory (defaults to `process.cwd()`) |
| `model` | `string` | Model ID, e.g. `'claude-sonnet-4-6'`, `'claude-opus-4-6'` |
| `permissionMode` | `PermissionMode` | `'default'`, `'acceptEdits'`, `'bypassPermissions'`, `'plan'`, `'dontAsk'` |
| `maxTurns` | `number` | Max conversation turns before stopping |
| `maxBudgetUsd` | `number` | Spending cap in USD |
| `thinking` | `ThinkingConfig` | `{ type: 'adaptive' }`, `{ type: 'enabled', budgetTokens: N }`, or `{ type: 'disabled' }` |
| `effort` | `EffortLevel` | `'low'`, `'medium'`, `'high'`, `'max'` |
| `tools` | `string[] \| preset` | Built-in tool set; empty array disables all |
| `allowedTools` | `string[]` | Auto-approved tool names |
| `mcpServers` | `Record<string, McpServerConfig>` | MCP server configurations |
| `includePartialMessages` | `boolean` | Emit streaming partial messages |
| `systemPrompt` | `string \| preset` | Custom or preset system prompt |

Full reference: `references/query-options.md`

---

## Message types overview

Every message yielded by the `Query` generator has a `type` field for discrimination. The most important subtypes:

| `type` value | TypeScript type | Description |
|---|---|---|
| `'system'` (subtype `'init'`) | `SDKSystemMessage` | First message; contains session info, tools, model, cwd |
| `'assistant'` | `SDKAssistantMessage` | Model response with `BetaMessage` content blocks |
| `'user'` | `SDKUserMessage` | User message (echoed back or replayed on resume) |
| `'result'` | `SDKResultMessage` | Final result — either `SDKResultSuccess` or `SDKResultError` |
| `'system'` (subtype `'status'`) | `SDKStatusMessage` | Status change (e.g. `'compacting'`) |
| `'stream_event'` | `SDKPartialAssistantMessage` | Streaming token events (requires `includePartialMessages`) |
| `'system'` (subtype `'task_started'`) | `SDKTaskStartedMessage` | Subagent task launched |
| `'system'` (subtype `'task_progress'`) | `SDKTaskProgressMessage` | Subagent progress update |
| `'system'` (subtype `'task_notification'`) | `SDKTaskNotificationMessage` | Subagent task completed/failed/stopped |
| `'prompt_suggestion'` | `SDKPromptSuggestionMessage` | Predicted next user prompt (after result) |
| `'rate_limit_event'` | `SDKRateLimitEvent` | Rate limit status change |
| `'tool_progress'` | `SDKToolProgressMessage` | Long-running tool heartbeat |

Full reference: `references/sdk-message-types.md`

---

## Common patterns

### 1. Single-shot prompt with result checking

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

async function ask(prompt: string): Promise<string> {
  const conversation = query({
    prompt,
    options: {
      cwd: process.cwd(),
      permissionMode: "plan",
      maxTurns: 1,
    },
  });

  for await (const message of conversation) {
    if (message.type === "result" && message.subtype === "success") {
      return message.result;
    }
    if (message.type === "result" && message.subtype !== "success") {
      throw new Error(`Query failed: ${message.subtype}`);
    }
  }

  throw new Error("No result received");
}
```

### 2. Streaming with partial messages

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const conversation = query({
  prompt: "Write a haiku about TypeScript.",
  options: {
    includePartialMessages: true,
    permissionMode: "plan",
    maxTurns: 1,
  },
});

for await (const message of conversation) {
  if (message.type === "stream_event") {
    const event = message.event;
    if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
      process.stdout.write(event.delta.text);
    }
  }
  if (message.type === "result") {
    console.log("\n--- done ---");
    console.log("Cost:", message.total_cost_usd, "USD");
  }
}
```

### 3. Setting model, thinking mode, and budget

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const conversation = query({
  prompt: "Audit this project for security vulnerabilities.",
  options: {
    cwd: "/path/to/project",
    model: "claude-opus-4-6",
    thinking: { type: "adaptive" },
    effort: "max",
    maxTurns: 20,
    maxBudgetUsd: 2.00,
    permissionMode: "plan",
  },
});

for await (const message of conversation) {
  if (message.type === "assistant") {
    for (const block of message.message.content) {
      if (block.type === "text") {
        console.log(block.text);
      }
    }
  }
  if (message.type === "result") {
    console.log(`Completed in ${message.num_turns} turns, $${message.total_cost_usd}`);
  }
}
```

---

## Reference files

- `references/query-options.md` — All `Options` fields grouped by category
- `references/sdk-message-types.md` — All `SDKMessage` subtypes with discriminant fields
