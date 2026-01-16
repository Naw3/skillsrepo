---
name: react-19-patterns
description: |
  React 19 modern patterns including the use() hook, Server Components, Server Actions,
  React Compiler, and new APIs. Use when implementing features with React 19+,
  optimizing performance, or migrating from React 18.
---

# React 19 Patterns

Essential patterns for React 19's new features and APIs.

## What's New in React 19

| Feature | Description |
|---------|-------------|
| `use()` hook | Read promises and context in render |
| Server Actions | Mutations with `"use server"` |
| React Compiler | Automatic memoization |
| `useActionState` | Form action state management |
| `useFormStatus` | Pending state in forms |
| `useOptimistic` | Optimistic UI updates |
| Ref as prop | No more `forwardRef` needed |

## Patterns

### 1. The `use()` Hook

Read promises and context directly in components:

```typescript
import { use, Suspense } from 'react'

// Reading a promise
async function getData() {
  const res = await fetch('/api/data')
  return res.json()
}

function DataDisplay({ dataPromise }: { dataPromise: Promise<Data> }) {
  const data = use(dataPromise) // Suspends until resolved
  return <div>{data.title}</div>
}

// Usage with Suspense
function Page() {
  const dataPromise = getData() // Start fetching
  
  return (
    <Suspense fallback={<Loading />}>
      <DataDisplay dataPromise={dataPromise} />
    </Suspense>
  )
}
```

**Reading context conditionally:**

```typescript
function Component({ showUser }: { showUser: boolean }) {
  // ✅ React 19: Can call conditionally
  if (showUser) {
    const user = use(UserContext)
    return <div>{user.name}</div>
  }
  return <div>Guest</div>
}
```

### 2. Server Actions

```typescript
// app/actions.ts
'use server'

import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string
  const content = formData.get('content') as string

  // Validate
  if (!title || title.length < 3) {
    return { error: 'Title must be at least 3 characters' }
  }

  // Create in database
  await db.post.create({ data: { title, content } })

  // Revalidate and redirect
  revalidatePath('/posts')
  redirect('/posts')
}

// app/posts/new/page.tsx
import { createPost } from '@/app/actions'

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Title" required />
      <textarea name="content" placeholder="Content" />
      <button type="submit">Create Post</button>
    </form>
  )
}
```

### 3. useActionState (Form State)

```typescript
'use client'

import { useActionState } from 'react'
import { createPost } from '@/app/actions'

type State = { error?: string; success?: boolean }

export function PostForm() {
  const [state, formAction, isPending] = useActionState<State, FormData>(
    async (prevState, formData) => {
      const result = await createPost(formData)
      if (result?.error) return { error: result.error }
      return { success: true }
    },
    { error: undefined, success: false }
  )

  return (
    <form action={formAction}>
      <input name="title" placeholder="Title" disabled={isPending} />
      <textarea name="content" placeholder="Content" disabled={isPending} />
      
      {state.error && (
        <p className="text-red-500">{state.error}</p>
      )}
      
      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Post'}
      </button>
    </form>
  )
}
```

### 4. useFormStatus

```typescript
'use client'

import { useFormStatus } from 'react-dom'

function SubmitButton() {
  const { pending, data, method, action } = useFormStatus()

  return (
    <button type="submit" disabled={pending}>
      {pending ? (
        <>
          <Spinner className="animate-spin" />
          Submitting...
        </>
      ) : (
        'Submit'
      )}
    </button>
  )
}

// Usage - must be inside a <form>
function ContactForm() {
  return (
    <form action={submitContact}>
      <input name="email" type="email" />
      <textarea name="message" />
      <SubmitButton /> {/* Accesses parent form status */}
    </form>
  )
}
```

### 5. useOptimistic

```typescript
'use client'

import { useOptimistic, useTransition } from 'react'
import { likePost } from '@/app/actions'

interface Post {
  id: string
  likes: number
  isLiked: boolean
}

function LikeButton({ post }: { post: Post }) {
  const [isPending, startTransition] = useTransition()
  const [optimisticPost, addOptimistic] = useOptimistic(
    post,
    (state, newLiked: boolean) => ({
      ...state,
      likes: newLiked ? state.likes + 1 : state.likes - 1,
      isLiked: newLiked,
    })
  )

  const handleClick = () => {
    startTransition(async () => {
      addOptimistic(!optimisticPost.isLiked)
      await likePost(post.id, !optimisticPost.isLiked)
    })
  }

  return (
    <button onClick={handleClick} disabled={isPending}>
      {optimisticPost.isLiked ? '❤️' : '🤍'} {optimisticPost.likes}
    </button>
  )
}
```

### 6. Ref as Prop (No forwardRef)

