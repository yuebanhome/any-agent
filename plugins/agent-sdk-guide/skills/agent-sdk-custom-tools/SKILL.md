---
name: agent-sdk-custom-tools
description: "Use when the user asks about adding custom tools to a Claude agent, mentions createSdkMcpServer, tool() helper, MCP tools integration, in-process tools, connecting external MCP servers to the Agent SDK, or wants to extend agent capabilities with custom functions."
---

# Custom Tools for the Claude Agent SDK

## Overview

There are two approaches to adding tools to a Claude Agent SDK session:

1. **In-process SDK tools** -- Use `createSdkMcpServer()` + `tool()` to define tools that run in the same Node.js process. Best for custom business logic, database queries, API calls, or anything that benefits from sharing memory with your application.

2. **External MCP servers** -- Connect to standalone MCP servers via stdio, HTTP, or SSE transports. Best for reusing existing MCP servers, language-agnostic tools, or tools that need process isolation.

Both approaches use the `mcpServers` option in `query()`. You can combine them freely.

## In-Process Tools with createSdkMcpServer + tool()

This is the primary pattern. The `tool()` helper creates an `SdkMcpToolDefinition` and `createSdkMcpServer()` wraps them into an MCP server config the SDK can use.

### Complete Working Example

```typescript
import { query, createSdkMcpServer, tool } from '@anthropic-ai/claude-agent-sdk'
import { z } from 'zod/v4'

// Define tools
const getWeather = tool(
  'get-weather',
  'Get current weather for a city',
  { city: z.string(), units: z.enum(['celsius', 'fahrenheit']).optional() },
  async (args) => {
    const response = await fetch(
      `https://api.weather.example/v1?city=${args.city}&units=${args.units ?? 'celsius'}`
    )
    const data = await response.json()
    return { content: [{ type: 'text', text: JSON.stringify(data) }] }
  }
)

// Create the server config
const server = createSdkMcpServer({
  name: 'weather-tools',
  tools: [getWeather],
})

// Run a query with the tools available
const q = query({
  prompt: 'What is the weather in Tokyo?',
  options: {
    mcpServers: { 'weather-tools': server },
  },
})

// Iterate over results
for await (const message of q) {
  if (message.type === 'assistant') {
    console.log(message.message.content)
  } else if (message.type === 'result') {
    console.log('Final result:', message.result)
  }
}
```

### Key Points

- Import `z` from `'zod/v4'` -- the SDK uses Zod v4 internally.
- The `tool()` handler must return a `CallToolResult` from `@modelcontextprotocol/sdk/types.js`. The most common shape is `{ content: [{ type: 'text', text: '...' }] }`.
- `createSdkMcpServer()` returns a `McpSdkServerConfigWithInstance` which includes `{ type: 'sdk', name, instance }`.
- If tool handlers run longer than 60 seconds, set the `CLAUDE_CODE_STREAM_CLOSE_TIMEOUT` environment variable.

## External MCP Servers

Pass external server configs in the same `mcpServers` record. Four transport types are supported:

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk'

const q = query({
  prompt: 'Use my tools',
  options: {
    mcpServers: {
      // stdio: spawns a child process
      'file-server': {
        type: 'stdio',
        command: 'node',
        args: ['./my-mcp-server.js'],
        env: { DEBUG: 'true' },
      },

      // http: Streamable HTTP transport
      'remote-http': {
        type: 'http',
        url: 'https://my-server.example/mcp',
        headers: { Authorization: 'Bearer token' },
      },

      // sse: Server-Sent Events transport
      'remote-sse': {
        type: 'sse',
        url: 'https://my-server.example/sse',
        headers: { Authorization: 'Bearer token' },
      },

      // sdk: in-process server (what createSdkMcpServer returns)
      'my-sdk-tools': createSdkMcpServer({
        name: 'my-sdk-tools',
        tools: [/* ... */],
      }),
    },
  },
})
```

### Type Summary

- **stdio**: `{ type?: 'stdio', command: string, args?: string[], env?: Record<string, string> }`
- **http**: `{ type: 'http', url: string, headers?: Record<string, string> }`
- **sse**: `{ type: 'sse', url: string, headers?: Record<string, string> }`
- **sdk**: `{ type: 'sdk', name: string, instance: McpServer }` -- returned by `createSdkMcpServer()`

