# query() Options Reference

All fields on the `Options` type from `@anthropic-ai/claude-agent-sdk`. Every field is optional.

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const conversation = query({
  prompt: "...",
  options: {
    // All fields below are optional
  },
});
```

---

## Core

| Field | Type | Description |
|---|---|---|
| `cwd` | `string` | Working directory for the session. Defaults to `process.cwd()`. |
| `model` | `string` | Claude model ID (e.g. `'claude-sonnet-4-6'`, `'claude-opus-4-6'`). Defaults to CLI default. |
| `fallbackModel` | `string` | Fallback model if the primary is unavailable. |
| `agent` | `string` | Agent name for the main thread. The agent must be defined in `agents` or settings. Equivalent to `--agent` CLI flag. |
| `systemPrompt` | `string \| { type: 'preset'; preset: 'claude_code'; append?: string }` | Custom system prompt, or use the Claude Code default with optional appended instructions. |
| `additionalDirectories` | `string[]` | Extra directories Claude can access beyond cwd. Must be absolute paths. |

```typescript
// System prompt examples
systemPrompt: "You are a security auditor. Only report vulnerabilities."

systemPrompt: {
  type: "preset",
  preset: "claude_code",
  append: "Always explain your reasoning step by step."
}
```

---

## Execution

| Field | Type | Description |
|---|---|---|
| `maxTurns` | `number` | Maximum conversation turns (user + assistant round-trips) before stopping. |
| `maxBudgetUsd` | `number` | Spending cap in USD. Returns `error_max_budget_usd` result if exceeded. |
| `thinking` | `ThinkingConfig` | Controls extended thinking. `{ type: 'adaptive' }` (Claude decides, default on Opus 4.6+), `{ type: 'enabled', budgetTokens: number }` (fixed budget), or `{ type: 'disabled' }`. |
| `effort` | `EffortLevel` | Thinking effort: `'low'`, `'medium'`, `'high'` (default), `'max'` (Opus 4.6 only). |
| `maxThinkingTokens` | `number` | **Deprecated.** Use `thinking` instead. On Opus 4.6, treated as on/off (0 = disabled, nonzero = adaptive). |
| `taskBudget` | `{ total: number }` | API-side task budget in tokens. Alpha feature. |
| `abortController` | `AbortController` | Cancel the query externally. When aborted, the query stops and cleans up. |

```typescript
// ThinkingConfig variants
thinking: { type: "adaptive" }                    // Claude decides (Opus 4.6+ default)
thinking: { type: "enabled", budgetTokens: 8192 } // Fixed budget
thinking: { type: "disabled" }                     // No extended thinking
```

---

## Permissions

| Field | Type | Description |
|---|---|---|
| `permissionMode` | `PermissionMode` | `'default'` — prompts for dangerous ops. `'acceptEdits'` — auto-accept file edits. `'bypassPermissions'` — skip all checks (requires `allowDangerouslySkipPermissions`). `'plan'` — no tool execution. `'dontAsk'` — deny if not pre-approved. |
| `allowDangerouslySkipPermissions` | `boolean` | Must be `true` when using `permissionMode: 'bypassPermissions'`. Safety gate. |
| `canUseTool` | `CanUseTool` | Custom permission callback. Called before each tool execution with `(toolName, input, options)`. Return `{ behavior: 'allow' }` or `{ behavior: 'deny', message: '...' }`. |
| `allowedTools` | `string[]` | Tool names that are auto-allowed without prompting. |
| `disallowedTools` | `string[]` | Tool names removed from the model's context entirely. |
| `permissionPromptToolName` | `string` | Route permission requests through a named MCP tool instead of the default handler. |

```typescript
// Custom permission handler
canUseTool: async (toolName, input, { title, signal }) => {
  if (toolName === "Bash" && String(input.command).includes("rm")) {
    return { behavior: "deny", message: "Destructive commands not allowed" };
  }
  return { behavior: "allow" };
}
```

---

## Tools & MCP

| Field | Type | Description |
|---|---|---|
| `tools` | `string[] \| { type: 'preset'; preset: 'claude_code' }` | Built-in tool set. Array of names (e.g. `['Bash', 'Read', 'Edit']`), empty array to disable all, or preset for all defaults. |
| `mcpServers` | `Record<string, McpServerConfig>` | MCP server configurations. Keys are server names, values define transport (stdio, http, sse, or SDK). |
| `agents` | `Record<string, AgentDefinition>` | Custom subagent definitions. Each has `description`, `prompt`, optional `tools`, `disallowedTools`, `model`, `mcpServers`, `skills`, `maxTurns`. |
| `toolConfig` | `ToolConfig` | Per-tool configuration for built-in tools, e.g. `{ askUserQuestion: { previewFormat: 'html' } }`. |
| `strictMcpConfig` | `boolean` | When `true`, invalid MCP configs cause errors instead of warnings. |

```typescript
// MCP server configuration example
mcpServers: {
  "my-server": {
    command: "node",
    args: ["./my-mcp-server.js"],
    env: { API_KEY: process.env.API_KEY }
  }
}

