---
name: mcp-server-dev
description: |
  Patterns for building Model Context Protocol (MCP) servers including
  tools, resources, prompts, and transport handling. Use when creating
  MCP servers to extend AI assistant capabilities.
---

# MCP Server Development Patterns

Patterns for building Model Context Protocol servers.

## Overview

MCP (Model Context Protocol) allows AI assistants to interact with external tools, resources, and data sources through a standardized protocol.

## Setup

```bash
npm install @modelcontextprotocol/sdk
```

## Basic Server Structure

```typescript
// src/index.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
} from '@modelcontextprotocol/sdk/types.js'

const server = new Server(
  {
    name: 'my-mcp-server',
    version: '1.0.0',
  },
  {
    capabilities: {
      tools: {},
      resources: {},
      prompts: {},
    },
  }
)

// Start server
const transport = new StdioServerTransport()
await server.connect(transport)
```

## Implementing Tools

```typescript
// List available tools
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: 'search_files',
        description: 'Search for files by name or content',
        inputSchema: {
          type: 'object',
          properties: {
            query: {
              type: 'string',
              description: 'Search query',
            },
            path: {
              type: 'string',
              description: 'Directory to search in',
            },
            maxResults: {
              type: 'number',
              description: 'Maximum results to return',
              default: 10,
            },
          },
          required: ['query'],
        },
      },
      {
        name: 'execute_command',
        description: 'Execute a shell command',
        inputSchema: {
          type: 'object',
          properties: {
            command: {
              type: 'string',
              description: 'Command to execute',
            },
            cwd: {
              type: 'string',
              description: 'Working directory',
            },
          },
          required: ['command'],
        },
      },
    ],
  }
})

// Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params

  switch (name) {
    case 'search_files': {
      const results = await searchFiles(args.query, args.path, args.maxResults)
      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify(results, null, 2),
          },
        ],
      }
    }

    case 'execute_command': {
      const output = await executeCommand(args.command, args.cwd)
      return {
        content: [
          {
            type: 'text',
            text: output,
          },
        ],
      }
    }

    default:
      throw new Error(`Unknown tool: ${name}`)
  }
})
```

## Implementing Resources

```typescript
// List available resources
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return {
    resources: [
      {
        uri: 'file:///config/settings.json',
        name: 'Application Settings',
        description: 'Current application configuration',
        mimeType: 'application/json',
      },
      {
        uri: 'db://users',
        name: 'Users Database',
        description: 'User records from the database',
        mimeType: 'application/json',
      },
    ],
  }
})

// Read resource content
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const { uri } = request.params

  if (uri === 'file:///config/settings.json') {
    const content = await fs.readFile('config/settings.json', 'utf-8')
    return {
      contents: [
        {
          uri,
          mimeType: 'application/json',
          text: content,
        },
      ],
    }
  }

  if (uri === 'db://users') {
    const users = await db.user.findMany()
    return {
      contents: [
        {
          uri,
          mimeType: 'application/json',
          text: JSON.stringify(users, null, 2),
        },
      ],
    }
  }

  throw new Error(`Resource not found: ${uri}`)
})
```

## Dynamic Resources with Templates

```typescript
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return {
    resources: [],
    resourceTemplates: [
      {
        uriTemplate: 'github://repos/{owner}/{repo}',
        name: 'GitHub Repository',
        description: 'Information about a GitHub repository',
      },
      {
        uriTemplate: 'file:///{path}',
        name: 'File System',
        description: 'Read files from the filesystem',
      },
    ],
  }
})

server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const { uri } = request.params

  // Parse URI and handle accordingly
  if (uri.startsWith('github://repos/')) {
    const match = uri.match(/github:\/\/repos\/([^/]+)\/([^/]+)/)
    if (match) {
      const [, owner, repo] = match
      const data = await fetchGitHubRepo(owner, repo)
      return {
        contents: [
          {
            uri,
            mimeType: 'application/json',
            text: JSON.stringify(data, null, 2),
          },
        ],
      }
    }
  }

  throw new Error(`Cannot resolve: ${uri}`)
})
```

