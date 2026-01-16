---
name: typescript-patterns
description: |
  Advanced TypeScript patterns for type-safe applications.
  Use when implementing complex types, generics, utility types,
  discriminated unions, or type guards in TypeScript projects.
---

# TypeScript Patterns

Modern TypeScript patterns for building type-safe applications.

## Utility Types

### Built-in Utility Types

```typescript
interface User {
  id: string
  name: string
  email: string
  createdAt: Date
  role: 'admin' | 'user'
}

// Make all properties optional
type PartialUser = Partial<User>

// Make all properties required
type RequiredUser = Required<User>

// Make all properties readonly
type ReadonlyUser = Readonly<User>

// Pick specific properties
type UserCredentials = Pick<User, 'email' | 'name'>

// Omit specific properties
type PublicUser = Omit<User, 'email'>

// Record type
type UserRoles = Record<string, User['role']>

// Extract from union
type AdminRole = Extract<User['role'], 'admin'>

// Exclude from union
type NonAdminRole = Exclude<User['role'], 'admin'>

// Non-nullable
type NonNullableId = NonNullable<string | null | undefined>

// Return type of function
type GetUserReturn = ReturnType<typeof getUser>

// Parameters of function
type GetUserParams = Parameters<typeof getUser>
```

### Custom Utility Types

```typescript
// Deep partial
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P]
}

// Deep readonly
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P]
}

// Make specific keys optional
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>

// Make specific keys required
type RequiredBy<T, K extends keyof T> = T & Required<Pick<T, K>>

// Nullable
type Nullable<T> = T | null

// Maybe (nullable or undefined)
type Maybe<T> = T | null | undefined

// Value of object
type ValueOf<T> = T[keyof T]

// Async function type
type AsyncFunction<T> = () => Promise<T>
```

## Discriminated Unions

```typescript
// Status pattern
type Status = 
  | { type: 'idle' }
  | { type: 'loading' }
  | { type: 'success'; data: User[] }
  | { type: 'error'; error: Error }

function handleStatus(status: Status) {
  switch (status.type) {
    case 'idle':
      return 'Ready'
    case 'loading':
      return 'Loading...'
    case 'success':
      return `Loaded ${status.data.length} users` // data is typed
    case 'error':
      return `Error: ${status.error.message}` // error is typed
  }
}

// Result pattern (like Rust)
type Result<T, E = Error> = 
  | { ok: true; value: T }
  | { ok: false; error: E }

function parseJSON<T>(json: string): Result<T> {
  try {
    return { ok: true, value: JSON.parse(json) }
  } catch (e) {
    return { ok: false, error: e as Error }
  }
}

const result = parseJSON<User>('{"name": "John"}')
if (result.ok) {
  console.log(result.value.name) // Typed as User
} else {
  console.error(result.error.message)
}
```

## Type Guards

```typescript
// typeof guard
function isString(value: unknown): value is string {
  return typeof value === 'string'
}

// instanceof guard
function isError(value: unknown): value is Error {
  return value instanceof Error
}

// Property guard
function hasProperty<K extends string>(
  obj: unknown,
  key: K
): obj is Record<K, unknown> {
  return obj !== null && typeof obj === 'object' && key in obj
}

// Discriminated union guard
function isSuccess<T>(result: Result<T>): result is { ok: true; value: T } {
  return result.ok
}

// Array guard
function isNonEmptyArray<T>(arr: T[]): arr is [T, ...T[]] {
  return arr.length > 0
}

// Usage
function processValue(value: unknown) {
  if (isString(value)) {
    // value is string here
    return value.toUpperCase()
  }
  
  if (hasProperty(value, 'name') && isString(value.name)) {
    return value.name
  }
  
  return null
}
```

## Const Assertions

```typescript
// Object as const
const COLORS = {
  primary: '#6366f1',
  secondary: '#22d3ee',
  error: '#ef4444',
} as const

type Color = typeof COLORS[keyof typeof COLORS]
// Type: '#6366f1' | '#22d3ee' | '#ef4444'

// Array as const
const ROLES = ['admin', 'user', 'guest'] as const
type Role = typeof ROLES[number]
// Type: 'admin' | 'user' | 'guest'

// Enum-like pattern (preferred over enums)
const STATUS = {
  PENDING: 'pending',
  ACTIVE: 'active',
  INACTIVE: 'inactive',
} as const

type Status = typeof STATUS[keyof typeof STATUS]
```

