---
name: agent-sdk-advanced
description: "Use when the user asks about Agent SDK permissions, canUseTool callback, permission modes, hooks system, PreToolUse/PostToolUse, custom subagents, agent definitions, structured output, outputFormat, sandbox configuration, budget controls, maxTurns, file checkpointing, or thinking/effort configuration."
---

# Claude Agent SDK — Advanced Features

This skill covers advanced configuration options for `@anthropic-ai/claude-agent-sdk`. For basic usage (query, streaming, sessions), see the core SDK skill.

---

## 1. Permission Control

### Permission Modes

Set via `options.permissionMode`:

| Mode | Behavior |
|------|----------|
| `'default'` | Standard — prompts user for dangerous operations |
| `'acceptEdits'` | Auto-accept file edit operations |
| `'bypassPermissions'` | Skip all checks (requires `allowDangerouslySkipPermissions: true`) |
| `'plan'` | Planning mode — no actual tool execution |
| `'dontAsk'` | Never prompt — deny anything not pre-approved |

### Allowed / Disallowed Tools

```typescript
options: {
  allowedTools: ['Read', 'Grep', 'Glob'],    // auto-approved, no prompt
  disallowedTools: ['Bash', 'Write'],         // removed from model context entirely
}
```

### canUseTool Callback

Called before every tool execution. Must return `PermissionResult`:

```typescript
const canUseTool: CanUseTool = async (toolName, input, options) => {
  if (['Read', 'Grep', 'Glob'].includes(toolName)) return { behavior: 'allow' };
  if (toolName === 'Write' && (input.file_path as string)?.includes('/protected/')) {
    return { behavior: 'deny', message: 'Protected path' };
  }
  return { behavior: 'ask' };  // fall through to default prompt
};
const result = await query({ prompt: 'Update configs', options: { canUseTool } });
```

**PermissionResult type:**

```typescript
type PermissionResult =
  | { behavior: 'allow'; updatedInput?: Record<string, unknown>; updatedPermissions?: PermissionUpdate[] }
  | { behavior: 'deny'; message?: string }
  | { behavior: 'ask' };
```

- `'allow'` — proceed; optionally modify input or persist permission rules
- `'deny'` — block with an optional message shown to the model
- `'ask'` — fall through to the default permission prompt

The `canUseTool` options parameter also provides `title`, `displayName`, `description`, `blockedPath`, `decisionReason`, `toolUseID`, and `agentID` for context.

See `references/permissions-reference.md` for the complete permissions API.

---

## 2. Hooks System

Register event hooks via `options.hooks`. Each entry is a `HookCallbackMatcher[]`:

```typescript
interface HookCallbackMatcher {
  matcher?: string;        // optional tool name filter (tool events only)
  hooks: HookCallback[];   // callback functions
  timeout?: number;        // seconds
}
type HookCallback = (input: HookInput, toolUseID: string | undefined, options: { signal: AbortSignal }) => Promise<HookJSONOutput>;
```

### Key Hook Events

| Event | When it fires |
|-------|---------------|
| `PreToolUse` | Before tool executes — can approve, block, or modify input |
| `PostToolUse` | After successful execution — can add context or modify output |
| `PostToolUseFailure` | After tool fails |
| `SessionStart` | Session starts/resumes/clears/compacts |
| `SessionEnd` | Session ends |
| `SubagentStart` / `SubagentStop` | Subagent lifecycle |
| `Stop` | Main agent about to stop |
| `PermissionRequest` | Permission prompt about to be shown |
| `UserPromptSubmit` | User submits a prompt |

### PreToolUse Example

```typescript
const filterBash: HookCallback = async (input) => {
  const { tool_name, tool_input } = input as PreToolUseHookInput;
  if ((tool_input as any)?.command?.includes('rm -rf /')) {
    return { hookSpecificOutput: {
      hookEventName: 'PreToolUse', permissionDecision: 'deny',
      permissionDecisionReason: 'Dangerous command blocked',
    }};
  }
  return { hookSpecificOutput: { hookEventName: 'PreToolUse', permissionDecision: 'allow' }};
};

options.hooks = { PreToolUse: [{ matcher: 'Bash', hooks: [filterBash], timeout: 10 }] };
```

### PostToolUse Example

```typescript
const audit: HookCallback = async (input) => {
  const { tool_name, tool_response } = input as PostToolUseHookInput;
  console.log(`[audit] ${tool_name} done`);
  return { hookSpecificOutput: {
    hookEventName: 'PostToolUse',
    additionalContext: 'Run tests after edits.',
  }};
};
options.hooks = { PostToolUse: [{ hooks: [audit] }] };
```

