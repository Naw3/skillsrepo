---
name: error-handling
description: |
  Error handling patterns for production applications including
  error boundaries, API errors, logging, and user-friendly messages.
  Use when implementing error handling, creating error boundaries,
  or building robust error recovery flows.
---

# Error Handling Patterns

Comprehensive error handling patterns for production applications.

## Error Classes

### Base Error Class

```typescript
export class AppError extends Error {
  public readonly isOperational: boolean
  public readonly statusCode: number
  public readonly code: string
  public readonly timestamp: Date

  constructor(
    message: string,
    statusCode: number = 500,
    code: string = 'INTERNAL_ERROR',
    isOperational: boolean = true
  ) {
    super(message)
    this.name = this.constructor.name
    this.statusCode = statusCode
    this.code = code
    this.isOperational = isOperational
    this.timestamp = new Date()

    Error.captureStackTrace(this, this.constructor)
  }

  toJSON() {
    return {
      name: this.name,
      message: this.message,
      code: this.code,
      statusCode: this.statusCode,
      timestamp: this.timestamp.toISOString(),
    }
  }
}
```

### Specific Error Types

```typescript
export class ValidationError extends AppError {
  public readonly fields: Record<string, string[]>

  constructor(message: string, fields: Record<string, string[]> = {}) {
    super(message, 400, 'VALIDATION_ERROR')
    this.fields = fields
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id?: string) {
    const message = id 
      ? `${resource} with id '${id}' not found`
      : `${resource} not found`
    super(message, 404, 'NOT_FOUND')
  }
}

export class UnauthorizedError extends AppError {
  constructor(message: string = 'Authentication required') {
    super(message, 401, 'UNAUTHORIZED')
  }
}

export class ForbiddenError extends AppError {
  constructor(message: string = 'Access denied') {
    super(message, 403, 'FORBIDDEN')
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409, 'CONFLICT')
  }
}

export class RateLimitError extends AppError {
  public readonly retryAfter: number

  constructor(retryAfter: number = 60) {
    super('Too many requests', 429, 'RATE_LIMITED')
    this.retryAfter = retryAfter
  }
}

export class ExternalServiceError extends AppError {
  public readonly service: string

  constructor(service: string, originalError?: Error) {
    super(`${service} service unavailable`, 503, 'EXTERNAL_SERVICE_ERROR')
    this.service = service
    if (originalError) {
      this.cause = originalError
    }
  }
}
```

## React Error Boundaries

### Basic Error Boundary

```typescript
'use client'

import { Component, ReactNode } from 'react'

interface Props {
  children: ReactNode
  fallback?: ReactNode
  onError?: (error: Error, errorInfo: React.ErrorInfo) => void
}

interface State {
  hasError: boolean
  error: Error | null
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props)
    this.state = { hasError: false, error: null }
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error boundary caught:', error, errorInfo)
    this.props.onError?.(error, errorInfo)
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <div className="p-6 text-center">
          <h2 className="text-xl font-bold text-red-500">Something went wrong</h2>
          <button
            onClick={() => this.setState({ hasError: false, error: null })}
            className="mt-4 px-4 py-2 bg-primary-500 text-white rounded"
          >
            Try again
          </button>
        </div>
      )
    }

    return this.props.children
  }
}
```

### Error Boundary with Reset

