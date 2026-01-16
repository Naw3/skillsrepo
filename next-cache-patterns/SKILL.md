---
name: next-cache-patterns
description: |
  Next.js caching and data fetching patterns including cacheLife, PPR,
  revalidation strategies, and cache invalidation. Use when optimizing
  data fetching, implementing ISR, or configuring cache behavior.
---

# Next.js Cache Patterns

Comprehensive caching strategies for Next.js App Router.

## Caching Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    CACHING LAYERS                           │
├─────────────────────────────────────────────────────────────┤
│  Request Memoization  │ Dedupe identical requests in render │
│  Data Cache           │ Store fetch results across requests │
│  Full Route Cache     │ Pre-render static routes at build   │
│  Router Cache         │ Client-side route segment cache     │
└─────────────────────────────────────────────────────────────┘
```

## Fetch Caching

### Default Behavior

```typescript
// ✅ Cached by default (static)
const data = await fetch('https://api.example.com/data')

// ❌ Not cached (dynamic)
const data = await fetch('https://api.example.com/data', {
  cache: 'no-store'
})

// Time-based revalidation
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 3600 } // Revalidate every hour
})

// Tag-based revalidation
const data = await fetch('https://api.example.com/posts', {
  next: { tags: ['posts'] }
})
```

## cacheLife (Next.js 15+)

```typescript
// next.config.ts
const nextConfig = {
  experimental: {
    dynamicIO: true,
  },
}

// Define cache profiles
import { cacheLife } from 'next/cache'

// Built-in profiles
export async function getStaticData() {
  'use cache'
  cacheLife('hours')  // Cache for hours
  return fetch('/api/data')
}

// Custom profile
export async function getUserData(userId: string) {
  'use cache'
  cacheLife({
    stale: 300,      // 5 minutes stale
    revalidate: 600, // 10 minutes revalidate
    expire: 3600,    // 1 hour expire
  })
  return fetch(`/api/users/${userId}`)
}
```

### Cache Profiles

```typescript
// next.config.ts
const nextConfig = {
  experimental: {
    cacheLife: {
      // Custom profiles
      frequent: {
        stale: 60,
        revalidate: 120,
        expire: 300,
      },
      rare: {
        stale: 3600,
        revalidate: 86400,
        expire: 604800,
      },
    },
  },
}

// Usage
cacheLife('frequent')
cacheLife('rare')
```

## Partial Prerendering (PPR)

```typescript
// next.config.ts
const nextConfig = {
  experimental: {
    ppr: 'incremental',
  },
}

// Enable PPR for specific routes
// app/dashboard/page.tsx
export const experimental_ppr = true

export default async function Dashboard() {
  return (
    <div>
      {/* Static shell - prerendered */}
      <header>Dashboard</header>
      <nav>Static navigation</nav>
      
      {/* Dynamic content - streamed */}
      <Suspense fallback={<DashboardSkeleton />}>
        <DynamicDashboard />
      </Suspense>
    </div>
  )
}

async function DynamicDashboard() {
  const data = await fetch('/api/dashboard', { cache: 'no-store' })
  return <DashboardContent data={data} />
}
```

## unstable_cache (Function Caching)

```typescript
import { unstable_cache } from 'next/cache'

const getCachedUser = unstable_cache(
  async (userId: string) => {
    const user = await db.user.findUnique({ where: { id: userId } })
    return user
  },
  ['user'], // Cache key prefix
  {
    revalidate: 3600,        // 1 hour
    tags: ['users'],         // For revalidation
  }
)

// Usage
const user = await getCachedUser('user-123')
```

## Revalidation Strategies

### Time-based (ISR)

```typescript
// app/posts/page.tsx
export const revalidate = 60 // Revalidate every 60 seconds

export default async function Posts() {
  const posts = await getPosts()
  return <PostList posts={posts} />
}
```

### On-demand (revalidatePath/revalidateTag)

```typescript
// app/actions.ts
'use server'

