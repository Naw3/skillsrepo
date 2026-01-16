---
name: zustand-patterns
description: |
  State management patterns with Zustand 5, including store creation,
  slices pattern, persistence, devtools, and React integration.
  Use when implementing global state, complex store structures, or
  migrating from other state management solutions.
---

# Zustand Patterns

Modern state management patterns for Zustand 5 with TypeScript.

## Setup

```bash
npm install zustand
```

## Basic Store

```typescript
import { create } from 'zustand'

interface CounterStore {
  count: number
  increment: () => void
  decrement: () => void
  reset: () => void
}

export const useCounterStore = create<CounterStore>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}))

// Usage in component
function Counter() {
  const { count, increment, decrement } = useCounterStore()
  
  return (
    <div>
      <span>{count}</span>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  )
}
```

## Selectors (Performance)

```typescript
// ❌ Re-renders on any store change
const { count, user } = useStore()

// ✅ Only re-renders when count changes
const count = useStore((state) => state.count)

// ✅ Multiple selectors with shallow comparison
import { useShallow } from 'zustand/shallow'

const { count, user } = useStore(
  useShallow((state) => ({ count: state.count, user: state.user }))
)
```

## Slices Pattern (Modular Stores)

```typescript
import { create } from 'zustand'

// User slice
interface UserSlice {
  user: User | null
  setUser: (user: User | null) => void
  logout: () => void
}

const createUserSlice = (set: any): UserSlice => ({
  user: null,
  setUser: (user) => set({ user }),
  logout: () => set({ user: null }),
})

// Cart slice
interface CartSlice {
  items: CartItem[]
  addItem: (item: CartItem) => void
  removeItem: (id: string) => void
  clearCart: () => void
}

const createCartSlice = (set: any): CartSlice => ({
  items: [],
  addItem: (item) => set((state: any) => ({ 
    items: [...state.items, item] 
  })),
  removeItem: (id) => set((state: any) => ({
    items: state.items.filter((i: CartItem) => i.id !== id)
  })),
  clearCart: () => set({ items: [] }),
})

// Combined store
type Store = UserSlice & CartSlice

export const useStore = create<Store>()((...args) => ({
  ...createUserSlice(...args),
  ...createCartSlice(...args),
}))
```

## Persistence

```typescript
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

interface SettingsStore {
  theme: 'light' | 'dark'
  language: string
  setTheme: (theme: 'light' | 'dark') => void
  setLanguage: (language: string) => void
}

export const useSettingsStore = create<SettingsStore>()(
  persist(
    (set) => ({
      theme: 'light',
      language: 'en',
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
    }),
    {
      name: 'settings-storage',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({ 
        theme: state.theme,
        language: state.language,
      }),
    }
  )
)

// Hydration check for SSR
function SettingsProvider({ children }: { children: React.ReactNode }) {
  const [hydrated, setHydrated] = useState(false)
  
  useEffect(() => {
    setHydrated(true)
  }, [])
  
  if (!hydrated) {
    return <LoadingSkeleton />
  }
  
  return <>{children}</>
}
```

## DevTools

```typescript
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'

export const useStore = create<Store>()(
  devtools(
    (set) => ({
      count: 0,
      increment: () => set(
        (state) => ({ count: state.count + 1 }),
        undefined,
        'increment' // Action name for devtools
      ),
    }),
    { name: 'MyStore' }
  )
)
```

## Async Actions

```typescript
interface DataStore {
  data: Item[]
  loading: boolean
  error: string | null
  fetchData: () => Promise<void>
}

export const useDataStore = create<DataStore>((set) => ({
  data: [],
  loading: false,
  error: null,
  
  fetchData: async () => {
    set({ loading: true, error: null })
    
    try {
      const response = await fetch('/api/data')
      const data = await response.json()
      set({ data, loading: false })
    } catch (error) {
      set({ 
        error: error instanceof Error ? error.message : 'Unknown error',
        loading: false 
      })
    }
  },
}))
```

