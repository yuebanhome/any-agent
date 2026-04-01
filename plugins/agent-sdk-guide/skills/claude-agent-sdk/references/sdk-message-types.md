# SDKMessage Types Reference

All message types yielded by the `Query` async generator from `@anthropic-ai/claude-agent-sdk`.

`SDKMessage` is a discriminated union of 24 message types. Use the `type` field (and `subtype` for system messages) to discriminate.

---

## SDKAssistantMessage

**Discriminant:** `type: 'assistant'`

Model response containing content blocks (text, tool use, thinking).

| Field | Type | Description |
|---|---|---|
| `type` | `'assistant'` | Message type |
| `message` | `BetaMessage` | Full message from `@anthropic-ai/sdk` with `content` blocks |
| `parent_tool_use_id` | `string \| null` | Non-null when this is a subagent response |
| `error` | `SDKAssistantMessageError?` | Error category if the API call failed |
| `uuid` | `UUID` | Message identifier |
| `session_id` | `string` | Session identifier |

`SDKAssistantMessageError` values: `'authentication_failed'`, `'billing_error'`, `'rate_limit'`, `'invalid_request'`, `'server_error'`, `'unknown'`, `'max_output_tokens'`.

---

## SDKUserMessage / SDKUserMessageReplay

**Discriminant:** `type: 'user'`

User message echoed back in the stream. Key fields: `message` (MessageParam), `parent_tool_use_id`, `isSynthetic?`, `priority?` (`'now'`/`'next'`/`'later'`). `SDKUserMessageReplay` is the same shape with `isReplay: true` and required `uuid`/`session_id`, emitted when resuming a session.

---

## SDKResultMessage (SDKResultSuccess | SDKResultError)

**Discriminant:** `type: 'result'`

Final message of a query. Discriminate on `subtype`.

### SDKResultSuccess

| Field | Type | Description |
|---|---|---|
| `type` | `'result'` | Message type |
| `subtype` | `'success'` | Success indicator |
| `result` | `string` | The final text result |
| `structured_output` | `unknown?` | Parsed output when `outputFormat` was set |
| `duration_ms` | `number` | Total wall-clock time |
| `duration_api_ms` | `number` | Time spent in API calls |
| `is_error` | `boolean` | Always `false` for success |
| `num_turns` | `number` | Number of turns completed |
| `total_cost_usd` | `number` | Total cost in USD |
| `usage` | `NonNullableUsage` | Token usage breakdown |
| `modelUsage` | `Record<string, ModelUsage>` | Per-model usage |
| `permission_denials` | `SDKPermissionDenial[]` | Tools denied during execution |
| `stop_reason` | `string \| null` | API stop reason |

### SDKResultError

| Field | Type | Description |
|---|---|---|
| `type` | `'result'` | Message type |
| `subtype` | Error subtype | `'error_during_execution'`, `'error_max_turns'`, `'error_max_budget_usd'`, `'error_max_structured_output_retries'` |
| `errors` | `string[]` | Error messages |
| `is_error` | `boolean` | Always `true` for errors |
| *(other fields)* | | Same cost/usage/duration fields as success |

---

## SDKSystemMessage

**Discriminant:** `type: 'system'`, `subtype: 'init'`

First message emitted. Contains session init data: `cwd`, `model`, `tools[]`, `mcp_servers[]`, `permissionMode`, `claude_code_version`, `slash_commands[]`, `skills[]`, `plugins[]`, `agents?[]`, `betas?[]`.

---

## SDKPartialAssistantMessage

**Discriminant:** `type: 'stream_event'`

Streaming token events. Only emitted when `includePartialMessages: true`.

| Field | Type | Description |
|---|---|---|
| `type` | `'stream_event'` | Message type |
| `event` | `BetaRawMessageStreamEvent` | Raw streaming event (content_block_start, content_block_delta, etc.) |
| `parent_tool_use_id` | `string \| null` | Non-null in subagent context |
| `uuid` | `UUID` | Message identifier |
| `session_id` | `string` | Session identifier |

---

## SDKStatusMessage

**Discriminant:** `type: 'system'`, `subtype: 'status'`

Status change notification.

| Field | Type | Description |
|---|---|---|
| `status` | `SDKStatus` | `'compacting'` or `null` |
| `permissionMode` | `PermissionMode?` | Current permission mode if changed |

---

## SDKAPIRetryMessage

**Discriminant:** `type: 'system'`, `subtype: 'api_retry'`

Emitted when an API request fails with a retryable error.

| Field | Type | Description |
|---|---|---|
| `attempt` | `number` | Current retry attempt |
| `max_retries` | `number` | Maximum retries configured |
| `retry_delay_ms` | `number` | Delay before next retry |
| `error_status` | `number \| null` | HTTP status code, null for connection errors |
| `error` | `SDKAssistantMessageError` | Error category |

---

## SDKTaskStartedMessage

**Discriminant:** `type: 'system'`, `subtype: 'task_started'`

Emitted when a subagent task begins.