## Implementing Prompts

```typescript
import {
  ListPromptsRequestSchema,
  GetPromptRequestSchema,
} from '@modelcontextprotocol/sdk/types.js'

server.setRequestHandler(ListPromptsRequestSchema, async () => {
  return {
    prompts: [
      {
        name: 'code_review',
        description: 'Review code for best practices',
        arguments: [
          {
            name: 'language',
            description: 'Programming language',
            required: true,
          },
          {
            name: 'code',
            description: 'Code to review',
            required: true,
          },
        ],
      },
      {
        name: 'explain_error',
        description: 'Explain an error message',
        arguments: [
          {
            name: 'error',
            description: 'Error message or stack trace',
            required: true,
          },
        ],
      },
    ],
  }
})

server.setRequestHandler(GetPromptRequestSchema, async (request) => {
  const { name, arguments: args } = request.params

  switch (name) {
    case 'code_review':
      return {
        messages: [
          {
            role: 'user',
            content: {
              type: 'text',
              text: `Please review this ${args.language} code for best practices, potential bugs, and improvements:\n\n\`\`\`${args.language}\n${args.code}\n\`\`\``,
            },
          },
        ],
      }

    case 'explain_error':
      return {
        messages: [
          {
            role: 'user',
            content: {
              type: 'text',
              text: `Please explain this error and suggest how to fix it:\n\n${args.error}`,
            },
          },
        ],
      }

    default:
      throw new Error(`Unknown prompt: ${name}`)
  }
})
```

## Error Handling

```typescript
import { McpError, ErrorCode } from '@modelcontextprotocol/sdk/types.js'

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  try {
    // Tool implementation
  } catch (error) {
    if (error instanceof ValidationError) {
      throw new McpError(
        ErrorCode.InvalidParams,
        `Invalid parameters: ${error.message}`
      )
    }

    if (error instanceof NotFoundError) {
      throw new McpError(
        ErrorCode.InvalidRequest,
        `Resource not found: ${error.message}`
      )
    }

    // Unknown error
    throw new McpError(
      ErrorCode.InternalError,
      `Internal error: ${error.message}`
    )
  }
})
```

## Progress Notifications

```typescript
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params
  const progressToken = request.params._meta?.progressToken

  if (name === 'long_running_task') {
    const totalSteps = 10

    for (let i = 0; i < totalSteps; i++) {
      // Do work...
      await doStep(i)

      // Report progress
      if (progressToken) {
        await server.notification({
          method: 'notifications/progress',
          params: {
            progressToken,
            progress: i + 1,
            total: totalSteps,
          },
        })
      }
    }

    return { content: [{ type: 'text', text: 'Task completed!' }] }
  }
})
```

## HTTP Transport (SSE)

```typescript
import express from 'express'
import { SSEServerTransport } from '@modelcontextprotocol/sdk/server/sse.js'

const app = express()

app.get('/sse', async (req, res) => {
  const transport = new SSEServerTransport('/messages', res)
  await server.connect(transport)
})

app.post('/messages', async (req, res) => {
  // Handle incoming messages
})

app.listen(3001)
```

## Package Configuration

```json
{
  "name": "my-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "bin": {
    "my-mcp-server": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0"
  }
}
```

## Claude Desktop Configuration

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["/path/to/my-mcp-server/dist/index.js"],
      "env": {
        "API_KEY": "your-api-key"
      }
    }
  }
}
```

## Best Practices

1. **Validate inputs** - Use schemas for all tool parameters
2. **Handle errors gracefully** - Use proper MCP error codes
3. **Document clearly** - Good descriptions help the AI use tools correctly
4. **Report progress** - For long-running operations
5. **Use appropriate transports** - stdio for CLI, SSE for web
6. **Keep tools focused** - One responsibility per tool
7. **Return structured data** - JSON for complex responses