## Computed Values (Derived State)

```typescript
interface CartStore {
  items: CartItem[]
  // Computed values as getters
  get totalItems(): number
  get totalPrice(): number
}

export const useCartStore = create<CartStore>((set, get) => ({
  items: [],
  
  get totalItems() {
    return get().items.reduce((sum, item) => sum + item.quantity, 0)
  },
  
  get totalPrice() {
    return get().items.reduce(
      (sum, item) => sum + item.price * item.quantity, 
      0
    )
  },
}))

// Or use subscribeWithSelector for reactive derived state
import { subscribeWithSelector } from 'zustand/middleware'
```

## Immer Integration (Immutable Updates)

```typescript
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'

interface TodoStore {
  todos: Todo[]
  addTodo: (text: string) => void
  toggleTodo: (id: string) => void
  updateTodo: (id: string, text: string) => void
}

export const useTodoStore = create<TodoStore>()(
  immer((set) => ({
    todos: [],
    
    addTodo: (text) => set((state) => {
      state.todos.push({ id: crypto.randomUUID(), text, completed: false })
    }),
    
    toggleTodo: (id) => set((state) => {
      const todo = state.todos.find((t) => t.id === id)
      if (todo) todo.completed = !todo.completed
    }),
    
    updateTodo: (id, text) => set((state) => {
      const todo = state.todos.find((t) => t.id === id)
      if (todo) todo.text = text
    }),
  }))
)
```

## Subscribe to Changes

```typescript
// Subscribe outside React
const unsubscribe = useStore.subscribe(
  (state) => console.log('State changed:', state)
)

// Subscribe with selector
const unsubscribe = useStore.subscribe(
  (state) => state.count,
  (count, prevCount) => {
    console.log('Count changed from', prevCount, 'to', count)
  }
)

// Get state outside React
const currentCount = useStore.getState().count

// Set state outside React
useStore.setState({ count: 10 })
```

## Reset Store

```typescript
const initialState = {
  count: 0,
  user: null,
}

interface Store extends typeof initialState {
  increment: () => void
  reset: () => void
}

export const useStore = create<Store>((set) => ({
  ...initialState,
  increment: () => set((state) => ({ count: state.count + 1 })),
  reset: () => set(initialState),
}))
```

## Context Pattern (Multiple Instances)

```typescript
import { createContext, useContext, useRef } from 'react'
import { createStore, useStore as useZustandStore } from 'zustand'

// Create store factory
const createCounterStore = (initialCount = 0) => 
  createStore<CounterStore>((set) => ({
    count: initialCount,
    increment: () => set((state) => ({ count: state.count + 1 })),
  }))

type CounterStore = ReturnType<typeof createCounterStore>
const CounterContext = createContext<CounterStore | null>(null)

// Provider
export function CounterProvider({ 
  children, 
  initialCount 
}: { 
  children: React.ReactNode
  initialCount?: number 
}) {
  const storeRef = useRef<CounterStore>()
  if (!storeRef.current) {
    storeRef.current = createCounterStore(initialCount)
  }
  
  return (
    <CounterContext.Provider value={storeRef.current}>
      {children}
    </CounterContext.Provider>
  )
}

// Hook
export function useCounter<T>(selector: (state: CounterState) => T): T {
  const store = useContext(CounterContext)
  if (!store) throw new Error('Missing CounterProvider')
  return useZustandStore(store, selector)
}
```

## Best Practices

1. **Use selectors** - Prevent unnecessary re-renders
2. **Keep stores focused** - One concern per store or use slices
3. **Prefer `set` over `get`** - More predictable updates
4. **Use immer for nested updates** - Cleaner code
5. **Name actions in devtools** - Easier debugging
6. **Persist only necessary state** - Use `partialize`
7. **Handle hydration in SSR** - Check hydration state
