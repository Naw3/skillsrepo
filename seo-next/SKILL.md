---
name: seo-next
description: |
  SEO patterns for Next.js App Router including metadata, Open Graph,
  structured data, sitemaps, and Core Web Vitals optimization.
  Use when implementing SEO features or optimizing for search engines.
---

# SEO Patterns for Next.js

Comprehensive SEO implementation patterns for Next.js App Router.

## Metadata API

### Static Metadata

```typescript
// app/layout.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  metadataBase: new URL('https://example.com'),
  title: {
    default: 'My App',
    template: '%s | My App', // "Page Title | My App"
  },
  description: 'The best app for doing things',
  keywords: ['nextjs', 'react', 'seo'],
  authors: [{ name: 'Your Name', url: 'https://yoursite.com' }],
  creator: 'Your Company',
  publisher: 'Your Company',
  robots: {
    index: true,
    follow: true,
    googleBot: {
      index: true,
      follow: true,
      'max-video-preview': -1,
      'max-image-preview': 'large',
      'max-snippet': -1,
    },
  },
  openGraph: {
    type: 'website',
    locale: 'en_US',
    url: 'https://example.com',
    siteName: 'My App',
    title: 'My App - Do Amazing Things',
    description: 'The best app for doing things',
    images: [
      {
        url: '/og-image.jpg',
        width: 1200,
        height: 630,
        alt: 'My App',
      },
    ],
  },
  twitter: {
    card: 'summary_large_image',
    title: 'My App',
    description: 'The best app for doing things',
    creator: '@yourhandle',
    images: ['/twitter-image.jpg'],
  },
  verification: {
    google: 'google-verification-code',
    yandex: 'yandex-verification-code',
  },
  alternates: {
    canonical: 'https://example.com',
    languages: {
      'en-US': 'https://example.com/en-US',
      'fr-FR': 'https://example.com/fr-FR',
    },
  },
}
```

### Dynamic Metadata

```typescript
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next'

type Props = {
  params: Promise<{ slug: string }>
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params
  const post = await getPost(slug)

  if (!post) {
    return { title: 'Not Found' }
  }

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      type: 'article',
      publishedTime: post.publishedAt,
      authors: [post.author.name],
      images: [
        {
          url: post.coverImage,
          width: 1200,
          height: 630,
          alt: post.title,
        },
      ],
    },
    twitter: {
      card: 'summary_large_image',
      title: post.title,
      description: post.excerpt,
      images: [post.coverImage],
    },
  }
}
```

## Dynamic OG Images

```typescript
// app/blog/[slug]/opengraph-image.tsx
import { ImageResponse } from 'next/og'

export const runtime = 'edge'
export const alt = 'Blog Post'
export const size = { width: 1200, height: 630 }
export const contentType = 'image/png'

export default async function Image({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  const post = await getPost(slug)

  return new ImageResponse(
    (
      <div
        style={{
          display: 'flex',
          flexDirection: 'column',
          alignItems: 'flex-start',
          justifyContent: 'center',
          width: '100%',
          height: '100%',
          background: 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
          padding: 60,
        }}
      >
        <div
          style={{
            fontSize: 60,
            fontWeight: 'bold',
            color: 'white',
            marginBottom: 20,
            lineHeight: 1.2,
          }}
        >
          {post.title}
        </div>
        <div
          style={{
            fontSize: 30,
            color: 'rgba(255,255,255,0.8)',
          }}
        >
          {post.author.name} • {post.readTime} min read
        </div>
      </div>
    ),
    { ...size }
  )
}
```

## Structured Data (JSON-LD)

```typescript
// components/JsonLd.tsx
type JsonLdProps = {
  data: Record<string, unknown>
}

export function JsonLd({ data }: JsonLdProps) {
  return (
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{ __html: JSON.stringify(data) }}
    />
  )
}

// Usage in page
import { JsonLd } from '@/components/JsonLd'

export default function ProductPage({ product }) {
  const structuredData = {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: product.name,
    description: product.description,
    image: product.images,
    sku: product.sku,
    brand: {
      '@type': 'Brand',
      name: product.brand,
    },
    offers: {
      '@type': 'Offer',
      url: `https://example.com/products/${product.slug}`,
      priceCurrency: 'USD',
      price: product.price,
      availability: product.inStock
        ? 'https://schema.org/InStock'
        : 'https://schema.org/OutOfStock',
    },
    aggregateRating: product.rating && {
      '@type': 'AggregateRating',
      ratingValue: product.rating,
      reviewCount: product.reviewCount,
    },
  }

  return (
    <>
      <JsonLd data={structuredData} />
      <ProductContent product={product} />
    </>
  )
}
```

### Common Schema Types

```typescript
// Article
const articleSchema = {
  '@context': 'https://schema.org',
  '@type': 'Article',
  headline: 'Article Title',
  description: 'Article description',
  image: 'https://example.com/image.jpg',
  datePublished: '2024-01-15',
  dateModified: '2024-01-16',
  author: {
    '@type': 'Person',
    name: 'Author Name',
    url: 'https://example.com/author',
  },
  publisher: {
    '@type': 'Organization',
    name: 'Company Name',
    logo: {
      '@type': 'ImageObject',
      url: 'https://example.com/logo.png',
    },
  },
}

// Organization
const orgSchema = {
  '@context': 'https://schema.org',
  '@type': 'Organization',
  name: 'Company Name',
  url: 'https://example.com',
  logo: 'https://example.com/logo.png',
  sameAs: [
    'https://twitter.com/company',
    'https://linkedin.com/company/company',
  ],
  contactPoint: {
    '@type': 'ContactPoint',
    telephone: '+1-555-555-5555',
    contactType: 'customer service',
  },
}

