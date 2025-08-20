---
trigger: glob
description: Comprehensive rules for Next.js with TypeScript frontend development using App Router, covering configuration, file structure, components, data fetching, performance optimization, and common pitfalls
globs: [
  "**/*.ts",
  "**/*.tsx", 
  "**/next.config.*",
  "**/tsconfig.json",
  "**/tailwind.config.*",
  "**/app/**/*",
  "**/src/app/**/*",
  "**/.next/**/*",
  "**/package.json"
]
---

# Next.js + TypeScript Frontend Rules

## Project Configuration

### TypeScript Configuration
- Use `next.config.ts` instead of `next.config.js` for type safety and IDE support
- Configure TypeScript with strict mode enabled in `tsconfig.json`:
  ```json
  {
    "compilerOptions": {
      "strict": true,
      "skipLibCheck": true,
      "noImplicitAny": true,
      "strictNullChecks": true,
      "strictPropertyInitialization": true
    },
    "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"]
  }
  ```

### Next.js Configuration Best Practices
- Always type your Next.js config file:
  ```typescript
  import type { NextConfig } from 'next'
  
  const nextConfig: NextConfig = {
    experimental: {
      typedRoutes: true, // Enable type-safe routing
    },
    typescript: {
      ignoreBuildErrors: false, // Never disable unless absolutely necessary
    }
  }
  
  export default nextConfig
  ```

### ESLint Integration
- Extend Next.js TypeScript ESLint configuration:
  ```javascript
  const eslintConfig = [
    ...compat.config({
      extends: ['next/core-web-vitals', 'next/typescript'],
    }),
  ]
  ```

## File Structure and Routing

### App Router Structure
- Use the `app` directory for all new projects (App Router is recommended over Pages Router)
- Follow this structure:
  ```
  app/
  ├── layout.tsx          # Root layout (required)
  ├── page.tsx            # Home page
  ├── loading.tsx         # Loading UI
  ├── error.tsx           # Error UI
  ├── not-found.tsx       # 404 page
  ├── globals.css         # Global styles
  └── [folder]/
      ├── layout.tsx      # Nested layout
      ├── page.tsx        # Route page
      └── loading.tsx     # Route-specific loading
  ```

### Metadata Configuration
- Define metadata in layouts and pages using the Metadata API:
  ```typescript
  import type { Metadata } from 'next'
  
  export const metadata: Metadata = {
    title: 'Page Title',
    description: 'Page description',
  }
  ```

## Component Architecture

### Server vs Client Components
- Default to Server Components for better performance
- Only use `'use client'` when you need:
  - Browser APIs (localStorage, sessionStorage)
  - Event handlers (onClick, onChange)
  - React hooks (useState, useEffect, useContext)
  - Interactive features

#### Server Component Example:
```typescript
// No 'use client' directive - this is a Server Component
async function ServerComponent() {
  const data = await fetch('https://api.example.com/data', {
    next: { revalidate: 3600 } // Cache for 1 hour
  })
  
  return <div>{/* Render data */}</div>
}
```

#### Client Component Example:
```typescript
'use client'

import { useState } from 'react'

export default function ClientComponent() {
  const [count, setCount] = useState(0)
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  )
}
```

### Component Patterns
- Use proper TypeScript interfaces for props:
  ```typescript
  interface PageProps {
    params: Promise<{ id: string }>
    searchParams: Promise<{ [key: string]: string | string[] | undefined }>
  }
  
  export default async function Page({ params, searchParams }: PageProps) {
    const { id } = await params
    // Component logic
  }
  ```

## Data Fetching

### Server Component Data Fetching
- Fetch data directly in Server Components using async/await:
  ```typescript
  async function getData() {
    const res = await fetch('https://api.example.com/data', {
      cache: 'force-cache', // Static generation (like getStaticProps)
      // OR
      cache: 'no-store',   // Server-side rendering (like getServerSideProps)
      // OR
      next: { revalidate: 10 } // Incremental Static Regeneration
    })
    
    if (!res.ok) {
      throw new Error('Failed to fetch data')
    }
    
    return res.json()
  }
  
  export default async function Page() {
    const data = await getData()
    return <div>{/* Use data */}</div>
  }
  ```

### Parallel Data Fetching
- Use Promise.all for independent data fetching:
  ```typescript
  async function getPost(id: string) {
    const res = await fetch(`/api/posts/${id}`)
    return res.json()
  }
  
  async function getComments(id: string) {
    const res = await fetch(`/api/posts/${id}/comments`)
    return res.json()
  }
  
  export default async function PostPage({ params }: { params: Promise<{ id: string }> }) {
    const { id } = await params
    
    // Fetch in parallel
    const [post, comments] = await Promise.all([
      getPost(id),
      getComments(id)
    ])
    
    return (
      <div>
        <h1>{post.title}</h1>
        <Comments comments={comments} />
      </div>
    )
  }
  ```

