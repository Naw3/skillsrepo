---
name: framer-motion
description: |
  Animation patterns and micro-interactions with Framer Motion in React/Next.js.
  Use when implementing animations, page transitions, gesture interactions, 
  scroll-based animations, or any motion design in React applications.
---

# Framer Motion Patterns

Comprehensive animation patterns for React applications using Framer Motion.

## Quick Reference

```typescript
import { motion, AnimatePresence, useScroll, useTransform } from 'framer-motion'
```

## Core Concepts

### Basic Animation

```typescript
// Simple animate prop
<motion.div
  initial={{ opacity: 0, y: 20 }}
  animate={{ opacity: 1, y: 0 }}
  exit={{ opacity: 0, y: -20 }}
  transition={{ duration: 0.3, ease: 'easeOut' }}
>
  Content
</motion.div>
```

### Variants (Recommended Pattern)

```typescript
const containerVariants = {
  hidden: { opacity: 0 },
  visible: {
    opacity: 1,
    transition: {
      staggerChildren: 0.1,
      delayChildren: 0.2,
    },
  },
  exit: { opacity: 0 },
}

const itemVariants = {
  hidden: { opacity: 0, y: 20 },
  visible: { opacity: 1, y: 0 },
  exit: { opacity: 0, y: -10 },
}

function StaggerList({ items }) {
  return (
    <motion.ul
      variants={containerVariants}
      initial="hidden"
      animate="visible"
      exit="exit"
    >
      {items.map((item) => (
        <motion.li key={item.id} variants={itemVariants}>
          {item.name}
        </motion.li>
      ))}
    </motion.ul>
  )
}
```

## Patterns

### 1. Page Transitions

```typescript
// app/template.tsx - Wraps each page
'use client'

import { motion } from 'framer-motion'

export default function Template({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      exit={{ opacity: 0, y: 20 }}
      transition={{ duration: 0.3, ease: [0.25, 0.46, 0.45, 0.94] }}
    >
      {children}
    </motion.div>
  )
}
```

### 2. AnimatePresence for Exit Animations

```typescript
'use client'

import { AnimatePresence, motion } from 'framer-motion'
import { useState } from 'react'

function Modal({ isOpen, onClose, children }) {
  return (
    <AnimatePresence mode="wait">
      {isOpen && (
        <>
          {/* Backdrop */}
          <motion.div
            key="backdrop"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
            className="fixed inset-0 bg-black/50 backdrop-blur-sm"
          />
          
          {/* Modal */}
          <motion.div
            key="modal"
            initial={{ opacity: 0, scale: 0.95, y: 20 }}
            animate={{ opacity: 1, scale: 1, y: 0 }}
            exit={{ opacity: 0, scale: 0.95, y: 20 }}
            transition={{ type: 'spring', damping: 25, stiffness: 300 }}
            className="fixed inset-0 flex items-center justify-center"
          >
            {children}
          </motion.div>
        </>
      )}
    </AnimatePresence>
  )
}
```

### 3. Gesture Interactions

```typescript
<motion.button
  whileHover={{ scale: 1.05, y: -2 }}
  whileTap={{ scale: 0.95 }}
  transition={{ type: 'spring', stiffness: 400, damping: 17 }}
>
  Click me
</motion.button>

// Drag
<motion.div
  drag
  dragConstraints={{ left: -100, right: 100, top: -50, bottom: 50 }}
  dragElastic={0.1}
  whileDrag={{ scale: 1.1, cursor: 'grabbing' }}
/>
```

### 4. Scroll Animations

```typescript
'use client'

import { motion, useScroll, useTransform } from 'framer-motion'
import { useRef } from 'react'

function ParallaxHero() {
  const ref = useRef(null)
  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ['start start', 'end start'],
  })

  const y = useTransform(scrollYProgress, [0, 1], ['0%', '50%'])
  const opacity = useTransform(scrollYProgress, [0, 0.5], [1, 0])

  return (
    <section ref={ref} className="relative h-screen overflow-hidden">
      <motion.div style={{ y, opacity }} className="absolute inset-0">
        <img src="/hero.jpg" alt="Hero" className="object-cover w-full h-full" />
      </motion.div>
      <div className="relative z-10">
        <h1>Welcome</h1>
      </div>
    </section>
  )
}
```

### 5. Layout Animations

```typescript
'use client'

import { motion, LayoutGroup } from 'framer-motion'

function Tabs({ tabs, activeTab, onChange }) {
  return (
    <LayoutGroup>
      <div className="flex gap-2">
        {tabs.map((tab) => (
          <button
            key={tab.id}
            onClick={() => onChange(tab.id)}
            className="relative px-4 py-2"
          >
            {tab.label}
            {activeTab === tab.id && (
              <motion.div
                layoutId="activeTab"
                className="absolute inset-0 bg-primary-500 rounded-lg -z-10"
                transition={{ type: 'spring', bounce: 0.2, duration: 0.6 }}
              />
            )}
          </button>
        ))}
      </div>
    </LayoutGroup>
  )
}
```

### 6. Shared Element Transitions