## Generics

### Generic Functions

```typescript
// Basic generic
function identity<T>(value: T): T {
  return value
}

// Constrained generic
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}

// Default generic
function createArray<T = string>(length: number, value: T): T[] {
  return Array(length).fill(value)
}

// Multiple generics
function map<T, U>(arr: T[], fn: (item: T) => U): U[] {
  return arr.map(fn)
}
```

### Generic Interfaces

```typescript
interface Repository<T, ID = string> {
  find(id: ID): Promise<T | null>
  findAll(): Promise<T[]>
  create(data: Omit<T, 'id'>): Promise<T>
  update(id: ID, data: Partial<T>): Promise<T>
  delete(id: ID): Promise<void>
}

class UserRepository implements Repository<User> {
  async find(id: string) { /* ... */ }
  async findAll() { /* ... */ }
  async create(data: Omit<User, 'id'>) { /* ... */ }
  async update(id: string, data: Partial<User>) { /* ... */ }
  async delete(id: string) { /* ... */ }
}
```

### Generic Components (React)

```typescript
interface ListProps<T> {
  items: T[]
  renderItem: (item: T) => React.ReactNode
  keyExtractor: (item: T) => string
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map(item => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  )
}

// Usage - T is inferred
<List
  items={users}
  renderItem={user => <span>{user.name}</span>}
  keyExtractor={user => user.id}
/>
```

## Template Literal Types

```typescript
type EventName = 'click' | 'focus' | 'blur'
type Handler = `on${Capitalize<EventName>}`
// Type: 'onClick' | 'onFocus' | 'onBlur'

type HTTPMethod = 'GET' | 'POST' | 'PUT' | 'DELETE'
type Endpoint = '/users' | '/posts'
type Route = `${HTTPMethod} ${Endpoint}`
// Type: 'GET /users' | 'GET /posts' | 'POST /users' | ...

// Extract parts
type ExtractPath<T extends string> = 
  T extends `${infer Method} ${infer Path}` ? Path : never

type Path = ExtractPath<'GET /users'> // '/users'
```

## Mapped Types

```typescript
// Make all properties nullable
type Nullable<T> = {
  [K in keyof T]: T[K] | null
}

// Prefix all keys
type Prefixed<T, P extends string> = {
  [K in keyof T as `${P}${Capitalize<string & K>}`]: T[K]
}

type PrefixedUser = Prefixed<User, 'user'>
// { userId: string; userName: string; userEmail: string; ... }

// Filter properties by type
type FilterByType<T, U> = {
  [K in keyof T as T[K] extends U ? K : never]: T[K]
}

type StringPropsOnly = FilterByType<User, string>
// { id: string; name: string; email: string }
```

## Branded Types

```typescript
// Create nominal types for type safety
declare const brand: unique symbol

type Brand<T, B> = T & { [brand]: B }

type UserId = Brand<string, 'UserId'>
type PostId = Brand<string, 'PostId'>

function getUser(id: UserId) { /* ... */ }
function getPost(id: PostId) { /* ... */ }

const userId = 'user-123' as UserId
const postId = 'post-456' as PostId

getUser(userId) // ✅ OK
getUser(postId) // ❌ Error: PostId not assignable to UserId

// Factory function
function createUserId(id: string): UserId {
  // Add validation if needed
  return id as UserId
}
```

## Zod Integration

```typescript
import { z } from 'zod'

// Define schema
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(2).max(100),
  email: z.string().email(),
  role: z.enum(['admin', 'user']),
  createdAt: z.coerce.date(),
})

// Infer type from schema
type User = z.infer<typeof UserSchema>
// { id: string; name: string; email: string; role: 'admin' | 'user'; createdAt: Date }

// Validate
function parseUser(data: unknown): User {
  return UserSchema.parse(data) // Throws on invalid
}

// Safe parse
function safeParseUser(data: unknown): Result<User> {
  const result = UserSchema.safeParse(data)
  if (result.success) {
    return { ok: true, value: result.data }
  }
  return { ok: false, error: new Error(result.error.message) }
}
```

## Best Practices

1. **Prefer `unknown` over `any`** - Forces type checking
2. **Use `const` assertions** - For literal types
3. **Avoid enums** - Use const objects instead
4. **Narrow types early** - Use type guards
5. **Create utility types** - For reusable patterns
6. **Use branded types** - For domain modeling
7. **Validate at boundaries** - Use Zod for runtime validation