### Route Handlers (API Routes)
- Create type-safe API routes:
  ```typescript
  // app/api/users/route.ts
  import { NextRequest, NextResponse } from 'next/server'
  
  export async function GET(request: NextRequest) {
    try {
      const users = await fetchUsers()
      return NextResponse.json(users)
    } catch (error) {
      return NextResponse.json(
        { error: 'Failed to fetch users' },
        { status: 500 }
      )
    }
  }
  
  export async function POST(request: NextRequest) {
    const body = await request.json()
    // Handle POST request
    return NextResponse.json({ success: true })
  }
  ```

## Navigation and Routing

### Type-Safe Navigation
- Use Next.js navigation hooks with proper typing:
  ```typescript
  'use client'
  
  import { useRouter, usePathname, useSearchParams } from 'next/navigation'
  
  export default function NavigationComponent() {
    const router = useRouter()
    const pathname = usePathname()
    const searchParams = useSearchParams()
    
    return (
      <button onClick={() => router.push('/dashboard')}>
        Go to Dashboard
      </button>
    )
  }
  ```

### Link Component Usage
- Use Next.js Link with proper prefetching:
  ```typescript
  import Link from 'next/link'
  
  export default function Navigation() {
    return (
      <nav>
        <Link href="/dashboard" prefetch={true}>
          Dashboard
        </Link>
        <Link href="/profile" prefetch={false}>
          Profile
        </Link>
      </nav>
    )
  }
  ```

## Performance Optimization

### Image Optimization
- Always use `next/image` for optimal performance:
  ```typescript
  import Image from 'next/image'
  
  export default function OptimizedImage() {
    return (
      <Image
        src="/hero-image.jpg"
        alt="Hero image"
        width={800}
        height={600}
        priority // Use for above-the-fold images
        placeholder="blur"
        blurDataURL="data:image/jpeg;base64,..."
      />
    )
  }
  ```

### Font Optimization
- Use `next/font` for self-hosted fonts:
  ```typescript
  import { Inter } from 'next/font/google'
  
  const inter = Inter({
    subsets: ['latin'],
    display: 'swap',
  })
  
  export default function RootLayout({
    children,
  }: {
    children: React.ReactNode
  }) {
    return (
      <html lang="en" className={inter.className}>
        <body>{children}</body>
      </html>
    )
  }
  ```

### Dynamic Imports and Code Splitting
- Use dynamic imports for large components:
  ```typescript
  import dynamic from 'next/dynamic'
  
  const DynamicComponent = dynamic(() => import('./HeavyComponent'), {
    loading: () => <p>Loading...</p>,
    ssr: false // Disable server-side rendering if needed
  })
  
  export default function Page() {
    return (
      <div>
        <h1>My Page</h1>
        <DynamicComponent />
      </div>
    )
  }
  ```

### Script Optimization
- Use `next/script` for third-party scripts:
  ```typescript
  import Script from 'next/script'
  
  export default function Layout() {
    return (
      <>
        <Script
          src="https://www.googletagmanager.com/gtag/js?id=GA_MEASUREMENT_ID"
          strategy="afterInteractive"
        />
        <Script id="google-analytics" strategy="afterInteractive">
          {`
            window.dataLayer = window.dataLayer || [];
            function gtag(){dataLayer.push(arguments);}
            gtag('js', new Date());
            gtag('config', 'GA_MEASUREMENT_ID');
          `}
        </Script>
      </>
    )
  }
  ```

## Error Handling and Loading States

### Error Boundaries
- Create error.tsx files for graceful error handling:
  ```typescript
  'use client'
  
  export default function Error({
    error,
    reset,
  }: {
    error: Error & { digest?: string }
    reset: () => void
  }) {
    return (
      <div>
        <h2>Something went wrong!</h2>
        <button onClick={() => reset()}>
          Try again
        </button>
      </div>
    )
  }
  ```

### Loading States
- Create loading.tsx files for better UX:
  ```typescript
  export default function Loading() {
    return (
      <div className="animate-pulse">
        <div className="h-4 bg-gray-200 rounded w-3/4 mb-4"></div>
        <div className="h-4 bg-gray-200 rounded w-1/2"></div>
      </div>
    )
  }
  ```

### Suspense Boundaries
- Use Suspense for streaming and better loading experiences:
  ```typescript
  import { Suspense } from 'react'
  
  export default function Page() {
    return (
      <div>
        <h1>Dashboard</h1>
        <Suspense fallback={<div>Loading analytics...</div>}>
          <Analytics />
        </Suspense>
        <Suspense fallback={<div>Loading recent posts...</div>}>
          <RecentPosts />
        </Suspense>
      </div>
    )
  }
  ```