import { revalidatePath, revalidateTag } from 'next/cache'

export async function createPost(data: PostData) {
  await db.post.create({ data })
  
  // Revalidate specific path
  revalidatePath('/posts')
  
  // Revalidate by tag
  revalidateTag('posts')
}

// Revalidate with layout
revalidatePath('/posts', 'layout')

// Revalidate nested routes
revalidatePath('/posts/[slug]', 'page')
```

### API Route Revalidation

```typescript
// app/api/revalidate/route.ts
import { revalidateTag } from 'next/cache'
import { NextRequest, NextResponse } from 'next/server'

export async function POST(request: NextRequest) {
  const { tag, secret } = await request.json()
  
  // Verify secret
  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: 'Invalid secret' }, { status: 401 })
  }
  
  revalidateTag(tag)
  
  return NextResponse.json({ revalidated: true, tag })
}
```

## Dynamic Rendering

```typescript
// Force dynamic rendering
export const dynamic = 'force-dynamic'

// Force static rendering
export const dynamic = 'force-static'

// Auto (default)
export const dynamic = 'auto'

// Error on dynamic
export const dynamic = 'error'
```

### Dynamic Functions

```typescript
import { cookies, headers } from 'next/headers'
import { connection } from 'next/server'

export default async function Page() {
  // These make the route dynamic
  const cookieStore = await cookies()
  const headersList = await headers()
  
  // Or explicitly opt into dynamic
  await connection()
  
  return <Content />
}
```

## generateStaticParams

```typescript
// app/posts/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await getPosts()
  
  return posts.map((post) => ({
    slug: post.slug,
  }))
}

// Control behavior for paths not generated
export const dynamicParams = true // Allow dynamic (default)
export const dynamicParams = false // Return 404
```

## Route Segment Config

```typescript
// Per-route configuration
export const revalidate = 3600      // ISR interval
export const dynamic = 'auto'       // Rendering mode
export const dynamicParams = true   // Allow dynamic params
export const fetchCache = 'auto'    // Fetch cache behavior
export const runtime = 'nodejs'     // Runtime
export const preferredRegion = 'auto'
```

## Caching with Database

```typescript
import { unstable_cache } from 'next/cache'

// Cache database queries
const getProducts = unstable_cache(
  async (category: string) => {
    return db.product.findMany({
      where: { category },
      orderBy: { createdAt: 'desc' },
    })
  },
  ['products'],
  { 
    revalidate: 300,
    tags: ['products'] 
  }
)

// Cache with user-specific data
const getUserOrders = unstable_cache(
  async (userId: string) => {
    return db.order.findMany({
      where: { userId },
      include: { items: true },
    })
  },
  ['orders'],
  {
    revalidate: 60,
    tags: ['orders', `user-${userId}-orders`],
  }
)
```

## Cache Debugging

```typescript
// Check cache status in response headers
// x-nextjs-cache: HIT | MISS | STALE

// Development logging
// next.config.ts
const nextConfig = {
  logging: {
    fetches: {
      fullUrl: true,
    },
  },
}
```

## Best Practices

1. **Default to caching** - Opt-out when needed
2. **Use tags for granular invalidation** - More control than paths
3. **Combine ISR with on-demand** - Background refresh + instant updates
4. **Cache at the right layer** - Function vs fetch vs route
5. **Use PPR for mixed content** - Static shell + dynamic content
6. **Monitor cache hit rates** - Optimize cache durations
7. **Invalidate conservatively** - Don't over-invalidate

## Anti-patterns

```typescript
// ❌ Fetching in client components when data could be server-cached
'use client'
useEffect(() => {
  fetch('/api/data').then(...)
}, [])

// ✅ Fetch in Server Component with caching
async function Page() {
  const data = await fetch('/api/data', { next: { revalidate: 60 } })
  return <ClientComponent data={data} />
}

// ❌ Over-invalidating
revalidatePath('/') // Invalidates entire app

// ✅ Granular invalidation
revalidateTag('posts')
```