| Field | Type | Description |
|---|---|---|
| `task_id` | `string` | Unique task identifier |
| `tool_use_id` | `string?` | Associated tool use |
| `description` | `string` | Task description |
| `task_type` | `string?` | Task type identifier |
| `workflow_name` | `string?` | Workflow script name (for `local_workflow` tasks) |
| `prompt` | `string?` | The prompt given to the subagent |

---

## SDKTaskProgressMessage

**Discriminant:** `type: 'system'`, `subtype: 'task_progress'`

Periodic progress update for a running subagent.

| Field | Type | Description |
|---|---|---|
| `task_id` | `string` | Task identifier |
| `description` | `string` | Current activity description |
| `usage` | `{ total_tokens: number; tool_uses: number; duration_ms: number }` | Resource usage |
| `last_tool_name` | `string?` | Most recently used tool |
| `summary` | `string?` | AI-generated summary (when `agentProgressSummaries` is enabled) |

---

## SDKTaskNotificationMessage

**Discriminant:** `type: 'system'`, `subtype: 'task_notification'`

Emitted when a subagent task completes, fails, or is stopped.

| Field | Type | Description |
|---|---|---|
| `task_id` | `string` | Task identifier |
| `status` | `'completed' \| 'failed' \| 'stopped'` | Final task status |
| `output_file` | `string` | Path to task output file |
| `summary` | `string` | Result summary |
| `usage` | `{ total_tokens: number; tool_uses: number; duration_ms: number }?` | Final resource usage |

---

## SDKToolProgressMessage

**Discriminant:** `type: 'tool_progress'`

Heartbeat for long-running tool executions.

| Field | Type | Description |
|---|---|---|
| `tool_use_id` | `string` | Active tool use ID |
| `tool_name` | `string` | Tool name |
| `elapsed_time_seconds` | `number` | Time elapsed since tool start |
| `parent_tool_use_id` | `string \| null` | Non-null in subagent context |
| `task_id` | `string?` | Associated task if applicable |

---

## SDKPromptSuggestionMessage

**Discriminant:** `type: 'prompt_suggestion'`

Predicted next user prompt. Emitted after the `result` message when `promptSuggestions: true`.

| Field | Type | Description |
|---|---|---|
| `suggestion` | `string` | Suggested next prompt |

---

## SDKRateLimitEvent

**Discriminant:** `type: 'rate_limit_event'`

Rate limit status change for claude.ai subscription users.

| Field | Type | Description |
|---|---|---|
| `rate_limit_info` | `SDKRateLimitInfo` | Contains `status` (`'allowed'`, `'allowed_warning'`, `'rejected'`), `resetsAt`, `utilization`, and overage details |

---

## Other message types

| Type | Discriminant | Description |
|---|---|---|
| `SDKSessionStateChangedMessage` | `type: 'system'`, `subtype: 'session_state_changed'` | State transitions: `'idle'`, `'running'`, `'requires_action'` |
| `SDKCompactBoundaryMessage` | `type: 'system'`, `subtype: 'compact_boundary'` | Context compaction boundary with token count metadata |
| `SDKAuthStatusMessage` | `type: 'auth_status'` | OAuth/API key auth progress (`isAuthenticating`, `output[]`, `error?`) |
| `SDKToolUseSummaryMessage` | `type: 'tool_use_summary'` | Summary of preceding tool uses after compaction |
| `SDKLocalCommandOutputMessage` | `type: 'system'`, `subtype: 'local_command_output'` | Output from local slash commands (`/voice`, `/cost`) |
| `SDKHookStartedMessage` | `type: 'system'`, `subtype: 'hook_started'` | Hook began executing |
| `SDKHookProgressMessage` | `type: 'system'`, `subtype: 'hook_progress'` | Intermediate hook stdout/stderr |
| `SDKHookResponseMessage` | `type: 'system'`, `subtype: 'hook_response'` | Hook completed (`'success'`, `'error'`, `'cancelled'`) |
| `SDKFilesPersistedEvent` | `type: 'system'`, `subtype: 'files_persisted'` | Files persisted to storage (succeeded + failed lists) |
| `SDKElicitationCompleteMessage` | `type: 'system'`, `subtype: 'elicitation_complete'` | URL-mode MCP elicitation confirmed complete |

---

## Message handling example

```typescript
for await (const message of conversation) {
  switch (message.type) {
    case "system":
      if (message.subtype === "init") console.log(`Model: ${message.model}`);
      else if (message.subtype === "task_notification") console.log(`Task ${message.task_id}: ${message.status}`);
      break;
    case "assistant":
      for (const block of message.message.content) {
        if (block.type === "text") console.log(block.text);
      }
      break;
    case "result":
      if (message.subtype === "success") console.log("Result:", message.result);
      else console.error("Failed:", message.subtype, message.errors);
      break;
    case "tool_progress":
      console.log(`Tool ${message.tool_name} running (${message.elapsed_time_seconds}s)`);
      break;
    case "rate_limit_event":
      console.warn("Rate limit:", message.rate_limit_info.status);
      break;
  }
}
```
