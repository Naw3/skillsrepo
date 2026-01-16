---
name: api-rate-limiting
description: |
  Patterns for robust API integration including retry logic, circuit breakers,
  request deduplication, and rate limiting. Use when integrating external APIs,
  handling failures gracefully, or optimizing API call efficiency.
---

# API Rate Limiting & Resilience Patterns

Patterns for building robust API integrations with proper error handling.

## Core Patterns

### 1. Fetch with Retry

```typescript
interface RetryOptions {
  maxRetries?: number
  baseDelay?: number
  maxDelay?: number
  retryOn?: (error: Error, response?: Response) => boolean
}

export async function fetchWithRetry(
  url: string,
  options: RequestInit = {},
  retryOptions: RetryOptions = {}
): Promise<Response> {
  const {
    maxRetries = 3,
    baseDelay = 1000,
    maxDelay = 10000,
    retryOn = defaultRetryCondition,
  } = retryOptions

  let lastError: Error | null = null

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, {
        ...options,
        signal: AbortSignal.timeout(30000), // 30s timeout
      })

      // Check if we should retry based on response
      if (!response.ok && retryOn(new Error(`HTTP ${response.status}`), response)) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`)
      }

      return response
    } catch (error) {
      lastError = error as Error

      if (attempt === maxRetries || !retryOn(lastError)) {
        throw lastError
      }

      // Exponential backoff with jitter
      const delay = Math.min(
        baseDelay * Math.pow(2, attempt) + Math.random() * 1000,
        maxDelay
      )

      console.log(`Retry ${attempt + 1}/${maxRetries} after ${delay}ms`)
      await sleep(delay)
    }
  }

  throw lastError
}

function defaultRetryCondition(error: Error, response?: Response): boolean {
  // Retry on network errors
  if (error.name === 'TypeError' || error.name === 'AbortError') {
    return true
  }

  // Retry on 5xx and 429 (rate limit)
  if (response) {
    return response.status >= 500 || response.status === 429
  }

  return false
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms))
}
```

### 2. Circuit Breaker

```typescript
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN'

interface CircuitBreakerOptions {
  failureThreshold?: number
  successThreshold?: number
  timeout?: number
}

class CircuitBreaker {
  private state: CircuitState = 'CLOSED'
  private failures = 0
  private successes = 0
  private lastFailureTime = 0
  
  private readonly failureThreshold: number
  private readonly successThreshold: number
  private readonly timeout: number

  constructor(options: CircuitBreakerOptions = {}) {
    this.failureThreshold = options.failureThreshold ?? 5
    this.successThreshold = options.successThreshold ?? 2
    this.timeout = options.timeout ?? 30000 // 30 seconds
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime >= this.timeout) {
        this.state = 'HALF_OPEN'
        this.successes = 0
      } else {
        throw new Error('Circuit breaker is OPEN')
      }
    }

    try {
      const result = await fn()
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      throw error
    }
  }

  private onSuccess(): void {
    this.failures = 0
    
    if (this.state === 'HALF_OPEN') {
      this.successes++
      if (this.successes >= this.successThreshold) {
        this.state = 'CLOSED'
      }
    }
  }

  private onFailure(): void {
    this.failures++
    this.lastFailureTime = Date.now()

    if (this.failures >= this.failureThreshold) {
      this.state = 'OPEN'
    }
  }

  getState(): CircuitState {
    return this.state
  }
}

// Usage
const apiCircuit = new CircuitBreaker({ failureThreshold: 3 })

async function callExternalAPI() {
  return apiCircuit.execute(async () => {
    const response = await fetch('https://api.example.com/data')
    if (!response.ok) throw new Error('API failed')
    return response.json()
  })
}
```

### 3. Request Deduplication

```typescript
interface CacheEntry<T> {
  data: T
  timestamp: number
}

class RequestDeduplicator<T> {
  private cache = new Map<string, CacheEntry<T>>()
  private pending = new Map<string, Promise<T>>()
  private readonly ttl: number

  constructor(ttlMs: number = 60000) {
    this.ttl = ttlMs
  }

  async dedupe(key: string, fn: () => Promise<T>): Promise<T> {
    // Check cache first
    const cached = this.cache.get(key)
    if (cached && Date.now() - cached.timestamp < this.ttl) {
      return cached.data
    }

    // Check if request is already in flight
    const pending = this.pending.get(key)
    if (pending) {
      return pending
    }

    // Make new request
    const promise = fn()
      .then(data => {
        this.cache.set(key, { data, timestamp: Date.now() })
        return data
      })
      .finally(() => {
        this.pending.delete(key)
      })

    this.pending.set(key, promise)
    return promise
  }

  invalidate(key: string): void {
    this.cache.delete(key)
  }

  clear(): void {
    this.cache.clear()
    this.pending.clear()
  }
}

// Usage
const deduplicator = new RequestDeduplicator<UserData>(30000)

async function getUser(id: string) {
  return deduplicator.dedupe(`user:${id}`, async () => {
    const response = await fetch(`/api/users/${id}`)
    return response.json()
  })
}
```

### 4. Rate Limiter (Token Bucket)

```typescript
class RateLimiter {
  private tokens: number
  private lastRefill: number
  
  constructor(
    private readonly maxTokens: number,
    private readonly refillRate: number, // tokens per second
  ) {
    this.tokens = maxTokens
    this.lastRefill = Date.now()
  }

