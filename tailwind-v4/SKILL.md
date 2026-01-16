---
name: tailwind-v4
description: |
  Tailwind CSS v4 patterns and new features including CSS-first configuration,
  @theme directive, container queries, and modern utilities.
  Use when styling with Tailwind CSS v4+ or migrating from v3.
---

# Tailwind CSS v4 Patterns

New features and patterns for Tailwind CSS v4.

## Key Changes from v3

| v3 | v4 |
|----|-----|
| `tailwind.config.js` | CSS-based `@theme` |
| PostCSS plugin config | `@import "tailwindcss"` |
| `@apply` for utilities | Still supported |
| JavaScript theme | CSS custom properties |

## Setup

```css
/* app/globals.css */
@import "tailwindcss";

@theme {
  /* Your custom theme configuration */
}
```

```javascript
// postcss.config.mjs
export default {
  plugins: {
    '@tailwindcss/postcss': {},
  },
}
```

## @theme Directive

### Colors

```css
@theme {
  /* Custom color scale */
  --color-primary-50: #eef2ff;
  --color-primary-100: #e0e7ff;
  --color-primary-200: #c7d2fe;
  --color-primary-300: #a5b4fc;
  --color-primary-400: #818cf8;
  --color-primary-500: #6366f1;
  --color-primary-600: #4f46e5;
  --color-primary-700: #4338ca;
  --color-primary-800: #3730a3;
  --color-primary-900: #312e81;
  --color-primary-950: #1e1b4b;

  /* Semantic colors */
  --color-success: #22c55e;
  --color-warning: #f59e0b;
  --color-error: #ef4444;

  /* Surface colors for dark mode */
  --color-surface-800: #1e293b;
  --color-surface-900: #0f172a;
  --color-surface-950: #020617;
}
```

### Typography

```css
@theme {
  /* Font families */
  --font-sans: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;

  /* Font sizes (with line-height) */
  --text-xs: 0.75rem;
  --text-sm: 0.875rem;
  --text-base: 1rem;
  --text-lg: 1.125rem;
  --text-xl: 1.25rem;
  --text-2xl: 1.5rem;
  --text-3xl: 1.875rem;
  --text-4xl: 2.25rem;

  /* Line heights */
  --leading-tight: 1.25;
  --leading-normal: 1.5;
  --leading-relaxed: 1.75;

  /* Letter spacing */
  --tracking-tight: -0.025em;
  --tracking-normal: 0;
  --tracking-wide: 0.025em;
}
```

### Spacing & Sizing

```css
@theme {
  /* Custom spacing scale */
  --spacing-18: 4.5rem;
  --spacing-88: 22rem;
  --spacing-128: 32rem;

  /* Container sizes */
  --width-content: 65ch;
  --width-prose: 75ch;
  
  /* Breakpoints */
  --breakpoint-xs: 475px;
  --breakpoint-3xl: 1920px;
}
```

### Animations

```css
@theme {
  /* Transition durations */
  --duration-fast: 150ms;
  --duration-normal: 300ms;
  --duration-slow: 500ms;

  /* Easing functions */
  --ease-in-out-back: cubic-bezier(0.68, -0.55, 0.265, 1.55);
  --ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);

  /* Keyframe animations */
  --animate-fade-in: fade-in 0.3s ease-out;
  --animate-slide-up: slide-up 0.3s ease-out;
  --animate-spin: spin 1s linear infinite;
}

@keyframes fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes slide-up {
  from { 
    opacity: 0;
    transform: translateY(10px);
  }
  to { 
    opacity: 1;
    transform: translateY(0);
  }
}
```

### Shadows & Effects

```css
@theme {
  /* Box shadows */
  --shadow-soft: 0 2px 15px -3px rgb(0 0 0 / 0.1);
  --shadow-glow: 0 0 20px rgb(99 102 241 / 0.3);
  --shadow-inner-glow: inset 0 0 20px rgb(99 102 241 / 0.1);

  /* Blur values */
  --blur-xs: 2px;
  --blur-sm: 4px;
  --blur-md: 8px;
  --blur-lg: 16px;
  --blur-xl: 24px;

  /* Border radius */
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 1rem;
  --radius-xl: 1.5rem;
  --radius-full: 9999px;
}
```

## Container Queries

```css
/* Define container */
.card-container {
  container-type: inline-size;
  container-name: card;
}
```