`McpServerConfig` is the union of all four.

## Tool Return Types

The `CallToolResult` type supports several content formats:

```typescript
// Text result
return { content: [{ type: 'text', text: 'Hello world' }] }

// Image result (base64)
return {
  content: [{
    type: 'image',
    data: base64String,
    mimeType: 'image/png',
  }],
}

// Embedded resource
return {
  content: [{
    type: 'resource',
    resource: {
      uri: 'file:///path/to/file.txt',
      text: fileContents,
      mimeType: 'text/plain',
    },
  }],
}

// Error result
return {
  content: [{ type: 'text', text: 'Something went wrong' }],
  isError: true,
}

// Multiple content items
return {
  content: [
    { type: 'text', text: 'Here is the chart:' },
    { type: 'image', data: chartBase64, mimeType: 'image/png' },
  ],
}
```

## Tool Extras and Annotations

The `tool()` helper accepts an optional fifth argument for metadata:

```typescript
const myTool = tool(
  'search-docs',
  'Search documentation by keyword',
  { keyword: z.string(), limit: z.number().optional() },
  async (args) => {
    const results = await searchIndex(args.keyword, args.limit ?? 10)
    return { content: [{ type: 'text', text: JSON.stringify(results) }] }
  },
  {
    // Hint text that helps Claude decide when to use this tool
    searchHint: 'documentation search lookup reference',

    // If true, the tool definition is always included in context
    alwaysLoad: true,

    // MCP ToolAnnotations
    annotations: {
      readOnlyHint: true,       // tool does not modify state
      destructiveHint: false,   // tool does not destroy data
      idempotentHint: true,     // safe to retry
      openWorldHint: true,      // interacts with external systems
      title: 'Search Docs',     // human-readable name
    },
  }
)
```

## MCP Server Management at Runtime

The `Query` object returned by `query()` exposes methods to manage MCP servers dynamically during a session:

```typescript
const q = query({ prompt: 'Hello', options: { mcpServers: {} } })

// Replace the full set of dynamic MCP servers
const result = await q.setMcpServers({
  'new-server': {
    type: 'stdio',
    command: 'node',
    args: ['./new-server.js'],
  },
})
// result: { added: ['new-server'], removed: [], errors: {} }

// Reconnect a server that may have dropped
await q.reconnectMcpServer('new-server')

// Disable a server temporarily
await q.toggleMcpServer('new-server', false)

// Re-enable it
await q.toggleMcpServer('new-server', true)
```

`setMcpServers()` returns an `McpSetServersResult`:
```typescript
type McpSetServersResult = {
  added: string[]
  removed: string[]
  errors: Record<string, string>
}
```

## Common Patterns

### Database Query Tool

```typescript
import { query, createSdkMcpServer, tool } from '@anthropic-ai/claude-agent-sdk'
import { z } from 'zod/v4'
import { Pool } from 'pg'

const pool = new Pool({ connectionString: process.env.DATABASE_URL })

const dbQuery = tool(
  'db-query',
  'Run a read-only SQL query against the application database. Returns rows as JSON.',
  {
    sql: z.string().describe('SELECT query to execute'),
    params: z.array(z.string()).optional().describe('Parameterized values'),
  },
  async (args) => {
    try {
      const result = await pool.query(args.sql, args.params)
      return {
        content: [{
          type: 'text',
          text: JSON.stringify(result.rows, null, 2),
        }],
      }
    } catch (err) {
      return {
        content: [{ type: 'text', text: `Query error: ${(err as Error).message}` }],
        isError: true,
      }
    }
  },
  { annotations: { readOnlyHint: true, destructiveHint: false } }
)

const server = createSdkMcpServer({ name: 'db-tools', tools: [dbQuery] })

const q = query({
  prompt: 'How many users signed up this month?',
  options: { mcpServers: { 'db-tools': server } },
})

for await (const message of q) {
  if (message.type === 'result') {
    console.log(message.result)
  }
}
```

