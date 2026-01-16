---
name: vercel-ai-sdk
description: |
  Patterns for building AI applications with Vercel AI SDK including
  streaming LLM responses, tool calling, structured outputs, and
  multi-provider support. Use when integrating LLMs into applications.
---

# Vercel AI SDK Patterns

Modern patterns for building AI-powered applications.

## Setup

```bash
npm install ai @ai-sdk/openai @ai-sdk/anthropic
```

## Basic Streaming Chat

### Server Action

```typescript
// app/actions.ts
'use server'

import { createStreamableValue } from 'ai/rsc'
import { streamText } from 'ai'
import { openai } from '@ai-sdk/openai'

export async function chat(messages: Message[]) {
  const stream = createStreamableValue('')

  ;(async () => {
    const { textStream } = await streamText({
      model: openai('gpt-4o'),
      messages,
    })

    for await (const text of textStream) {
      stream.update(text)
    }

    stream.done()
  })()

  return { output: stream.value }
}
```

### Client Component

```typescript
'use client'

import { useChat } from 'ai/react'

export function Chat() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } = useChat({
    api: '/api/chat',
  })

  return (
    <div className="flex flex-col h-screen">
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((message) => (
          <div
            key={message.id}
            className={message.role === 'user' ? 'text-right' : 'text-left'}
          >
            <div className={`inline-block p-3 rounded-lg ${
              message.role === 'user' 
                ? 'bg-primary-500 text-white' 
                : 'bg-gray-100'
            }`}>
              {message.content}
            </div>
          </div>
        ))}
      </div>

      <form onSubmit={handleSubmit} className="p-4 border-t">
        <div className="flex gap-2">
          <input
            value={input}
            onChange={handleInputChange}
            placeholder="Type a message..."
            className="flex-1 p-2 border rounded-lg"
            disabled={isLoading}
          />
          <button 
            type="submit" 
            disabled={isLoading}
            className="px-4 py-2 bg-primary-500 text-white rounded-lg"
          >
            Send
          </button>
        </div>
      </form>
    </div>
  )
}
```

## Route Handler

```typescript
// app/api/chat/route.ts
import { openai } from '@ai-sdk/openai'
import { streamText } from 'ai'

export async function POST(req: Request) {
  const { messages } = await req.json()

  const result = await streamText({
    model: openai('gpt-4o'),
    system: 'You are a helpful assistant.',
    messages,
  })

  return result.toDataStreamResponse()
}
```

## Tool Calling

```typescript
import { openai } from '@ai-sdk/openai'
import { generateText, tool } from 'ai'
import { z } from 'zod'

const result = await generateText({
  model: openai('gpt-4o'),
  tools: {
    weather: tool({
      description: 'Get the weather for a location',
      parameters: z.object({
        city: z.string().describe('The city to get weather for'),
        unit: z.enum(['celsius', 'fahrenheit']).optional(),
      }),
      execute: async ({ city, unit = 'celsius' }) => {
        // Call weather API
        const weather = await getWeather(city, unit)
        return weather
      },
    }),
    
    searchWeb: tool({
      description: 'Search the web for information',
      parameters: z.object({
        query: z.string().describe('The search query'),
      }),
      execute: async ({ query }) => {
        const results = await searchWeb(query)
        return results
      },
    }),
  },
  maxSteps: 5, // Allow multiple tool calls
  prompt: 'What is the weather in Paris and what are the top news today?',
})

// Access tool results
for (const step of result.steps) {
  for (const toolResult of step.toolResults) {
    console.log(`Tool: ${toolResult.toolName}`)
    console.log(`Result: ${JSON.stringify(toolResult.result)}`)
  }
}
```

## Streaming Tool Calls

```typescript
// app/api/chat/route.ts
import { openai } from '@ai-sdk/openai'
import { streamText, tool } from 'ai'
import { z } from 'zod'

export async function POST(req: Request) {
  const { messages } = await req.json()

  const result = await streamText({
    model: openai('gpt-4o'),
    messages,
    tools: {
      getStockPrice: tool({
        description: 'Get stock price',
        parameters: z.object({
          symbol: z.string(),
        }),
        execute: async ({ symbol }) => {
          return { symbol, price: 150.25, change: +2.5 }
        },
      }),
    },
  })

  return result.toDataStreamResponse()
}
```

