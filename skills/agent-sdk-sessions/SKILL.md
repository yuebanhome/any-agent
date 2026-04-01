---
name: agent-sdk-sessions
description: "Use when the user asks about session management, resuming conversations, multi-turn interactions, conversation history, the V2 session API, listSessions, forkSession, or managing persistent agent conversations with the Claude Agent SDK."
---

# Claude Agent SDK — Sessions

## Overview

Sessions allow you to persist and resume agent conversations. The SDK provides two APIs:

1. **`query()`-based API (stable)** — Resume sessions via `resume` and `continue` options, manage sessions with standalone functions (`listSessions`, `getSessionInfo`, etc.).
2. **V2 Session API (unstable/alpha)** — Object-oriented `SDKSession` interface for multi-turn conversations via `unstable_v2_createSession()`, `unstable_v2_resumeSession()`, and `unstable_v2_prompt()`.

All imports come from `@anthropic-ai/claude-agent-sdk`.

---

## Session Resume with query()

Use the `resume` and `continue` options in `Options` to load prior conversation history.

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk'

// Resume a specific session by ID
const q = query({
  prompt: 'Continue where we left off',
  options: { resume: sessionId }
})
for await (const message of q) {
  // process messages
}

// Continue the most recent session in the current directory
const q2 = query({
  prompt: 'What was I working on?',
  options: { continue: true }
})

// Resume but fork to a new session (preserves original)
const q3 = query({
  prompt: 'Try a different approach',
  options: {
    resume: sessionId,
    forkSession: true,
    // Optionally specify a custom UUID for the fork:
    // sessionId: customUuid,
  }
})

// Resume from a specific point in the conversation
const q4 = query({
  prompt: 'Redo from this point',
  options: {
    resume: sessionId,
    resumeSessionAt: messageUuid, // SDKAssistantMessage.uuid
  }
})
```

Key `Options` fields for sessions:

| Field              | Type      | Description                                                        |
| ------------------ | --------- | ------------------------------------------------------------------ |
| `resume`           | `string`  | Session UUID to resume                                             |
| `continue`         | `boolean` | Resume the most recent session in cwd (mutually exclusive with `resume`) |
| `sessionId`        | `string`  | Use a specific UUID for the new session                            |
| `forkSession`      | `boolean` | Fork when resuming instead of continuing in-place                  |
| `resumeSessionAt`  | `string`  | Resume only up to this message UUID                                |
| `persistSession`   | `boolean` | Set `false` to disable saving to disk (default `true`)             |

---

## Session Management APIs

### listSessions()

List sessions with optional filtering, pagination, and project scoping.

```typescript
import { listSessions } from '@anthropic-ai/claude-agent-sdk'
import type { SDKSessionInfo, ListSessionsOptions } from '@anthropic-ai/claude-agent-sdk'

const allSessions: SDKSessionInfo[] = await listSessions()
const projectSessions = await listSessions({ dir: '/path/to/project' })
const page1 = await listSessions({ limit: 50 })
const page2 = await listSessions({ limit: 50, offset: 50 })
const noWorktrees = await listSessions({ dir: '/path/to/project', includeWorktrees: false })
```

`SDKSessionInfo` fields: `sessionId`, `summary`, `lastModified`, `fileSize?`, `customTitle?`, `firstPrompt?`, `gitBranch?`, `cwd?`, `tag?`, `createdAt?`.

### getSessionInfo()

Read metadata for a single session without scanning all sessions.

```typescript
import { getSessionInfo } from '@anthropic-ai/claude-agent-sdk'

const info = await getSessionInfo(sessionId) // SDKSessionInfo | undefined
const info2 = await getSessionInfo(sessionId, { dir: '/path/to/project' })
```

### getSessionMessages()

Read the conversation transcript (user and assistant messages).

```typescript
import { getSessionMessages } from '@anthropic-ai/claude-agent-sdk'

const messages: SessionMessage[] = await getSessionMessages(sessionId)
const recent = await getSessionMessages(sessionId, { limit: 10 })
const older = await getSessionMessages(sessionId, { limit: 10, offset: 10 })
```

`SessionMessage`: `{ type: 'user' | 'assistant', uuid, session_id, message, parent_tool_use_id }`.

### renameSession()

Set a custom display title for a session.

```typescript
import { renameSession } from '@anthropic-ai/claude-agent-sdk'

await renameSession(sessionId, 'Refactor auth module')

// With project directory
await renameSession(sessionId, 'New title', { dir: '/path/to/project' })
```

### tagSession()

Add or clear a tag on a session.

```typescript
import { tagSession } from '@anthropic-ai/claude-agent-sdk'

