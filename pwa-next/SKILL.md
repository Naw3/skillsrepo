---
name: pwa-next
description: |
  Progressive Web App patterns for Next.js including service workers,
  offline support, caching strategies, and installability. Use when
  building offline-capable web applications with Next.js.
---

# PWA Patterns for Next.js

Progressive Web App implementation patterns for Next.js.

## Setup with next-pwa

```bash
npm install next-pwa
```

```typescript
// next.config.ts
import withPWA from 'next-pwa'

const nextConfig = {
  // your config
}

export default withPWA({
  dest: 'public',
  register: true,
  skipWaiting: true,
  disable: process.env.NODE_ENV === 'development',
})(nextConfig)
```

## Web App Manifest

```json
// public/manifest.json
{
  "name": "My App",
  "short_name": "MyApp",
  "description": "My awesome PWA application",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#0a0a0a",
  "theme_color": "#6366f1",
  "orientation": "portrait-primary",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/desktop.png",
      "sizes": "1280x720",
      "type": "image/png",
      "form_factor": "wide"
    },
    {
      "src": "/screenshots/mobile.png",
      "sizes": "750x1334",
      "type": "image/png",
      "form_factor": "narrow"
    }
  ]
}
```

```typescript
// app/layout.tsx
export const metadata = {
  manifest: '/manifest.json',
  themeColor: '#6366f1',
  appleWebApp: {
    capable: true,
    statusBarStyle: 'default',
    title: 'My App',
  },
}
```

## Custom Service Worker

```typescript
// public/sw.js
import { precacheAndRoute } from 'workbox-precaching'
import { registerRoute } from 'workbox-routing'
import { 
  StaleWhileRevalidate, 
  CacheFirst, 
  NetworkFirst 
} from 'workbox-strategies'
import { ExpirationPlugin } from 'workbox-expiration'
import { CacheableResponsePlugin } from 'workbox-cacheable-response'

// Precache static assets
precacheAndRoute(self.__WB_MANIFEST)

// Cache API responses
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({
    cacheName: 'api-cache',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 100,
        maxAgeSeconds: 60 * 60, // 1 hour
      }),
    ],
  })
)

// Cache images
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'image-cache',
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200],
      }),
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 60 * 60 * 24 * 30, // 30 days
      }),
    ],
  })
)

// Cache fonts
registerRoute(
  ({ request }) => request.destination === 'font',
  new CacheFirst({
    cacheName: 'font-cache',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 10,
        maxAgeSeconds: 60 * 60 * 24 * 365, // 1 year
      }),
    ],
  })
)

// Cache pages
registerRoute(
  ({ request }) => request.mode === 'navigate',
  new NetworkFirst({
    cacheName: 'page-cache',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 50,
      }),
    ],
  })
)
```

## Offline Fallback Page

```typescript
// app/offline/page.tsx
export default function OfflinePage() {
  return (
    <div className="min-h-screen flex items-center justify-center">
      <div className="text-center p-6">
        <div className="text-6xl mb-4">📡</div>
        <h1 className="text-2xl font-bold mb-2">You're Offline</h1>
        <p className="text-gray-500 mb-4">
          Please check your internet connection and try again.
        </p>
        <button
          onClick={() => window.location.reload()}
          className="px-4 py-2 bg-primary-500 text-white rounded-lg"
        >
          Retry
        </button>
      </div>
    </div>
  )
}
```

```javascript
// In service worker
self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => {
        return caches.match('/offline')
      })
    )
  }
})
```

## Online/Offline Detection

```typescript
'use client'

import { useSyncExternalStore } from 'react'

function subscribe(callback: () => void) {
  window.addEventListener('online', callback)
  window.addEventListener('offline', callback)
  return () => {
    window.removeEventListener('online', callback)
    window.removeEventListener('offline', callback)
  }
}

function getSnapshot() {
  return navigator.onLine
}

function getServerSnapshot() {
  return true
}

export function useOnlineStatus() {
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot)
}

// Usage
function ConnectionStatus() {
  const isOnline = useOnlineStatus()
  
  return (
    <div className={`fixed bottom-4 right-4 px-4 py-2 rounded-lg ${
      isOnline ? 'bg-green-500' : 'bg-red-500'
    } text-white`}>
      {isOnline ? '🟢 Online' : '🔴 Offline'}
    </div>
  )
}
```

## Background Sync