See `references/hooks-reference.md` for all 26 hook events.

---

## 3. Custom Agents / Subagents

Define via `options.agents`. Invoked via the built-in Agent tool.

```typescript
const result = await query({
  prompt: 'Review the auth module',
  options: {
    agents: {
      reviewer: {
        description: 'Reviews code for bugs and style issues',
        prompt: 'You are a senior code reviewer...',
        tools: ['Read', 'Grep', 'Glob'],
        model: 'claude-sonnet-4-6',
        maxTurns: 10,
      },
    },
    agent: 'reviewer',  // use as main thread agent
  },
});
```

**AgentDefinition fields:** `description` (string), `prompt` (string), `tools?` (string[]), `disallowedTools?` (string[]), `model?` (string), `maxTurns?` (number), `mcpServers?`, `skills?` (string[]), `initialPrompt?` (string), `criticalSystemReminder_EXPERIMENTAL?` (string).

---

## 4. Structured Output

```typescript
const result = await query({
  prompt: 'List top 3 issues',
  options: {
    outputFormat: {
      type: 'json_schema',
      schema: {
        type: 'object',
        properties: {
          issues: { type: 'array', items: { type: 'object', properties: {
            severity: { type: 'string' }, file: { type: 'string' }, description: { type: 'string' },
          }, required: ['severity', 'file', 'description'] }},
          summary: { type: 'string' },
        },
        required: ['issues', 'summary'],
      },
    },
  },
});
// Result in SDKResultSuccess.structured_output
```

**Types:**

```typescript
type OutputFormat = JsonSchemaOutputFormat;
type JsonSchemaOutputFormat = { type: 'json_schema'; schema: Record<string, unknown> };
```

The structured data is available on `SDKResultSuccess.structured_output`. If validation fails after retries, the result subtype will be `'error_max_structured_output_retries'`.

---

## 5. Budget Controls

```typescript
options: {
  maxTurns: 15,              // max API round-trips; error subtype: 'error_max_turns'
  maxBudgetUsd: 5.0,         // dollar ceiling; error subtype: 'error_max_budget_usd'
  taskBudget: { total: 100000 }, // alpha: API-side token budget, model paces itself
}
```

---

## 6. Sandbox

```typescript
options: {
  sandbox: {
    enabled: true,
    failIfUnavailable: true,
    autoAllowBashIfSandboxed: true,
    filesystem: {
      allowWrite: ['/tmp', '/workspace/src'],
      denyWrite: ['/workspace/.env'],
      denyRead: ['/etc/shadow'],
    },
    network: {
      allowedDomains: ['registry.npmjs.org'],
      allowLocalBinding: true,
    },
  },
}
```

**SandboxSettings fields:**

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | `boolean?` | Enable sandboxing |
| `failIfUnavailable` | `boolean?` | Error if sandbox cannot be established |
| `autoAllowBashIfSandboxed` | `boolean?` | Auto-approve Bash when sandboxed |
| `allowUnsandboxedCommands` | `boolean?` | Allow commands that escape the sandbox |
| `filesystem` | `object?` | `allowWrite`, `denyWrite`, `allowRead`, `denyRead` (string arrays), `allowManagedReadPathsOnly` |
| `network` | `object?` | `allowedDomains`, `allowLocalBinding`, `allowUnixSockets`, `allowAllUnixSockets`, `allowManagedDomainsOnly` |

---

## 7. Thinking & Effort

```typescript
// Adaptive — Claude decides when/how much to think (Opus 4.6+)
options: { thinking: { type: 'adaptive' } }
// Fixed budget
options: { thinking: { type: 'enabled', budgetTokens: 10000 } }
// Disabled
options: { thinking: { type: 'disabled' } }
```

**Effort level** (works with adaptive thinking):

```typescript
options: { effort: 'low' }  // 'low' | 'medium' | 'high' | 'max'
```

| Effort | Behavior |
|--------|----------|
| `'low'` | Minimal thinking, fastest responses |
| `'medium'` | Moderate thinking |
| `'high'` | Deep reasoning (default) |
| `'max'` | Maximum effort (select models only) |

---

## 8. File Checkpointing

Enable with `enableFileCheckpointing: true`. Rewind files to any previous user message state:

```typescript
const result = await session.rewindFiles(userMessageId, { dryRun: false });
// Returns: { canRewind: boolean; error?: string; filesChanged?: string[]; insertions?: number; deletions?: number }
```

Use `dryRun: true` to preview changes without modifying files.