```html
<!-- Usage with @container -->
<div class="card-container">
  <div class="@container">
    <div class="flex flex-col @md:flex-row @lg:gap-6">
      <img class="w-full @md:w-1/3" src="..." />
      <div class="@md:flex-1">
        <h3 class="text-lg @lg:text-xl">Title</h3>
        <p class="hidden @sm:block">Description</p>
      </div>
    </div>
  </div>
</div>
```

**Container query breakpoints:**
- `@xs` - 320px
- `@sm` - 384px
- `@md` - 448px
- `@lg` - 512px
- `@xl` - 576px
- `@2xl` - 672px

## New Utilities

### Text Balance & Wrap

```html
<!-- Balanced text (headlines) -->
<h1 class="text-balance">
  A very long headline that wraps nicely across multiple lines
</h1>

<!-- Pretty text (paragraphs) -->
<p class="text-pretty">
  Body text with improved line breaking
</p>
```

### Logical Properties

```html
<!-- Logical margin/padding (for RTL support) -->
<div class="ms-4">Margin start (left in LTR, right in RTL)</div>
<div class="me-4">Margin end</div>
<div class="ps-4">Padding start</div>
<div class="pe-4">Padding end</div>

<!-- Logical borders -->
<div class="border-s-2">Border start</div>
<div class="border-e-2">Border end</div>
```

### Subgrid

```html
<div class="grid grid-cols-3">
  <div class="col-span-2 grid grid-cols-subgrid">
    <!-- Inherits parent grid columns -->
  </div>
</div>
```

### Starting Style (View Transitions)

```html
<div class="starting:opacity-0 starting:scale-95 transition-all">
  Content that animates in
</div>
```

### Field Sizing (Auto-growing inputs)

```html
<textarea class="field-sizing-content">
  Auto-grows with content
</textarea>
```

## Dark Mode

```css
/* v4 uses :dark selector by default */
body {
  @apply bg-surface-950 text-white;
}

/* Or use dark: prefix in HTML */
```

```html
<div class="bg-white dark:bg-surface-900">
  <p class="text-gray-900 dark:text-white">Content</p>
</div>
```

## Custom Utilities with @utility

```css
/* Define custom utilities */
@utility glass {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.2);
}

@utility gradient-text {
  background: linear-gradient(135deg, var(--color-primary-400), var(--color-accent-400));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

@utility hover-lift {
  transition: transform 0.2s ease-out;
  &:hover {
    transform: translateY(-4px);
  }
}
```

```html
<!-- Usage -->
<div class="glass rounded-xl p-6">
  <h2 class="gradient-text text-2xl">Glassmorphism Card</h2>
</div>

<button class="hover-lift">Lift on hover</button>
```

## Component Patterns

### Glass Card

```html
<div class="glass rounded-xl p-6 shadow-glow">
  <h3 class="text-xl font-bold mb-2">Card Title</h3>
  <p class="text-surface-300">Card content goes here.</p>
</div>
```

```css
@utility glass {
  background: rgba(15, 23, 42, 0.6);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.1);
}
```

### Gradient Border

```html
<div class="relative p-[1px] rounded-xl bg-gradient-to-r from-primary-500 to-accent-500">
  <div class="bg-surface-900 rounded-xl p-6">
    Content with gradient border
  </div>
</div>
```

### Animated Button

```html
<button class="
  relative px-6 py-3 
  bg-primary-500 hover:bg-primary-600
  text-white font-medium rounded-lg
  transition-all duration-fast
  hover:-translate-y-0.5 hover:shadow-glow
  active:translate-y-0 active:shadow-none
">
  Click me
</button>
```

### Skeleton Loading

```css
@utility skeleton {
  background: linear-gradient(
    90deg,
    var(--color-surface-800) 0%,
    var(--color-surface-700) 50%,
    var(--color-surface-800) 100%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}

@keyframes shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
```

```html
<div class="skeleton h-4 w-48 rounded"></div>
```

## Migration from v3

### Config Migration

```javascript
// v3: tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        primary: {
          500: '#6366f1',
        },
      },
    },
  },
}
```

```css
/* v4: globals.css */
@theme {
  --color-primary-500: #6366f1;
}
```

### Breaking Changes

1. **No `tailwind.config.js`** - Use `@theme` in CSS
2. **No `@tailwind` directives** - Use `@import "tailwindcss"`
3. **Default dark mode is `:dark`** - Not `class`
4. **Some utilities renamed** - Check migration guide

## Best Practices

1. **Use CSS custom properties** - For dynamic theming
2. **Create semantic utilities** - `@utility` for component styles
3. **Leverage container queries** - For responsive components
4. **Use logical properties** - For internationalization
5. **Define animation presets** - In `@theme` for consistency