```typescript
// ✅ React 19: refs are regular props
function Input({ ref, ...props }: { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />
}

// Usage
function Form() {
  const inputRef = useRef<HTMLInputElement>(null)
  
  return (
    <form>
      <Input ref={inputRef} placeholder="Name" />
      <button onClick={() => inputRef.current?.focus()}>
        Focus
      </button>
    </form>
  )
}

// ❌ React 18: Required forwardRef
const Input = forwardRef<HTMLInputElement, InputProps>((props, ref) => {
  return <input ref={ref} {...props} />
})
```

### 7. React Compiler (Auto-Memoization)

```typescript
// next.config.ts
const nextConfig = {
  reactCompiler: true, // Enable React Compiler
}

// Before: Manual memoization
const MemoizedComponent = memo(function Component({ data }) {
  const processed = useMemo(() => heavyComputation(data), [data])
  const handler = useCallback(() => doSomething(processed), [processed])
  return <Child data={processed} onClick={handler} />
})

// After: React Compiler handles it automatically
function Component({ data }) {
  const processed = heavyComputation(data)
  const handler = () => doSomething(processed)
  return <Child data={processed} onClick={handler} />
}
```

### 8. Document Metadata in Components

```typescript
// React 19: Metadata hoisted to <head> automatically
function BlogPost({ post }) {
  return (
    <article>
      <title>{post.title}</title>
      <meta name="description" content={post.excerpt} />
      <link rel="canonical" href={`https://blog.com/${post.slug}`} />
      
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  )
}
```

### 9. Asset Loading APIs

```typescript
import { preload, preconnect, prefetchDNS, preinit } from 'react-dom'

function App() {
  // Preload critical resources
  preload('/fonts/inter.woff2', { as: 'font', type: 'font/woff2' })
  preload('/hero.jpg', { as: 'image' })
  
  // Preconnect to external domains
  preconnect('https://api.example.com')
  
  // DNS prefetch for less critical domains
  prefetchDNS('https://analytics.example.com')
  
  // Preinit scripts
  preinit('/critical.js', { as: 'script' })

  return <MainContent />
}
```

### 10. Error Handling in Actions

```typescript
'use server'

import { z } from 'zod'

const PostSchema = z.object({
  title: z.string().min(3).max(100),
  content: z.string().min(10),
})

export async function createPost(formData: FormData) {
  const rawData = {
    title: formData.get('title'),
    content: formData.get('content'),
  }

  // Validate with Zod
  const result = PostSchema.safeParse(rawData)
  
  if (!result.success) {
    return {
      errors: result.error.flatten().fieldErrors,
    }
  }

  try {
    await db.post.create({ data: result.data })
    return { success: true }
  } catch (error) {
    return { error: 'Failed to create post' }
  }
}
```

## Server vs Client Components

```
┌─────────────────────────────────────────────────────────┐
│                    SERVER COMPONENTS                     │
│  • Data fetching           • No useState/useEffect      │
│  • Access secrets          • No browser APIs            │
│  • Direct DB access        • No event handlers          │
│  • Smaller bundles         • Can render Client children │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    CLIENT COMPONENTS                     │
│  • Interactivity           • 'use client' directive     │
│  • useState/useEffect      • Browser APIs               │
│  • Event handlers          • Third-party libs           │
│  • Can't render Server     • Adds to JS bundle          │
└─────────────────────────────────────────────────────────┘
```

### Composition Pattern

```typescript
// Server Component (default)
async function Dashboard() {
  const data = await fetchDashboardData() // Server-side fetch
  
  return (
    <div>
      <DashboardHeader user={data.user} />
      {/* Client component for interactivity */}
      <InteractiveChart data={data.metrics} />
      <RecentActivity items={data.activity} />
    </div>
  )
}

// Client Component
'use client'

function InteractiveChart({ data }) {
  const [filter, setFilter] = useState('all')
  // Interactive logic...
}
```

## Best Practices

### Do's

- ✅ Start with Server Components, add `'use client'` only when needed
- ✅ Use `use()` for reading promises in render
- ✅ Use Server Actions for mutations
- ✅ Let React Compiler handle memoization
- ✅ Use `useOptimistic` for instant UI feedback

### Don'ts

- ❌ Don't use `useState`/`useEffect` for data fetching (use RSC)
- ❌ Don't pass functions from Server → Client components
- ❌ Don't use `memo`/`useMemo`/`useCallback` with React Compiler enabled
- ❌ Don't call Server Actions in loops (batch instead)

## Migration from React 18

```typescript
// Before (React 18)
'use client'
import { useState, useEffect } from 'react'

function UserProfile({ userId }) {
  const [user, setUser] = useState(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        setUser(data)
        setLoading(false)
      })
  }, [userId])

  if (loading) return <Skeleton />
  return <Profile user={user} />
}

// After (React 19) - Server Component
async function UserProfile({ userId }) {
  const user = await fetch(`/api/users/${userId}`).then(r => r.json())
  return <Profile user={user} />
}
```
