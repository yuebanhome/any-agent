# Permissions Reference — Claude Agent SDK

Complete reference for the permission system in `@anthropic-ai/claude-agent-sdk`.

---

## PermissionMode

Controls the overall permission behavior for a session.

```typescript
type PermissionMode = 'default' | 'acceptEdits' | 'bypassPermissions' | 'plan' | 'dontAsk';
```

| Mode | Description |
|------|-------------|
| `'default'` | Standard behavior. Prompts the user for dangerous operations (file writes, shell commands). Safe reads are auto-allowed. |
| `'acceptEdits'` | Auto-accept file edit operations (Write, Edit). Other dangerous operations still prompt. |
| `'bypassPermissions'` | Bypass all permission checks. **Requires** `allowDangerouslySkipPermissions: true` as a safety gate. |
| `'plan'` | Planning mode. No tools are actually executed. The model can reason about what it would do. |
| `'dontAsk'` | Never show a permission prompt. If a tool is not pre-approved (via `allowedTools` or rules), it is denied. |

```typescript
const result = await query({
  prompt: 'Fix the tests',
  options: {
    permissionMode: 'bypassPermissions',
    allowDangerouslySkipPermissions: true,  // required for bypassPermissions
  },
});
```

---

## allowedTools / disallowedTools

### allowedTools

An array of tool names that are auto-allowed without prompting. These tools execute immediately without user approval.

```typescript
options: {
  allowedTools: ['Read', 'Grep', 'Glob', 'Bash'],
}
```

### disallowedTools

An array of tool names that are completely removed from the model's context. The model cannot use these tools at all, even if they would otherwise be allowed.

```typescript
options: {
  disallowedTools: ['Write', 'Edit'],  // model cannot write files
}
```

These two options work independently:
- `allowedTools` controls which tools skip the permission prompt
- `disallowedTools` controls which tools are unavailable entirely

---

## canUseTool Callback

The primary mechanism for dynamic, programmatic permission control. Called before every tool execution.

### Full Signature

```typescript
type CanUseTool = (
  toolName: string,
  input: Record<string, unknown>,
  options: {
    signal: AbortSignal;
    suggestions?: PermissionUpdate[];
    blockedPath?: string;
    decisionReason?: string;
    title?: string;
    displayName?: string;
    description?: string;
    toolUseID: string;
    agentID?: string;
  }
) => Promise<PermissionResult>;
```

Key options fields: `signal` (AbortSignal), `suggestions` (PermissionUpdate[]), `blockedPath`, `decisionReason`, `title` (full prompt sentence), `displayName` (short label), `description`, `toolUseID`, `agentID`.

---

## PermissionResult

```typescript
type PermissionResult =
  | { behavior: 'allow'; updatedInput?: Record<string, unknown>; updatedPermissions?: PermissionUpdate[]; decisionClassification?: PermissionDecisionClassification }
  | { behavior: 'deny'; message?: string; decisionClassification?: PermissionDecisionClassification }
  | { behavior: 'ask' };
```

- `'allow'` — proceed; `updatedInput` modifies tool args, `updatedPermissions` persists rules
- `'deny'` — block; `message` is shown to the model
- `'ask'` — fall through to the default permission prompt

---

## PermissionUpdate

Used in `updatedPermissions` to persist permission rules across the session.

```typescript
type PermissionUpdate =
  | { type: 'addRules'; rules: PermissionRuleValue[]; behavior: PermissionBehavior; destination: PermissionUpdateDestination }
  | { type: 'replaceRules'; rules: PermissionRuleValue[]; behavior: PermissionBehavior; destination: PermissionUpdateDestination }
  | { type: 'removeRules'; rules: PermissionRuleValue[]; behavior: PermissionBehavior; destination: PermissionUpdateDestination }
  | { type: 'setMode'; mode: PermissionMode; destination: PermissionUpdateDestination }
  | { type: 'addDirectories'; /* ... */ };

type PermissionRuleValue = {
  toolName: string;
  ruleContent?: string;  // e.g., glob pattern for file paths
};

type PermissionBehavior = 'allow' | 'deny' | 'ask';

type PermissionUpdateDestination = 'userSettings' | 'projectSettings' | 'localSettings' | 'session' | 'cliArg';
```

---

## Permission Decision Flow

The SDK evaluates permissions in this order:

1. **disallowedTools** — if the tool is in this list, it is removed from context entirely (never reaches permission checks)
2. **Hooks (PreToolUse)** — PreToolUse hooks with `permissionDecision` of `'allow'` or `'deny'` take priority
3. **allowedTools** — if the tool is in this list, it is auto-allowed without prompting
4. **permissionMode** — the mode determines default behavior:
   - `'bypassPermissions'` — allow everything
   - `'dontAsk'` — deny if not already approved
   - `'plan'` — deny execution (planning only)
   - `'acceptEdits'` — allow edit tools, prompt for others
   - `'default'` — prompt for dangerous operations
5. **canUseTool callback** — if provided, called for tools that reach this stage
6. **Default prompt** — if `canUseTool` returns `{ behavior: 'ask' }` or is not provided, the built-in permission prompt is shown

---

## Integration with Hooks

The `PermissionRequest` hook event fires when a permission prompt is about to be shown. This is separate from `canUseTool` and fires later in the flow.

```typescript
options: {
  hooks: {
    PermissionRequest: [{
      hooks: [async (input) => {
        const hookInput = input as PermissionRequestHookInput;
        // Auto-approve specific tools
        if (hookInput.tool_name === 'Bash') {
          return {
            hookSpecificOutput: {
              hookEventName: 'PermissionRequest',
              permissionDecision: 'allow',
            },
          };
        }
        return {};
      }],
    }],
  },
}
```

---

## Common Patterns

**Read-only mode:** allow reads, deny writes:

```typescript
const canUseTool: CanUseTool = async (toolName) => {
  if (['Read', 'Grep', 'Glob'].includes(toolName)) return { behavior: 'allow' };
  if (['Write', 'Edit', 'Bash'].includes(toolName)) return { behavior: 'deny', message: 'Writes disabled' };
  return { behavior: 'ask' };
};
```

**Path-based approval:**

```typescript
const canUseTool: CanUseTool = async (toolName, input) => {
  if (toolName === 'Write' && !(input.file_path as string)?.startsWith('/workspace/src/'))
    return { behavior: 'deny', message: 'Writes only allowed in /workspace/src/' };
  return { behavior: 'ask' };
};
```

**CI bypass:** `{ permissionMode: 'bypassPermissions', allowDangerouslySkipPermissions: true }`