// BreadcrumbList
const breadcrumbSchema = {
  '@context': 'https://schema.org',
  '@type': 'BreadcrumbList',
  itemListElement: [
    {
      '@type': 'ListItem',
      position: 1,
      name: 'Home',
      item: 'https://example.com',
    },
    {
      '@type': 'ListItem',
      position: 2,
      name: 'Products',
      item: 'https://example.com/products',
    },
    {
      '@type': 'ListItem',
      position: 3,
      name: 'Product Name',
    },
  ],
}

// FAQ
const faqSchema = {
  '@context': 'https://schema.org',
  '@type': 'FAQPage',
  mainEntity: [
    {
      '@type': 'Question',
      name: 'What is this product?',
      acceptedAnswer: {
        '@type': 'Answer',
        text: 'This product is...',
      },
    },
  ],
}
```

## Sitemap

```typescript
// app/sitemap.ts
import type { MetadataRoute } from 'next'

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const baseUrl = 'https://example.com'

  // Static pages
  const staticPages = [
    '',
    '/about',
    '/contact',
    '/pricing',
  ].map(route => ({
    url: `${baseUrl}${route}`,
    lastModified: new Date(),
    changeFrequency: 'monthly' as const,
    priority: route === '' ? 1 : 0.8,
  }))

  // Dynamic pages from database
  const posts = await db.post.findMany({
    select: { slug: true, updatedAt: true },
  })

  const postPages = posts.map(post => ({
    url: `${baseUrl}/blog/${post.slug}`,
    lastModified: post.updatedAt,
    changeFrequency: 'weekly' as const,
    priority: 0.6,
  }))

  return [...staticPages, ...postPages]
}
```

### Multiple Sitemaps (Large Sites)

```typescript
// app/sitemap.ts
import type { MetadataRoute } from 'next'

export async function generateSitemaps() {
  const totalProducts = await db.product.count()
  const sitemapsNeeded = Math.ceil(totalProducts / 50000)
  
  return Array.from({ length: sitemapsNeeded }, (_, i) => ({ id: i }))
}

export default async function sitemap({
  id,
}: {
  id: number
}): Promise<MetadataRoute.Sitemap> {
  const start = id * 50000
  const products = await db.product.findMany({
    skip: start,
    take: 50000,
    select: { slug: true, updatedAt: true },
  })

  return products.map(product => ({
    url: `https://example.com/products/${product.slug}`,
    lastModified: product.updatedAt,
  }))
}
```

## Robots.txt

```typescript
// app/robots.ts
import type { MetadataRoute } from 'next'

export default function robots(): MetadataRoute.Robots {
  const baseUrl = 'https://example.com'

  return {
    rules: [
      {
        userAgent: '*',
        allow: '/',
        disallow: ['/api/', '/admin/', '/private/'],
      },
      {
        userAgent: 'Googlebot',
        allow: '/',
      },
    ],
    sitemap: `${baseUrl}/sitemap.xml`,
  }
}
```

## Canonical URLs

```typescript
// lib/seo/canonical.ts
export function getCanonicalUrl(path: string): string {
  const baseUrl = process.env.NEXT_PUBLIC_SITE_URL || 'https://example.com'
  
  // Remove trailing slash
  const cleanPath = path.replace(/\/$/, '')
  
  // Remove query params for canonical
  const pathWithoutQuery = cleanPath.split('?')[0]
  
  return `${baseUrl}${pathWithoutQuery}`
}

// Usage in generateMetadata
export async function generateMetadata({ params }): Promise<Metadata> {
  return {
    alternates: {
      canonical: getCanonicalUrl(`/blog/${params.slug}`),
    },
  }
}
```

## Viewport Configuration

```typescript
// app/layout.tsx
import type { Viewport } from 'next'

export const viewport: Viewport = {
  width: 'device-width',
  initialScale: 1,
  maximumScale: 5,
  themeColor: [
    { media: '(prefers-color-scheme: light)', color: '#ffffff' },
    { media: '(prefers-color-scheme: dark)', color: '#0a0a0a' },
  ],
}
```

## Performance SEO

### Font Optimization

```typescript
import { Inter } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap', // Prevents FOIT
  preload: true,
})
```

### Image Optimization

```typescript
import Image from 'next/image'

<Image
  src="/hero.jpg"
  alt="Descriptive alt text for SEO"
  width={1200}
  height={600}
  priority // LCP image
  sizes="(max-width: 768px) 100vw, 1200px"
/>
```

### Core Web Vitals

```typescript
// Measure and report CWV
export function reportWebVitals(metric: NextWebVitalsMetric) {
  if (metric.label === 'web-vital') {
    // Send to analytics
    analytics.track('Web Vital', {
      name: metric.name,    // CLS, FID, FCP, LCP, TTFB
      value: metric.value,
      rating: metric.rating, // good, needs-improvement, poor
    })
  }
}
```

## SEO Checklist

- [ ] Unique title and description per page
- [ ] Open Graph and Twitter cards configured
- [ ] Structured data for relevant content types
- [ ] Sitemap generated and submitted
- [ ] Robots.txt configured
- [ ] Canonical URLs set
- [ ] Images optimized with alt text
- [ ] Core Web Vitals passing
- [ ] Mobile-friendly design
- [ ] HTTPS enabled
