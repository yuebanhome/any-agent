# Advanced MCP Tool Patterns

Reference for advanced patterns when building custom tools with the Claude Agent SDK.

## Multiple Tools in One Server

Group related tools into a single `createSdkMcpServer` call:

```typescript
import { createSdkMcpServer, tool } from '@anthropic-ai/claude-agent-sdk'
import { z } from 'zod/v4'

const server = createSdkMcpServer({
  name: 'user-management',
  tools: [
    tool(
      'get-user',
      'Fetch a user by ID',
      { userId: z.string() },
      async (args) => {
        const user = await db.users.findById(args.userId)
        return { content: [{ type: 'text', text: JSON.stringify(user) }] }
      },
      { annotations: { readOnlyHint: true } }
    ),

    tool('update-user', 'Update user profile fields',
      { userId: z.string(), updates: z.object({ name: z.string().optional(), email: z.email().optional(), role: z.enum(['admin', 'member', 'viewer']).optional() }) },
      async (args) => {
        const updated = await db.users.update(args.userId, args.updates)
        return { content: [{ type: 'text', text: JSON.stringify(updated) }] }
      },
      { annotations: { readOnlyHint: false, destructiveHint: false } }
    ),

    tool('delete-user', 'Permanently delete a user account',
      { userId: z.string(), confirm: z.literal('DELETE') },
      async (args) => {
        await db.users.delete(args.userId)
        return { content: [{ type: 'text', text: `User ${args.userId} deleted.` }] }
      },
      { annotations: { destructiveHint: true } }
    ),
  ],
})
```

## Error Handling in Tool Handlers

Return `isError: true` so Claude knows the call failed and can retry or adjust:

```typescript
const safeTool = tool(
  'call-api',
  'Call an external API endpoint',
  { url: z.string(), method: z.enum(['GET', 'POST']).optional() },
  async (args) => {
    try {
      const response = await fetch(args.url, { method: args.method ?? 'GET' })

      if (!response.ok) {
        return {
          content: [{
            type: 'text',
            text: `HTTP ${response.status}: ${response.statusText}`,
          }],
          isError: true,
        }
      }

      const body = await response.text()
      return { content: [{ type: 'text', text: body }] }
    } catch (err) {
      return {
        content: [{
          type: 'text',
          text: `Network error: ${(err as Error).message}`,
        }],
        isError: true,
      }
    }
  }
)
```

Always return `isError: true` rather than throwing -- thrown exceptions crash the MCP transport. Include enough detail for Claude to understand the failure.

## Tools That Return Images

Return base64-encoded image data with the `image` content type:

```typescript
const generateChart = tool(
  'generate-chart',
  'Generate a chart image from data points',
  {
    title: z.string(),
    dataPoints: z.array(z.object({ label: z.string(), value: z.number() })),
    chartType: z.enum(['bar', 'line', 'pie']),
  },
  async (args) => {
    const buffer = await renderChart(args)
    return {
      content: [
        { type: 'text', text: `Chart: ${args.title}` },
        { type: 'image', data: buffer.toString('base64'), mimeType: 'image/png' },
      ],
    }
  }
)
```

You can also return multiple content items mixing text and images in one response.

## Tools with Complex Zod Schemas

The SDK supports both Zod 3 and Zod 4 schemas. Use `zod/v4` for new code.

### Nested Objects

```typescript
const deployTool = tool('deploy-service', 'Deploy a service', {
  service: z.object({
    name: z.string(),
    version: z.string(),
    config: z.object({
      replicas: z.number().int().min(1).max(10),
      env: z.record(z.string(), z.string()).optional(),
    }),
  }),
  environment: z.enum(['staging', 'production']),
  dryRun: z.boolean().optional(),
}, async (args) => {
  const result = await deployer.deploy(args.service, args.environment)
  return { content: [{ type: 'text', text: JSON.stringify(result) }] }
}, { annotations: { destructiveHint: true } })
```

### Arrays, Enums, and Optional Fields

```typescript
const batchProcess = tool(
  'batch-process',
  'Process multiple items in a batch',
  {
    items: z.array(z.object({
      id: z.string(),
      action: z.enum(['create', 'update', 'delete']),
      data: z.record(z.string(), z.unknown()).optional(),
    })).min(1).max(100),
    parallel: z.boolean().optional(),
    pageSize: z.number().int().min(1).max(100).optional(),
  },
  async (args) => {
    const results = await processBatch(args.items, { parallel: args.parallel })
    return { content: [{ type: 'text', text: JSON.stringify(results) }] }
  }
)
```

## Combining In-Process and External MCP Servers

Mix SDK tools with external servers in a single `mcpServers` record:

```typescript
const appTools = createSdkMcpServer({
  name: 'app-tools',
  tools: [
    tool('get-config', 'Read application config', {}, async () => {
      const config = await loadConfig()
      return { content: [{ type: 'text', text: JSON.stringify(config) }] }
    }),
  ],
})

const q = query({
  prompt: 'Analyze the project and check the config',
  options: {
    mcpServers: {
      'app-tools': appTools,
      'filesystem': { type: 'stdio', command: 'npx', args: ['-y', '@modelcontextprotocol/server-filesystem', '/workspace'] },
      'remote-api': { type: 'http', url: 'https://mcp.example.com/api', headers: { Authorization: `Bearer ${process.env.API_TOKEN}` } },
    },
  },
})
```

## Tool Validation and Testing

### Unit Testing Tool Handlers

Extract handlers into testable functions and test them independently:

```typescript
// tools/get-user.ts -- export handler separately for unit testing
export const getUserSchema = { userId: z.string().uuid() }

export async function getUserHandler(args: { userId: string }) {
  const user = await db.users.findById(args.userId)
  if (!user) {
    return { content: [{ type: 'text' as const, text: `User ${args.userId} not found` }], isError: true }
  }
  return { content: [{ type: 'text' as const, text: JSON.stringify(user) }] }
}

export const getUserTool = tool('get-user', 'Fetch a user by ID', getUserSchema, getUserHandler, { annotations: { readOnlyHint: true } })
```

```typescript
// tools/get-user.test.ts
import { describe, it, expect } from 'vitest'
import { getUserHandler, getUserSchema } from './get-user.js'

describe('get-user tool', () => {
  it('returns user data for valid ID', async () => {
    const result = await getUserHandler({ userId: 'abc-123' })
    expect(result.isError).toBeUndefined()
  })

  it('returns isError for missing user', async () => {
    const result = await getUserHandler({ userId: 'nonexistent' })
    expect(result.isError).toBe(true)
  })

  it('validates input schema', () => {
    const schema = z.object(getUserSchema)
    expect(() => schema.parse({ userId: 'not-a-uuid' })).toThrow()
  })
})
```

### Integration Testing

For end-to-end validation, run a query with `maxTurns: 1` to confirm the tool is callable:

```typescript
const server = createSdkMcpServer({ name: 'test-tools', tools: [getUserTool] })
const q = query({
  prompt: 'Call get-user with userId "550e8400-e29b-41d4-a716-446655440000"',
  options: { mcpServers: { 'test-tools': server }, maxTurns: 1 },
})
for await (const msg of q) {
  if (msg.type === 'result') console.log('Passed:', msg.result)
}
```
