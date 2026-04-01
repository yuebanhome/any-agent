# V2 Session API Reference

> **Warning:** This entire API is unstable and marked `@alpha`. All function names carry the `unstable_v2_` prefix. Signatures, behavior, and availability may change in any release without notice. **Do not depend on this API for production workloads.**

---

## SDKSessionOptions

Options passed to `unstable_v2_createSession()` and `unstable_v2_resumeSession()`.

```typescript
type SDKSessionOptions = {
  /** Model to use (required). E.g. 'claude-sonnet-4-6', 'claude-opus-4-6'. */
  model: string

  /** Path to Claude Code executable. Omit to auto-detect. */
  pathToClaudeCodeExecutable?: string

  /** Executable runtime: 'node' (default) or 'bun'. */
  executable?: 'node' | 'bun'

  /** Additional arguments passed to the executable. */
  executableArgs?: string[]

  /**
   * Environment variables for the Claude Code process.
   * Defaults to `process.env`.
   * Set CLAUDE_AGENT_SDK_CLIENT_APP for User-Agent identification.
   */
  env?: { [envVar: string]: string | undefined }

  /** Tool names auto-allowed without permission prompts. */
  allowedTools?: string[]

  /** Tool names to disallow entirely (removed from model context). */
  disallowedTools?: string[]

  /** Custom permission handler called before each tool execution. */
  canUseTool?: CanUseTool

  /** Hook callbacks for responding to events during execution. */
  hooks?: Partial<Record<HookEvent, HookCallbackMatcher[]>>

  /**
   * Permission mode for the session.
   * - 'default'     — Standard prompting for dangerous operations
   * - 'acceptEdits' — Auto-accept file edit operations
   * - 'plan'        — Planning mode, no tool execution
   * - 'dontAsk'     — Deny if not pre-approved, never prompt
   */
  permissionMode?: PermissionMode
}
```

---

## SDKSession Interface

Returned by `unstable_v2_createSession()` and `unstable_v2_resumeSession()`. Represents a persistent, multi-turn conversation.

```typescript
interface SDKSession {
  /**
   * The session ID (UUID). Available after the first message is received.
   * For resumed sessions, available immediately.
   * Throws if accessed before initialization.
   */
  readonly sessionId: string

  /**
   * Send a message to the agent.
   * Accepts a plain string or a full SDKUserMessage object.
   */
  send(message: string | SDKUserMessage): Promise<void>

  /**
   * Stream messages from the agent.
   * Returns an AsyncGenerator that yields SDKMessage objects
   * until the agent turn completes.
   */
  stream(): AsyncGenerator<SDKMessage, void>

  /**
   * Close the session and terminate the underlying process.
   * After calling close(), no further messages can be sent or received.
   */
  close(): void

  /**
   * Async disposal support via Symbol.asyncDispose.
   * Calls close() if the session is not already closed.
   * Enables `await using session = unstable_v2_createSession(...)`.
   */
  [Symbol.asyncDispose](): Promise<void>
}
```

---

## Session Lifecycle

### Create, Send, Stream, Close

The standard lifecycle for a V2 session:

```typescript
import { unstable_v2_createSession } from '@anthropic-ai/claude-agent-sdk'

// 1. Create
const session = unstable_v2_createSession({
  model: 'claude-sonnet-4-6',
  permissionMode: 'acceptEdits',
  allowedTools: ['Read', 'Edit', 'Bash'],
})

// 2. Send a message
await session.send('List all TypeScript files in src/')

// 3. Stream the response
for await (const msg of session.stream()) {
  if (msg.type === 'assistant') {
    console.log(msg.message)
  }
}

// 4. Send follow-up (multi-turn)
await session.send('Now add unit tests for each file')
for await (const msg of session.stream()) {
  // process response
}

// 5. Close when done
session.close()
```

### Using `await using` (auto-dispose)

`SDKSession` supports `Symbol.asyncDispose`, so `await using` will auto-close:

```typescript
{
  await using session = unstable_v2_createSession({ model: 'claude-sonnet-4-6' })
  await session.send('Hello')
  for await (const msg of session.stream()) { console.log(msg) }
} // session.close() called automatically
```

---

## Resume Pattern

Resume an existing session to continue a prior conversation with full history:

```typescript
import {
  unstable_v2_resumeSession,
  listSessions,
} from '@anthropic-ai/claude-agent-sdk'

// Find the session to resume
const sessions = await listSessions({ dir: '/path/to/project', limit: 1 })
const targetId = sessions[0].sessionId

// Resume it
const session = unstable_v2_resumeSession(targetId, {
  model: 'claude-sonnet-4-6',
  permissionMode: 'default',
})

// The sessionId is available immediately for resumed sessions
console.log(session.sessionId) // same as targetId

// Continue the conversation
await session.send('What were we working on?')
for await (const msg of session.stream()) {
  // process response with full prior context
}

session.close()
```

---

## unstable_v2_prompt()

A single-turn convenience function. Creates a session, sends one message, collects the full response, and returns it. No session object is exposed.

```typescript
import { unstable_v2_prompt } from '@anthropic-ai/claude-agent-sdk'
import type { SDKResultMessage } from '@anthropic-ai/claude-agent-sdk'

const result: SDKResultMessage = await unstable_v2_prompt(
  'What files are in this directory?',
  { model: 'claude-sonnet-4-6' }
)
```

Use `unstable_v2_prompt()` for fire-and-forget queries where you do not need multi-turn interaction or streaming control.

---

## Differences from the query() API

| Aspect               | `query()` (stable)                        | V2 Session API (alpha)                        |
| -------------------- | ----------------------------------------- | --------------------------------------------- |
| **Multi-turn**       | Via `AsyncIterable<SDKUserMessage>` prompt or `streamInput()` | Explicit `send()` / `stream()` loop           |
| **Resume**           | `options.resume` / `options.continue`     | `unstable_v2_resumeSession()`                 |
| **Return type**      | `Query` (AsyncGenerator + control methods)| `SDKSession` (send/stream/close)              |
| **Control methods**  | `interrupt()`, `setModel()`, `setPermissionMode()`, etc. | Not available on `SDKSession`        |
| **File checkpointing** | `enableFileCheckpointing` + `rewindFiles()` | Not available                              |
| **Stability**        | Stable                                    | Alpha, may change without notice              |
| **Session forking**  | `options.forkSession`                     | Use `forkSession()` standalone function       |
| **Streaming**        | `for await (const msg of q)`              | `for await (const msg of session.stream())`   |

---

## Migration Considerations

If you are evaluating the V2 API:

1. **Do not migrate production code.** The `unstable_v2_` prefix signals that breaking changes are expected.
2. **The stable `query()` API covers all session use cases** via `resume`, `continue`, `forkSession`, `streamInput()`, and `AsyncIterable<SDKUserMessage>` prompts.
3. **V2 is simpler for basic multi-turn patterns** — `send()` / `stream()` may be more intuitive than managing an `AsyncIterable` prompt generator.
4. **V2 lacks control methods.** If you need `interrupt()`, `setModel()`, `setPermissionMode()`, `setMaxThinkingTokens()`, or `rewindFiles()`, use the `query()` API.
5. **V2 lacks file checkpointing.** The `enableFileCheckpointing` option and `rewindFiles()` method are only available on `Query`.

When the V2 API stabilizes, a migration guide will be provided. Until then, prefer the `query()` API for production use.

---

> **Reminder:** All `unstable_v2_*` functions and the `SDKSession` interface are subject to change. Pin your SDK version and test thoroughly after any upgrade.