```typescript
// Client handling tool calls
'use client'

import { useChat } from 'ai/react'

export function ChatWithTools() {
  const { messages, input, handleSubmit, handleInputChange } = useChat({
    api: '/api/chat',
    onToolCall: async ({ toolCall }) => {
      // Handle tool calls on client if needed
      console.log('Tool called:', toolCall.toolName)
    },
  })

  return (
    <div>
      {messages.map((m) => (
        <div key={m.id}>
          {m.role}: {m.content}
          {m.toolInvocations?.map((tool) => (
            <div key={tool.toolCallId}>
              Tool: {tool.toolName}
              {tool.state === 'result' && (
                <pre>{JSON.stringify(tool.result, null, 2)}</pre>
              )}
            </div>
          ))}
        </div>
      ))}
    </div>
  )
}
```

## Structured Output

```typescript
import { openai } from '@ai-sdk/openai'
import { generateObject } from 'ai'
import { z } from 'zod'

const { object } = await generateObject({
  model: openai('gpt-4o'),
  schema: z.object({
    recipe: z.object({
      name: z.string(),
      ingredients: z.array(z.object({
        name: z.string(),
        amount: z.string(),
      })),
      steps: z.array(z.string()),
      prepTime: z.number().describe('Preparation time in minutes'),
      cookTime: z.number().describe('Cooking time in minutes'),
    }),
  }),
  prompt: 'Generate a recipe for chocolate chip cookies',
})

console.log(object.recipe.name)
console.log(object.recipe.ingredients)
```

## Multi-Provider Support

```typescript
import { openai } from '@ai-sdk/openai'
import { anthropic } from '@ai-sdk/anthropic'
import { google } from '@ai-sdk/google'
import { streamText } from 'ai'

// Switch providers easily
const providers = {
  openai: openai('gpt-4o'),
  anthropic: anthropic('claude-3-5-sonnet-20241022'),
  google: google('gemini-1.5-pro'),
}

export async function chat(
  messages: Message[], 
  provider: keyof typeof providers
) {
  const result = await streamText({
    model: providers[provider],
    messages,
  })

  return result.toDataStreamResponse()
}
```

## Image Input

```typescript
import { openai } from '@ai-sdk/openai'
import { generateText } from 'ai'

const result = await generateText({
  model: openai('gpt-4o'),
  messages: [
    {
      role: 'user',
      content: [
        { type: 'text', text: 'Describe this image' },
        { type: 'image', image: imageUrl },
      ],
    },
  ],
})
```

## Embeddings

```typescript
import { openai } from '@ai-sdk/openai'
import { embed, embedMany } from 'ai'

// Single embedding
const { embedding } = await embed({
  model: openai.embedding('text-embedding-3-small'),
  value: 'The quick brown fox',
})

// Multiple embeddings
const { embeddings } = await embedMany({
  model: openai.embedding('text-embedding-3-small'),
  values: ['First text', 'Second text', 'Third text'],
})
```

## RAG Pattern

```typescript
import { openai } from '@ai-sdk/openai'
import { generateText, embed } from 'ai'

async function ragQuery(query: string) {
  // 1. Generate embedding for query
  const { embedding } = await embed({
    model: openai.embedding('text-embedding-3-small'),
    value: query,
  })

  // 2. Search vector database
  const relevantDocs = await vectorDb.search({
    vector: embedding,
    limit: 5,
  })

  // 3. Generate response with context
  const { text } = await generateText({
    model: openai('gpt-4o'),
    system: `Answer based on the following context:
    
${relevantDocs.map(d => d.content).join('\n\n')}`,
    prompt: query,
  })

  return text
}
```

## Error Handling

```typescript
import { APICallError, RetryError } from 'ai'

try {
  const result = await generateText({
    model: openai('gpt-4o'),
    prompt: 'Hello',
  })
} catch (error) {
  if (error instanceof APICallError) {
    console.error('API Error:', error.message)
    console.error('Status:', error.statusCode)
    console.error('Response:', error.responseBody)
  } else if (error instanceof RetryError) {
    console.error('Retry failed:', error.lastError)
  }
}
```

## Best Practices

1. **Stream responses** - Better UX for long generations
2. **Use structured output** - Type-safe AI responses
3. **Implement tool calling** - Extend LLM capabilities
4. **Handle errors gracefully** - Retry and fallback strategies
5. **Use appropriate models** - Match model to task complexity
6. **Set reasonable maxTokens** - Control costs and response length
7. **Cache embeddings** - Avoid redundant API calls