// Subagent definition example
agents: {
  "test-runner": {
    description: "Runs tests and reports results",
    prompt: "You are a test runner. Run the test suite and report failures.",
    tools: ["Bash", "Read", "Grep", "Glob"],
    maxTurns: 5
  }
}
```

---

## Session

| Field | Type | Description |
|---|---|---|
| `resume` | `string` | Session ID to resume. Loads conversation history. Mutually exclusive with `continue`. |
| `continue` | `boolean` | Continue the most recent conversation in the cwd. Mutually exclusive with `resume`. |
| `sessionId` | `string` | Use a specific UUID for the session instead of auto-generating one. |
| `forkSession` | `boolean` | When resuming, fork to a new session ID instead of continuing the original. |
| `resumeSessionAt` | `string` | When resuming, only load messages up to this UUID (from `SDKAssistantMessage.uuid`). |
| `enableFileCheckpointing` | `boolean` | Track file changes so they can be rewound via `Query.rewindFiles()`. |
| `persistSession` | `boolean` | When `false`, sessions are not saved to disk. Defaults to `true`. |

---

## Output

| Field | Type | Description |
|---|---|---|
| `outputFormat` | `OutputFormat` | Structured output config: `{ type: 'json_schema', schema: { ... } }`. The agent returns data matching the schema. |
| `includePartialMessages` | `boolean` | Emit `SDKPartialAssistantMessage` (type `'stream_event'`) during streaming. |
| `promptSuggestions` | `boolean` | Emit a `prompt_suggestion` message after each turn with a predicted next prompt. |
| `agentProgressSummaries` | `boolean` | Emit AI-generated progress summaries on `task_progress` events for running subagents (~30s interval). |

---

## Settings

| Field | Type | Description |
|---|---|---|
| `settings` | `string \| Settings` | Path to a settings JSON file or inline settings object. Loaded as "flag settings" (highest user-controlled priority). |
| `settingSources` | `SettingSource[]` | Which filesystem settings to load: `'user'`, `'project'`, `'local'`. Empty = SDK isolation mode. Must include `'project'` to load CLAUDE.md files. |
| `env` | `Record<string, string \| undefined>` | Environment variables for the Claude Code process. Set `CLAUDE_AGENT_SDK_CLIENT_APP` for User-Agent identification. |
| `sandbox` | `SandboxSettings` | Sandbox settings for command execution isolation. Controls `enabled`, `autoAllowBashIfSandboxed`, network, and filesystem restrictions. |
| `plugins` | `SdkPluginConfig[]` | Plugins to load. Currently only `{ type: 'local', path: '...' }`. |
| `betas` | `SdkBeta[]` | Enable beta features, e.g. `'context-1m-2025-08-07'` for 1M context (Sonnet 4/4.5 only). |

---

## Hooks

| Field | Type | Description |
|---|---|---|
| `hooks` | `Partial<Record<HookEvent, HookCallbackMatcher[]>>` | Hook callbacks keyed by event name. Each matcher has optional `matcher` pattern, `hooks` array of async callbacks, and optional `timeout` in seconds. |
| `onElicitation` | `OnElicitation` | Callback for MCP elicitation requests not handled by hooks. Return `{ action: 'accept', content: {...} }` or `{ action: 'decline' }`. |

Supported hook events: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Notification`, `UserPromptSubmit`, `SessionStart`, `SessionEnd`, `Stop`, `StopFailure`, `SubagentStart`, `SubagentStop`, `PreCompact`, `PostCompact`, `PermissionRequest`, `Setup`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `Elicitation`, `ElicitationResult`, `ConfigChange`, `WorktreeCreate`, `WorktreeRemove`, `InstructionsLoaded`, `CwdChanged`, `FileChanged`.

---

## Debug

| Field | Type | Description |
|---|---|---|
| `debug` | `boolean` | Enable verbose debug logging (equivalent to `--debug` CLI flag). |
| `debugFile` | `string` | Write debug logs to this file path. Implicitly enables debug mode. |
| `stderr` | `(data: string) => void` | Callback for stderr output from the Claude Code process. |

---

## Infrastructure

| Field | Type | Description |
|---|---|---|
| `executable` | `'bun' \| 'deno' \| 'node'` | JavaScript runtime to use. Auto-detected if omitted. |
| `executableArgs` | `string[]` | Additional arguments for the JS runtime executable. |
| `pathToClaudeCodeExecutable` | `string` | Path to the Claude Code executable. Uses built-in if omitted. |
| `spawnClaudeCodeProcess` | `(options: SpawnOptions) => SpawnedProcess` | Custom spawn function for running Claude Code in VMs, containers, or remote environments. |
| `extraArgs` | `Record<string, string \| null>` | Additional CLI arguments. Keys are arg names without `--`, values are arg values. Use `null` for boolean flags. |

```typescript
// Custom spawn for running in a Docker container
spawnClaudeCodeProcess: (options) => {
  // options: { command, args, cwd, env, signal }
  return spawn("docker", ["exec", "-i", "my-container", options.command, ...options.args], {
    cwd: options.cwd,
    env: options.env,
    signal: options.signal,
  });
}
```