await tagSession(sessionId, 'in-progress')

// Clear the tag
await tagSession(sessionId, null)
```

### forkSession()

Branch a session into a new conversation, optionally slicing at a specific message.

```typescript
import { forkSession } from '@anthropic-ai/claude-agent-sdk'
import type { ForkSessionResult } from '@anthropic-ai/claude-agent-sdk'

// Fork the entire session
const result: ForkSessionResult = await forkSession(sessionId)
console.log(result.sessionId) // new UUID

// Fork up to a specific message with a custom title
const result2 = await forkSession(sessionId, {
  upToMessageId: messageUuid,
  title: 'Experiment branch',
})

// Resume the fork
const q = query({
  prompt: 'Continue from the fork',
  options: { resume: result2.sessionId }
})
```

---

## Multi-Turn Conversations with Streaming Input

Pass an `AsyncIterable<SDKUserMessage>` as the prompt for interactive multi-turn conversations within a single query.

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk'
import type { SDKUserMessage } from '@anthropic-ai/claude-agent-sdk'

async function* userMessages(): AsyncGenerator<SDKUserMessage> {
  yield { type: 'user' as const, message: { role: 'user', content: 'Analyze the auth module' }, parent_tool_use_id: null }
  // ... wait for user input ...
  yield { type: 'user' as const, message: { role: 'user', content: 'Now refactor the token refresh' }, parent_tool_use_id: null }
}

const q = query({ prompt: userMessages(), options: { model: 'claude-sonnet-4-6' } })
for await (const message of q) {
  // Process streaming messages from the agent
}
```

You can also use `Query.streamInput()` after the query has started:

```typescript
const q = query({ prompt: 'Hello', options: { ... } })

// Later, send additional messages
await q.streamInput(additionalMessages())
```

---

## Query Control Methods

The `Query` interface extends `AsyncGenerator<SDKMessage>` and exposes control methods (available in streaming input mode):

```typescript
const q = query({ prompt: 'Start working', options: { ... } })

// Interrupt the current execution
await q.interrupt()

// Change permission mode mid-session
await q.setPermissionMode('acceptEdits')  // 'default' | 'acceptEdits' | 'plan' | 'dontAsk'

// Switch model for subsequent responses
await q.setModel('claude-sonnet-4-6')
await q.setModel()  // reset to default

// Limit thinking tokens (deprecated — prefer `thinking` option in query())
await q.setMaxThinkingTokens(4096)
await q.setMaxThinkingTokens(null)  // clear limit

// Stream additional user messages into the conversation
await q.streamInput(asyncIterableOfMessages)

// Forcefully close the query and terminate the process
q.close()
```

---

## V2 API (Alpha)

> **Warning:** The V2 session API is unstable and marked `@alpha`. Function names are prefixed with `unstable_v2_` to signal that signatures may change in any release without notice. Do not use in production.

The V2 API provides an object-oriented `SDKSession` for persistent, multi-turn conversations.

```typescript
import {
  unstable_v2_createSession,
  unstable_v2_resumeSession,
  unstable_v2_prompt,
} from '@anthropic-ai/claude-agent-sdk'

// Create a new session
const session = unstable_v2_createSession({
  model: 'claude-sonnet-4-6',
  permissionMode: 'acceptEdits',
  allowedTools: ['Read', 'Edit', 'Bash'],
})

// Send a message and stream the response
await session.send('What files are in this directory?')
for await (const msg of session.stream()) {
  console.log(msg)
}

// Resume an existing session
const resumed = unstable_v2_resumeSession(existingSessionId, {
  model: 'claude-sonnet-4-6',
})

// Single-turn convenience (no session object)
const result = await unstable_v2_prompt('What files are here?', {
  model: 'claude-sonnet-4-6',
})

// Clean up
session.close()
```

For the full V2 API reference including `SDKSessionOptions`, `SDKSession` methods, lifecycle patterns, and migration notes, see [references/v2-session-api.md](references/v2-session-api.md).

---

## File Checkpointing

Enable file checkpointing to track and rewind file changes made during a session.

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk'

const q = query({
  prompt: 'Refactor the auth module',
  options: {
    enableFileCheckpointing: true,
  }
})

for await (const msg of q) { /* process messages, track UUIDs */ }

// Rewind files to their state at a specific user message
const result = await q.rewindFiles(userMessageUuid)
// RewindFilesResult: { canRewind, error?, filesChanged?, insertions?, deletions? }

// Dry run to preview changes without modifying files
const preview = await q.rewindFiles(userMessageUuid, { dryRun: true })
```