## Known Issues and Mitigations

### Common TypeScript Build Errors
- **Issue**: Build fails on TypeScript errors in production
  **Solution**: Fix all TypeScript errors before deployment. Use `tsc --noEmit` in CI/CD pipeline
  **Anti-pattern**: Setting `ignoreBuildErrors: true` in production

- **Issue**: Custom types getting overwritten in `next-env.d.ts`
  **Solution**: Create separate type declaration files (e.g., `types/global.d.ts`) and include them in `tsconfig.json`

- **Issue**: Hydration mismatches between server and client
  **Solution**: Ensure server and client render identical content initially. Avoid using `Math.random()`, `Date.now()`, or browser-only APIs in initial render

### Performance Anti-Patterns
- **Avoid**: Large client-side bundles with unnecessary `'use client'` directives
- **Avoid**: Blocking the main thread with heavy computations
- **Avoid**: Not optimizing images and fonts
- **Avoid**: Excessive API calls in client components

### Memory Leaks and Resource Management
- Always cleanup subscriptions and event listeners in `useEffect`:
  ```typescript
  'use client'
  
  import { useEffect } from 'react'
  
  export default function Component() {
    useEffect(() => {
      const handleResize = () => {
        // Handle resize
      }
      
      window.addEventListener('resize', handleResize)
      
      return () => {
        window.removeEventListener('resize', handleResize)
      }
    }, [])
    
    return <div>Component</div>
  }
  ```

## Development and Production Best Practices

### Environment Variables
- Use Next.js environment variable conventions:
  ```typescript
  // .env.local
  NEXT_PUBLIC_API_URL=https://api.example.com
  DATABASE_URL=postgres://...
  
  // In components
  const apiUrl = process.env.NEXT_PUBLIC_API_URL
  ```

### Type Safety for Environment Variables
- Create type-safe environment variable access:
  ```typescript
  // lib/env.ts
  export const env = {
    API_URL: process.env.NEXT_PUBLIC_API_URL!,
    DATABASE_URL: process.env.DATABASE_URL!,
  }
  
  // Validate at runtime
  Object.entries(env).forEach(([key, value]) => {
    if (!value) {
      throw new Error(`Missing environment variable: ${key}`)
    }
  })
  ```

### Testing Configuration
- Add TypeScript type checking to package.json scripts:
  ```json
  {
    "scripts": {
      "type-check": "tsc --noEmit",
      "build": "npm run type-check && next build",
      "test": "jest",
      "test:watch": "jest --watch"
    }
  }
  ```

### Bundle Analysis
- Analyze bundle size regularly:
  ```bash
  # Install bundle analyzer
  npm install @next/bundle-analyzer
  
  # In next.config.ts
  const withBundleAnalyzer = require('@next/bundle-analyzer')({
    enabled: process.env.ANALYZE === 'true',
  })
  
  export default withBundleAnalyzer(nextConfig)
  ```

## Deployment and Production Readiness

### Build Optimization
- Ensure proper build configuration:
  ```typescript
  // next.config.ts
  const nextConfig: NextConfig = {
    output: 'standalone', // For Docker deployments
    images: {
      domains: ['example.com'],
      formats: ['image/webp', 'image/avif'],
    },
    experimental: {
      optimizeCss: true,
    }
  }
  ```

### Monitoring and Analytics
- Implement Web Vitals monitoring:
  ```typescript
  // app/layout.tsx
  import { Analytics } from '@vercel/analytics/react'
  import { SpeedInsights } from '@vercel/speed-insights/next'
  
  export default function RootLayout({
    children,
  }: {
    children: React.ReactNode
  }) {
    return (
      <html lang="en">
        <body>
          {children}
          <Analytics />
          <SpeedInsights />
        </body>
      </html>
    )
  }
  ```

### Security Headers
- Configure security headers in next.config.ts:
  ```typescript
  const nextConfig: NextConfig = {
    async headers() {
      return [
        {
          source: '/(.*)',
          headers: [
            {
              key: 'X-Frame-Options',
              value: 'DENY',
            },
            {
              key: 'X-Content-Type-Options',
              value: 'nosniff',
            },
            {
              key: 'Referrer-Policy',
              value: 'origin-when-cross-origin',
            },
          ],
        },
      ]
    },
  }
  ```

This ruleset covers the essential patterns, best practices, and common pitfalls when building production-ready Next.js applications with TypeScript using the App Router architecture. Always prioritize type safety, performance, and maintainability in your implementation decisions.