```typescript
// List view
<motion.div layoutId={`card-${item.id}`}>
  <img src={item.thumbnail} />
  <h3>{item.title}</h3>
</motion.div>

// Detail view
<motion.div layoutId={`card-${item.id}`}>
  <img src={item.fullImage} />
  <h1>{item.title}</h1>
  <p>{item.description}</p>
</motion.div>
```

### 7. Skeleton to Content Transition

```typescript
function Card({ isLoading, data }) {
  return (
    <AnimatePresence mode="wait">
      {isLoading ? (
        <motion.div
          key="skeleton"
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          exit={{ opacity: 0 }}
          className="animate-pulse bg-gray-200 rounded-lg h-48"
        />
      ) : (
        <motion.div
          key="content"
          initial={{ opacity: 0, y: 10 }}
          animate={{ opacity: 1, y: 0 }}
          transition={{ duration: 0.3 }}
        >
          <h3>{data.title}</h3>
          <p>{data.description}</p>
        </motion.div>
      )}
    </AnimatePresence>
  )
}
```

### 8. Ambient Background Effects

```typescript
'use client'

import { motion } from 'framer-motion'

export function AmbientBackground({ intensity = 0.5 }) {
  return (
    <div className="fixed inset-0 -z-10 overflow-hidden pointer-events-none">
      {/* Floating orbs */}
      <motion.div
        className="absolute w-96 h-96 rounded-full bg-primary-500/20 blur-3xl"
        animate={{
          x: [0, 100, 0],
          y: [0, -50, 0],
          scale: [1, 1.1, 1],
        }}
        transition={{
          duration: 20,
          repeat: Infinity,
          ease: 'easeInOut',
        }}
        style={{ top: '10%', left: '20%', opacity: intensity }}
      />
      <motion.div
        className="absolute w-80 h-80 rounded-full bg-accent-500/20 blur-3xl"
        animate={{
          x: [0, -80, 0],
          y: [0, 60, 0],
          scale: [1, 0.9, 1],
        }}
        transition={{
          duration: 15,
          repeat: Infinity,
          ease: 'easeInOut',
          delay: 2,
        }}
        style={{ bottom: '20%', right: '10%', opacity: intensity }}
      />
    </div>
  )
}
```

## Performance Best Practices

### 1. Use `will-change` Sparingly

```typescript
// Only for complex animations that need GPU hint
<motion.div
  style={{ willChange: 'transform' }}
  animate={{ x: 100 }}
/>
```

### 2. Prefer Transform Properties

```typescript
// ✅ Good - GPU accelerated
<motion.div animate={{ x: 100, scale: 1.1, rotate: 45 }} />

// ❌ Avoid - triggers layout
<motion.div animate={{ width: '200px', left: '100px' }} />
```

### 3. Respect Reduced Motion

```typescript
import { useReducedMotion } from 'framer-motion'

function AnimatedComponent() {
  const shouldReduceMotion = useReducedMotion()

  return (
    <motion.div
      animate={{ x: shouldReduceMotion ? 0 : 100 }}
      transition={{ duration: shouldReduceMotion ? 0 : 0.5 }}
    />
  )
}
```

### 4. Lazy Load Heavy Animations

```typescript
import dynamic from 'next/dynamic'

const HeavyAnimation = dynamic(
  () => import('./HeavyAnimation'),
  { ssr: false }
)
```

## Server Components Compatibility

Framer Motion requires `'use client'`. Pattern for RSC:

```typescript
// components/motion/index.ts - Re-export with client boundary
'use client'

export { motion, AnimatePresence } from 'framer-motion'
export { FadeIn } from './FadeIn'
export { SlideIn } from './SlideIn'

// Usage in Server Component
import { FadeIn } from '@/components/motion'

export default function Page() {
  return (
    <FadeIn>
      <ServerContent />
    </FadeIn>
  )
}
```

## Transition Presets

```typescript
export const transitions = {
  spring: { type: 'spring', stiffness: 300, damping: 30 },
  springBouncy: { type: 'spring', stiffness: 400, damping: 17 },
  smooth: { duration: 0.3, ease: [0.25, 0.46, 0.45, 0.94] },
  snappy: { duration: 0.15, ease: 'easeOut' },
  slow: { duration: 0.6, ease: 'easeInOut' },
}

// Usage
<motion.div transition={transitions.spring} />
```

## Animation Variants Library

```typescript
export const fadeInUp = {
  initial: { opacity: 0, y: 20 },
  animate: { opacity: 1, y: 0 },
  exit: { opacity: 0, y: -10 },
}

export const fadeInScale = {
  initial: { opacity: 0, scale: 0.95 },
  animate: { opacity: 1, scale: 1 },
  exit: { opacity: 0, scale: 0.95 },
}

export const slideInRight = {
  initial: { opacity: 0, x: 50 },
  animate: { opacity: 1, x: 0 },
  exit: { opacity: 0, x: -50 },
}

export const staggerContainer = {
  hidden: { opacity: 0 },
  visible: {
    opacity: 1,
    transition: { staggerChildren: 0.1 },
  },
}
```