  private refill(): void {
    const now = Date.now()
    const elapsed = (now - this.lastRefill) / 1000
    this.tokens = Math.min(this.maxTokens, this.tokens + elapsed * this.refillRate)
    this.lastRefill = now
  }

  async acquire(tokens: number = 1): Promise<boolean> {
    this.refill()
    
    if (this.tokens >= tokens) {
      this.tokens -= tokens
      return true
    }

    return false
  }

  async waitForToken(tokens: number = 1): Promise<void> {
    while (!(await this.acquire(tokens))) {
      const waitTime = ((tokens - this.tokens) / this.refillRate) * 1000
      await new Promise(resolve => setTimeout(resolve, Math.max(100, waitTime)))
    }
  }
}

// Usage: 10 requests per second
const limiter = new RateLimiter(10, 10)

async function rateLimitedFetch(url: string) {
  await limiter.waitForToken()
  return fetch(url)
}
```

### 5. Batch Request Queue

```typescript
interface QueuedRequest<T> {
  id: string
  fn: () => Promise<T>
  resolve: (value: T) => void
  reject: (error: Error) => void
}

class BatchQueue<T> {
  private queue: QueuedRequest<T>[] = []
  private processing = false
  
  constructor(
    private readonly batchSize: number = 5,
    private readonly delayBetweenBatches: number = 1000
  ) {}

  async add(id: string, fn: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push({ id, fn, resolve, reject })
      this.process()
    })
  }

  private async process(): Promise<void> {
    if (this.processing || this.queue.length === 0) return
    this.processing = true

    while (this.queue.length > 0) {
      const batch = this.queue.splice(0, this.batchSize)
      
      await Promise.all(
        batch.map(async ({ fn, resolve, reject }) => {
          try {
            const result = await fn()
            resolve(result)
          } catch (error) {
            reject(error as Error)
          }
        })
      )

      if (this.queue.length > 0) {
        await new Promise(r => setTimeout(r, this.delayBetweenBatches))
      }
    }

    this.processing = false
  }
}
```

### 6. API Client with All Patterns

```typescript
interface APIClientOptions {
  baseUrl: string
  maxRetries?: number
  rateLimit?: { requests: number; perSeconds: number }
  circuitBreaker?: CircuitBreakerOptions
  cacheTtl?: number
}

class ResilientAPIClient {
  private readonly circuit: CircuitBreaker
  private readonly limiter: RateLimiter
  private readonly deduplicator: RequestDeduplicator<unknown>
  private readonly baseUrl: string
  private readonly maxRetries: number

  constructor(options: APIClientOptions) {
    this.baseUrl = options.baseUrl
    this.maxRetries = options.maxRetries ?? 3
    this.circuit = new CircuitBreaker(options.circuitBreaker)
    this.limiter = new RateLimiter(
      options.rateLimit?.requests ?? 10,
      options.rateLimit?.requests ?? 10 / (options.rateLimit?.perSeconds ?? 1)
    )
    this.deduplicator = new RequestDeduplicator(options.cacheTtl ?? 30000)
  }

  async get<T>(path: string, options?: { cache?: boolean }): Promise<T> {
    const url = `${this.baseUrl}${path}`
    const cacheKey = `GET:${url}`

    const fetchFn = async (): Promise<T> => {
      await this.limiter.waitForToken()
      
      return this.circuit.execute(async () => {
        const response = await fetchWithRetry(url, {
          method: 'GET',
          headers: { 'Content-Type': 'application/json' },
        }, { maxRetries: this.maxRetries })

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`)
        }

        return response.json()
      })
    }

    if (options?.cache !== false) {
      return this.deduplicator.dedupe(cacheKey, fetchFn) as Promise<T>
    }

    return fetchFn()
  }

  async post<T>(path: string, body: unknown): Promise<T> {
    const url = `${this.baseUrl}${path}`

    await this.limiter.waitForToken()

    return this.circuit.execute(async () => {
      const response = await fetchWithRetry(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body),
      }, { maxRetries: this.maxRetries })

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`)
      }

      return response.json()
    })
  }

  getCircuitState() {
    return this.circuit.getState()
  }
}

// Usage
const api = new ResilientAPIClient({
  baseUrl: 'https://api.example.com',
  maxRetries: 3,
  rateLimit: { requests: 10, perSeconds: 1 },
  cacheTtl: 60000,
})

const data = await api.get('/users/123')
```

## Error Types

```typescript
class APIError extends Error {
  constructor(
    message: string,
    public readonly status?: number,
    public readonly code?: string,
    public readonly retryable: boolean = false
  ) {
    super(message)
    this.name = 'APIError'
  }
}

class RateLimitError extends APIError {
  constructor(
    public readonly retryAfter: number
  ) {
    super('Rate limit exceeded', 429, 'RATE_LIMITED', true)
    this.name = 'RateLimitError'
  }
}

class CircuitOpenError extends APIError {
  constructor() {
    super('Service temporarily unavailable', 503, 'CIRCUIT_OPEN', true)
    this.name = 'CircuitOpenError'
  }
}
```

## Best Practices

1. **Always set timeouts** - Use `AbortSignal.timeout()` or custom timeouts
2. **Log failures** - Track retry attempts and circuit breaker state
3. **Graceful degradation** - Return cached/stale data when possible
4. **Backoff with jitter** - Prevent thundering herd
5. **Monitor metrics** - Track success rates, latencies, circuit state
