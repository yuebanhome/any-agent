# Hooks Reference — Claude Agent SDK

## Hook Registration

```typescript
options: {
  hooks: {
    PreToolUse: [{ matcher: 'Bash', hooks: [myCallback], timeout: 10 }],
    PostToolUse: [{ hooks: [anotherCallback] }],  // no matcher = all tools
  }
}
```

### HookCallbackMatcher

```typescript
interface HookCallbackMatcher {
  matcher?: string;        // Tool name filter (tool events only)
  hooks: HookCallback[];
  timeout?: number;        // Seconds before cancellation via AbortSignal
}
```

### HookCallback & Output

```typescript
type HookCallback = (input: HookInput, toolUseID: string | undefined, options: { signal: AbortSignal }) => Promise<HookJSONOutput>;

type SyncHookJSONOutput = {
  continue?: boolean;       // false = stop execution
  suppressOutput?: boolean;
  stopReason?: string;
  decision?: 'approve' | 'block';
  systemMessage?: string;
  reason?: string;
  hookSpecificOutput?: /* event-specific union */;
};

type AsyncHookJSONOutput = { async: true; asyncTimeout?: number };
```

### BaseHookInput (all events extend this)

```typescript
type BaseHookInput = {
  session_id: string; transcript_path: string; cwd: string;
  permission_mode?: string; agent_id?: string; agent_type?: string;
};
```

---

## Tool Events

### PreToolUse

Before tool execution. Can approve/block/modify input.

```typescript
Input:  { hook_event_name: 'PreToolUse'; tool_name: string; tool_input: unknown; tool_use_id: string }
Output: { hookEventName: 'PreToolUse'; permissionDecision?: 'allow'|'deny'|'ask'; permissionDecisionReason?: string; updatedInput?: Record<string,unknown>; additionalContext?: string }
```

### PostToolUse

After successful tool execution. Can add context or modify MCP output.

```typescript
Input:  { hook_event_name: 'PostToolUse'; tool_name: string; tool_input: unknown; tool_response: unknown; tool_use_id: string }
Output: { hookEventName: 'PostToolUse'; additionalContext?: string; updatedMCPToolOutput?: unknown }
```

### PostToolUseFailure

After a tool execution fails.

```typescript
Input:  { hook_event_name: 'PostToolUseFailure'; tool_name: string; tool_input: unknown; tool_use_id: string; error: string; is_interrupt?: boolean }
Output: { hookEventName: 'PostToolUseFailure'; additionalContext?: string }
```

---

## Session Events

### SessionStart

Fires on startup, resume, clear, or compact.

```typescript
Input:  { hook_event_name: 'SessionStart'; source: 'startup'|'resume'|'clear'|'compact'; agent_type?: string; model?: string }
Output: { hookEventName: 'SessionStart'; additionalContext?: string; initialUserMessage?: string }
```

### SessionEnd

```typescript
Input: { hook_event_name: 'SessionEnd'; reason: ExitReason }
```

No hook-specific output.

### Stop

Main agent about to stop.

```typescript
Input: { hook_event_name: 'Stop'; stop_hook_active: boolean; last_assistant_message?: string }
```

### StopFailure

Agent stops due to error.

```typescript
Input: { hook_event_name: 'StopFailure'; error: SDKAssistantMessageError; error_details?: string; last_assistant_message?: string }
```

---

## Subagent Events

### SubagentStart

```typescript
Input:  { hook_event_name: 'SubagentStart'; agent_id: string; agent_type: string }
Output: { hookEventName: 'SubagentStart'; additionalContext?: string }
```

### SubagentStop

```typescript
Input: { hook_event_name: 'SubagentStop'; stop_hook_active: boolean; agent_id: string; agent_transcript_path: string; agent_type: string; last_assistant_message?: string }
```

---

## User Events

### UserPromptSubmit

```typescript
Input:  { hook_event_name: 'UserPromptSubmit'; prompt: string }
Output: { hookEventName: 'UserPromptSubmit'; additionalContext?: string }
```

### Notification

```typescript
Input:  { hook_event_name: 'Notification'; message: string; title?: string; notification_type: string }
Output: { hookEventName: 'Notification'; suppressNotification?: boolean }
```

---

## Permission Events

### PermissionRequest

Fires when a permission prompt is about to be shown.

```typescript
Input:  { hook_event_name: 'PermissionRequest'; tool_name: string; tool_input: unknown; permission_suggestions?: PermissionUpdate[] }
Output: { hookEventName: 'PermissionRequest'; permissionDecision?: PermissionBehavior; updatedPermissions?: PermissionUpdate[] }
```

---

## Task Events

### TaskCreated

```typescript
Input: { hook_event_name: 'TaskCreated'; task_id: string; task_subject: string; task_description?: string; teammate_name?: string; team_name?: string }
```

### TaskCompleted

```typescript
Input: { hook_event_name: 'TaskCompleted'; task_id: string; task_subject: string; task_description?: string; teammate_name?: string; team_name?: string }
```

### TeammateIdle

```typescript
Input: { hook_event_name: 'TeammateIdle'; teammate_name: string; team_name: string }
```

---

## Compaction Events

### PreCompact

```typescript
Input: { hook_event_name: 'PreCompact'; trigger: 'manual'|'auto'; custom_instructions: string|null }
```

### PostCompact

```typescript
Input: { hook_event_name: 'PostCompact'; trigger: 'manual'|'auto'; compact_summary: string }
```

---

## MCP / Elicitation Events

### Elicitation

MCP server requests user input. Hook can auto-respond.

```typescript
Input:  { hook_event_name: 'Elicitation'; mcp_server_name: string; message: string; mode?: 'form'|'url'; url?: string; elicitation_id?: string; requested_schema?: Record<string,unknown> }
Output: { hookEventName: 'Elicitation'; action?: 'accept'|'decline'|'cancel'; content?: Record<string,unknown> }
```

### ElicitationResult

After user responds to an MCP elicitation.

```typescript
Input: { hook_event_name: 'ElicitationResult'; mcp_server_name: string; elicitation_id?: string; mode?: 'form'|'url'; action: 'accept'|'decline'|'cancel' }
```

---

## Configuration / Lifecycle Events

### Setup

```typescript
Input:  { hook_event_name: 'Setup'; trigger: 'init'|'maintenance' }
Output: { hookEventName: 'Setup'; additionalContext?: string }
```

### ConfigChange

```typescript
Input: { hook_event_name: 'ConfigChange'; source: 'user_settings'|'project_settings'|'local_settings'|'policy_settings'|'skills'; file_path?: string }
```

### InstructionsLoaded

```typescript
Input: { hook_event_name: 'InstructionsLoaded'; file_path: string; memory_type: 'User'|'Project'|'Local'|'Managed'; load_reason: 'session_start'|'nested_traversal'|'path_glob_match'|'include'|'compact'; globs?: string[]; trigger_file_path?: string; parent_file_path?: string }
```

---

## File / Worktree Events

### FileChanged

```typescript
Input: { hook_event_name: 'FileChanged'; file_path: string; event: 'change'|'add'|'unlink' }
```

### CwdChanged

```typescript
Input:  { hook_event_name: 'CwdChanged'; old_cwd: string; new_cwd: string }
Output: { hookEventName: 'CwdChanged'; watchPaths?: string[] }
```

### WorktreeCreate

```typescript
Input:  { hook_event_name: 'WorktreeCreate'; name: string }
Output: { hookEventName: 'WorktreeCreate'; worktreePath: string }
```

### WorktreeRemove

```typescript
Input: { hook_event_name: 'WorktreeRemove'; worktree_path: string }
```