```typescript
// Register background sync
async function saveForLater(data: FormData) {
  if ('serviceWorker' in navigator && 'SyncManager' in window) {
    const registration = await navigator.serviceWorker.ready
    
    // Store data in IndexedDB
    await storeInIDB('pending-sync', data)
    
    // Register sync
    await registration.sync.register('sync-forms')
  } else {
    // Fallback: try to send immediately
    await sendData(data)
  }
}

// In service worker
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-forms') {
    event.waitUntil(syncPendingForms())
  }
})

async function syncPendingForms() {
  const pendingForms = await getFromIDB('pending-sync')
  
  for (const form of pendingForms) {
    try {
      await sendData(form)
      await removeFromIDB('pending-sync', form.id)
    } catch (error) {
      // Will retry on next sync
      console.error('Sync failed:', error)
    }
  }
}
```

## Push Notifications

```typescript
// Request permission and subscribe
async function subscribeToPush() {
  const registration = await navigator.serviceWorker.ready
  
  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(VAPID_PUBLIC_KEY),
  })
  
  // Send subscription to server
  await fetch('/api/push/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(subscription),
  })
  
  return subscription
}

// In service worker
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? {}
  
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icons/icon-192.png',
      badge: '/icons/badge.png',
      data: data.url,
      actions: [
        { action: 'open', title: 'Open' },
        { action: 'dismiss', title: 'Dismiss' },
      ],
    })
  )
})

self.addEventListener('notificationclick', (event) => {
  event.notification.close()
  
  if (event.action === 'open' || !event.action) {
    event.waitUntil(
      clients.openWindow(event.notification.data || '/')
    )
  }
})
```

## Install Prompt

```typescript
'use client'

import { useState, useEffect } from 'react'

interface BeforeInstallPromptEvent extends Event {
  prompt: () => Promise<void>
  userChoice: Promise<{ outcome: 'accepted' | 'dismissed' }>
}

export function useInstallPrompt() {
  const [prompt, setPrompt] = useState<BeforeInstallPromptEvent | null>(null)
  const [isInstalled, setIsInstalled] = useState(false)

  useEffect(() => {
    // Check if already installed
    if (window.matchMedia('(display-mode: standalone)').matches) {
      setIsInstalled(true)
      return
    }

    const handler = (e: Event) => {
      e.preventDefault()
      setPrompt(e as BeforeInstallPromptEvent)
    }

    window.addEventListener('beforeinstallprompt', handler)
    
    return () => window.removeEventListener('beforeinstallprompt', handler)
  }, [])

  const install = async () => {
    if (!prompt) return false
    
    await prompt.prompt()
    const { outcome } = await prompt.userChoice
    
    if (outcome === 'accepted') {
      setIsInstalled(true)
      setPrompt(null)
    }
    
    return outcome === 'accepted'
  }

  return { canInstall: !!prompt && !isInstalled, isInstalled, install }
}

// Usage
function InstallButton() {
  const { canInstall, install } = useInstallPrompt()
  
  if (!canInstall) return null
  
  return (
    <button
      onClick={install}
      className="px-4 py-2 bg-primary-500 text-white rounded-lg"
    >
      📲 Install App
    </button>
  )
}
```

## IndexedDB for Offline Data

```typescript
import { openDB, DBSchema } from 'idb'

interface MyDB extends DBSchema {
  posts: {
    key: string
    value: Post
    indexes: { 'by-date': Date }
  }
  drafts: {
    key: string
    value: Draft
  }
}

const dbPromise = openDB<MyDB>('my-app-db', 1, {
  upgrade(db) {
    const postsStore = db.createObjectStore('posts', { keyPath: 'id' })
    postsStore.createIndex('by-date', 'createdAt')
    
    db.createObjectStore('drafts', { keyPath: 'id' })
  },
})

// CRUD operations
export async function saveDraft(draft: Draft) {
  const db = await dbPromise
  await db.put('drafts', draft)
}

export async function getDrafts() {
  const db = await dbPromise
  return db.getAll('drafts')
}

export async function deleteDraft(id: string) {
  const db = await dbPromise
  await db.delete('drafts', id)
}
```

## Best Practices

1. **Cache strategically** - Match strategy to content type
2. **Provide offline fallbacks** - Graceful degradation
3. **Show sync status** - Keep users informed
4. **Handle updates** - Prompt to reload for new versions
5. **Optimize assets** - Compress icons and images
6. **Test thoroughly** - Use Lighthouse and real devices
7. **Respect data usage** - Don't cache excessively on mobile