```typescript
'use client'

import { useEffect, useState } from 'react'

interface ErrorFallbackProps {
  error: Error
  reset: () => void
}

function ErrorFallback({ error, reset }: ErrorFallbackProps) {
  return (
    <div className="min-h-[400px] flex items-center justify-center">
      <div className="text-center p-6 max-w-md">
        <div className="w-16 h-16 mx-auto mb-4 rounded-full bg-red-500/10 flex items-center justify-center">
          <svg className="w-8 h-8 text-red-500" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
          </svg>
        </div>
        <h2 className="text-xl font-bold mb-2">Something went wrong</h2>
        <p className="text-gray-500 mb-4">{error.message}</p>
        <button
          onClick={reset}
          className="px-4 py-2 bg-primary-500 text-white rounded-lg hover:bg-primary-600 transition"
        >
          Try again
        </button>
      </div>
    </div>
  )
}

// Hook-based error boundary wrapper
export function withErrorBoundary<P extends object>(
  Component: React.ComponentType<P>,
  fallback?: React.ComponentType<ErrorFallbackProps>
) {
  return function WrappedComponent(props: P) {
    const [error, setError] = useState<Error | null>(null)

    useEffect(() => {
      const handleError = (event: ErrorEvent) => {
        setError(event.error)
      }
      window.addEventListener('error', handleError)
      return () => window.removeEventListener('error', handleError)
    }, [])

    if (error) {
      const Fallback = fallback ?? ErrorFallback
      return <Fallback error={error} reset={() => setError(null)} />
    }

    return <Component {...props} />
  }
}
```

## Next.js Error Files

### Global Error (app/error.tsx)

```typescript
'use client'

import { useEffect } from 'react'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  useEffect(() => {
    // Log to error reporting service
    console.error('Page error:', error)
  }, [error])

  return (
    <div className="min-h-screen flex items-center justify-center">
      <div className="text-center">
        <h1 className="text-4xl font-bold mb-4">Something went wrong</h1>
        <p className="text-gray-500 mb-6">
          {error.message || 'An unexpected error occurred'}
        </p>
        <div className="space-x-4">
          <button
            onClick={reset}
            className="px-6 py-3 bg-primary-500 text-white rounded-lg"
          >
            Try again
          </button>
          <a href="/" className="px-6 py-3 border rounded-lg">
            Go home
          </a>
        </div>
      </div>
    </div>
  )
}
```

### Global Error (app/global-error.tsx)

```typescript
'use client'

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <html>
      <body>
        <div className="min-h-screen flex items-center justify-center bg-gray-900 text-white">
          <div className="text-center">
            <h1 className="text-4xl font-bold mb-4">Critical Error</h1>
            <p className="text-gray-400 mb-6">
              The application encountered a critical error.
            </p>
            <button
              onClick={reset}
              className="px-6 py-3 bg-white text-gray-900 rounded-lg"
            >
              Reload Application
            </button>
          </div>
        </div>
      </body>
    </html>
  )
}
```

### Not Found (app/not-found.tsx)

```typescript
import Link from 'next/link'

export default function NotFound() {
  return (
    <div className="min-h-screen flex items-center justify-center">
      <div className="text-center">
        <h1 className="text-9xl font-bold text-gray-200">404</h1>
        <h2 className="text-2xl font-bold mt-4 mb-2">Page not found</h2>
        <p className="text-gray-500 mb-6">
          The page you're looking for doesn't exist or has been moved.
        </p>
        <Link
          href="/"
          className="px-6 py-3 bg-primary-500 text-white rounded-lg inline-block"
        >
          Go back home
        </Link>
      </div>
    </div>
  )
}
```

## API Error Handling

### Route Handler Pattern

```typescript
// app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { AppError, NotFoundError, ValidationError } from '@/lib/errors'

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params
    
    if (!id || !isValidUUID(id)) {
      throw new ValidationError('Invalid user ID format')
    }

    const user = await db.user.findUnique({ where: { id } })
    
    if (!user) {
      throw new NotFoundError('User', id)
    }

    return NextResponse.json(user)
  } catch (error) {
    return handleAPIError(error)
  }
}

function handleAPIError(error: unknown): NextResponse {
  console.error('API Error:', error)

  if (error instanceof AppError) {
    return NextResponse.json(
      {
        error: {
          code: error.code,
          message: error.message,
          ...(error instanceof ValidationError && { fields: error.fields }),
        },
      },
      { status: error.statusCode }
    )
  }

  // Unknown error
  return NextResponse.json(
    {
      error: {
        code: 'INTERNAL_ERROR',
        message: 'An unexpected error occurred',
      },
    },
    { status: 500 }
  )
}
```

### Server Action Error Handling

```typescript
'use server'

import { ValidationError } from '@/lib/errors'
import { z } from 'zod'

const CreateUserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
})

type ActionResult<T> = 
  | { success: true; data: T }
  | { success: false; error: string; fields?: Record<string, string[]> }

export async function createUser(formData: FormData): Promise<ActionResult<User>> {
  try {
    const rawData = {
      name: formData.get('name'),
      email: formData.get('email'),
    }

    const result = CreateUserSchema.safeParse(rawData)
    
    if (!result.success) {
      return {
        success: false,
        error: 'Validation failed',
        fields: result.error.flatten().fieldErrors,
      }
    }

    const user = await db.user.create({ data: result.data })
    
    return { success: true, data: user }
  } catch (error) {
    console.error('Create user error:', error)
    
    return {
      success: false,
      error: error instanceof Error ? error.message : 'Failed to create user',
    }
  }
}
```

## Error Logging

```typescript
interface ErrorLog {
  timestamp: string
  level: 'error' | 'warn' | 'info'
  message: string
  code?: string
  stack?: string
  context?: Record<string, unknown>
  userId?: string
  requestId?: string
}

class Logger {
  private static instance: Logger

  static getInstance(): Logger {
    if (!Logger.instance) {
      Logger.instance = new Logger()
    }
    return Logger.instance
  }

  error(error: Error | AppError, context?: Record<string, unknown>) {
    const log: ErrorLog = {
      timestamp: new Date().toISOString(),
      level: 'error',
      message: error.message,
      stack: error.stack,
      context,
    }

    if (error instanceof AppError) {
      log.code = error.code
    }

    // In development, log to console
    if (process.env.NODE_ENV === 'development') {
      console.error(log)
    }

    // In production, send to logging service
    if (process.env.NODE_ENV === 'production') {
      this.sendToLoggingService(log)
    }
  }

  private async sendToLoggingService(log: ErrorLog) {
    // Send to Sentry, LogRocket, DataDog, etc.
    try {
      await fetch('/api/logs', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(log),
      })
    } catch {
      // Fail silently - don't throw errors in error handling
      console.error('Failed to send log:', log)
    }
  }
}

export const logger = Logger.getInstance()
```

## User-Friendly Messages

```typescript
const ERROR_MESSAGES: Record<string, string> = {
  VALIDATION_ERROR: 'Please check your input and try again.',
  NOT_FOUND: 'The resource you requested could not be found.',
  UNAUTHORIZED: 'Please sign in to continue.',
  FORBIDDEN: 'You don\'t have permission to access this.',
  RATE_LIMITED: 'Too many requests. Please wait a moment.',
  NETWORK_ERROR: 'Unable to connect. Check your internet connection.',
  INTERNAL_ERROR: 'Something went wrong. Please try again later.',
}

export function getUserMessage(error: unknown): string {
  if (error instanceof AppError) {
    return ERROR_MESSAGES[error.code] ?? error.message
  }
  
  if (error instanceof Error) {
    if (error.name === 'TypeError' && error.message.includes('fetch')) {
      return ERROR_MESSAGES.NETWORK_ERROR
    }
  }

  return ERROR_MESSAGES.INTERNAL_ERROR
}
```

## Toast Notifications

```typescript
'use client'

import { toast } from 'sonner' // or your toast library

export function showError(error: unknown) {
  const message = getUserMessage(error)
  
  toast.error(message, {
    duration: 5000,
    action: {
      label: 'Dismiss',
      onClick: () => {},
    },
  })
}

export function showSuccess(message: string) {
  toast.success(message)
}

// Usage in component
async function handleSubmit() {
  try {
    await createPost(data)
    showSuccess('Post created successfully!')
  } catch (error) {
    showError(error)
  }
}
```

## Best Practices

1. **Create specific error classes** - Makes handling predictable
2. **Use error boundaries** - Catch rendering errors gracefully
3. **Log structured data** - Include context for debugging
4. **Show user-friendly messages** - Never expose technical details
5. **Fail gracefully** - Provide recovery options
6. **Don't swallow errors** - Always log before handling
7. **Type your errors** - Use discriminated unions when possible
