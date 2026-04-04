---
name: nextjs-developer
description: Next.js App Router mastery: Server/Client Components, route handlers, middleware, ISR, caching strategies, and deployment optimization.
---

# Next.js Developer

Expert-level Next.js development skill covering App Router architecture, advanced data fetching, the four-layer caching system, Server Actions, middleware patterns, performance optimization, and production deployment. Targets Next.js 14 and 15 patterns with emphasis on Server Components as the default rendering model.

## Table of Contents

- [App Router Architecture](#app-router-architecture)
- [Data Fetching Patterns](#data-fetching-patterns)
- [Caching Deep Dive](#caching-deep-dive)
- [Server Actions](#server-actions)
- [Middleware Patterns](#middleware-patterns)
- [Performance and Optimization](#performance-and-optimization)
- [Deployment Strategies](#deployment-strategies)
- [Common Patterns Quick Reference](#common-patterns-quick-reference)

---

## App Router Architecture

The App Router (introduced in Next.js 13, stable in 14+) uses the `app/` directory with file-system conventions for routing, layouts, and rendering boundaries.

### File Conventions

```
app/
  layout.tsx          # Root layout (required, wraps entire app)
  page.tsx            # Home route "/"
  loading.tsx         # Instant loading UI (Suspense boundary)
  error.tsx           # Error boundary (must be Client Component)
  not-found.tsx       # 404 UI
  global-error.tsx    # Root-level error boundary
  template.tsx        # Re-mounts on navigation (unlike layout)

  dashboard/
    layout.tsx        # Nested layout for /dashboard/*
    page.tsx          # /dashboard
    loading.tsx       # Loading state for /dashboard

    settings/
      page.tsx        # /dashboard/settings

  blog/
    [slug]/
      page.tsx        # /blog/:slug (dynamic segment)
    [...catchAll]/
      page.tsx        # /blog/* (catch-all segment)
    [[...optional]]/
      page.tsx        # /blog or /blog/* (optional catch-all)
```

### Layouts vs Templates

Layouts persist across navigations and preserve state. Templates remount on every navigation, useful for enter/exit animations or per-page effects.

```tsx
// app/layout.tsx — Root layout (Server Component by default)
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <nav>/* persists across all routes */</nav>
        {children}
      </body>
    </html>
  );
}

// app/dashboard/template.tsx — Remounts on each navigation
export default function DashboardTemplate({ children }: { children: React.ReactNode }) {
  return <div className="fade-in">{children}</div>;
}
```

### Route Groups

Route groups `(folderName)` organize routes without affecting the URL path. Useful for separate layouts or logical grouping.

```
app/
  (marketing)/
    layout.tsx        # Marketing-specific layout
    about/page.tsx    # /about
    pricing/page.tsx  # /pricing

  (app)/
    layout.tsx        # App-specific layout (with sidebar, auth)
    dashboard/page.tsx  # /dashboard
    settings/page.tsx   # /settings
```

### Parallel Routes

Parallel routes render multiple pages simultaneously in the same layout using named slots (`@slotName`).

```
app/
  layout.tsx          # Receives { children, analytics, team } props
  page.tsx
  @analytics/
    page.tsx          # Rendered in the analytics slot
    default.tsx       # Fallback when no match
  @team/
    page.tsx          # Rendered in the team slot
    default.tsx
```

```tsx
// app/layout.tsx
export default function Layout({
  children,
  analytics,
  team,
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
}) {
  return (
    <div>
      {children}
      <div className="grid grid-cols-2">
        {analytics}
        {team}
      </div>
    </div>
  );
}
```

### Intercepting Routes

Intercept routes to show a modal overlay while preserving the URL. Uses `(.)`, `(..)`, `(..)(..)`, or `(...)` conventions.

```
app/
  feed/
    page.tsx              # Main feed
    @modal/
      (.)photo/[id]/
        page.tsx          # Intercepts /photo/:id from /feed context (shows modal)
      default.tsx
  photo/
    [id]/
      page.tsx            # Full page view (direct navigation or refresh)
```

Architecture decision: Intercepting routes are ideal for photo galleries, login modals, and cart previews where the modal shares a URL with the full-page version.

---

## Data Fetching Patterns

Server Components fetch data directly without client-side state management or useEffect.

### Server Component Data Fetching

```tsx
// app/posts/page.tsx — Server Component (default)
async function getPosts() {
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 3600 }, // ISR: revalidate every hour
  });
  if (!res.ok) throw new Error('Failed to fetch posts');
  return res.json() as Promise<Post[]>;
}

export default async function PostsPage() {
  const posts = await getPosts();

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### Fetch Caching and Revalidation

```tsx
// No cache — fresh data every request
fetch(url, { cache: 'no-store' });

// Cache indefinitely (default in Next.js 14, opt-in in Next.js 15)
fetch(url, { cache: 'force-cache' });

// Time-based revalidation (ISR)
fetch(url, { next: { revalidate: 60 } });

// Tag-based revalidation
fetch(url, { next: { tags: ['posts'] } });
// Then invalidate from a Server Action or Route Handler:
// revalidateTag('posts');
```

**Next.js 15 breaking change:** `fetch` requests are no longer cached by default. You must explicitly opt in with `cache: 'force-cache'` or `next: { revalidate: N }`.

### generateStaticParams

Pre-render dynamic routes at build time.

```tsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then((r) => r.json());

  return posts.map((post: Post) => ({
    slug: post.slug,
  }));
}

// dynamicParams = true (default): unknown slugs render on-demand and cache
// dynamicParams = false: unknown slugs return 404
export const dynamicParams = true;

export default async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params; // Next.js 15: params is async
  const post = await getPost(slug);
  return <article>{post.content}</article>;
}
```

### Streaming with Suspense

Stream UI progressively using Suspense boundaries.

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      {/* Instant: static content renders immediately */}
      <Suspense fallback={<RevenueChartSkeleton />}>
        <RevenueChart /> {/* Streams in when data resolves */}
      </Suspense>
      <Suspense fallback={<ActivityFeedSkeleton />}>
        <ActivityFeed /> {/* Streams independently */}
      </Suspense>
    </div>
  );
}

// Each component fetches its own data — no waterfall
async function RevenueChart() {
  const data = await getRevenue(); // This can take 2 seconds
  return <Chart data={data} />;
}
```

Architecture: Place Suspense boundaries around slow data fetches. Each boundary streams independently, so fast sections appear immediately while slow ones show skeletons.

### Route Handlers

```tsx
// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = searchParams.get('page') ?? '1';

  const posts = await db.post.findMany({
    skip: (parseInt(page) - 1) * 10,
    take: 10,
  });

  return NextResponse.json(posts);
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  const post = await db.post.create({ data: body });

  return NextResponse.json(post, { status: 201 });
}
```

---

## Caching Deep Dive

Next.js has four caching layers. Understanding when each applies (and how to opt out) is critical for correctness.

### Layer 1: Request Memoization

**What:** Deduplicates identical `fetch` calls within a single server render pass.

**When it applies:** Same URL + same options in the same render tree. Automatic.

**How to opt out:** Use `AbortController` signal or different fetch options.

```tsx
// Both components share the same render — fetch is called once
async function Header() {
  const user = await fetch('/api/user').then((r) => r.json());
  return <h1>{user.name}</h1>;
}

async function Sidebar() {
  const user = await fetch('/api/user').then((r) => r.json());
  return <p>{user.email}</p>;
}
```

For non-fetch data (e.g., direct DB calls), use React's `cache()`:

```tsx
import { cache } from 'react';

export const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } });
});
```

### Layer 2: Data Cache

**What:** Persists fetch responses across requests and deployments on the server.

**When it applies:** `fetch` calls with `cache: 'force-cache'` or `next: { revalidate: N }`.

**How to opt out:** `cache: 'no-store'` or `export const dynamic = 'force-dynamic'` on the page.

**Invalidation:**

```tsx
import { revalidatePath, revalidateTag } from 'next/cache';

// Invalidate by tag (surgical)
revalidateTag('posts');

// Invalidate by path (broader)
revalidatePath('/blog');

// Invalidate layout + all child pages
revalidatePath('/blog', 'layout');
```

### Layer 3: Full Route Cache

**What:** Caches the rendered HTML and RSC payload of static routes at build time.

**When it applies:** Routes with no dynamic functions (`cookies()`, `headers()`, `searchParams`) and all cached data fetches.

**How to opt out:**
- Use a dynamic function in the route
- Set `export const dynamic = 'force-dynamic'`
- Set `export const revalidate = 0`
- Use `cache: 'no-store'` on any fetch in the route

### Layer 4: Router Cache (Client-side)

**What:** In-memory client-side cache of RSC payloads for visited and prefetched routes.

**When it applies:** Automatic on all client-side navigations.

**Duration (Next.js 15):** Dynamic pages = 0 seconds (no caching), static pages = 5 minutes. This changed from Next.js 14 where dynamic pages cached for 30 seconds.

**How to opt out:**
```tsx
// Programmatic refresh
import { useRouter } from 'next/navigation';
const router = useRouter();
router.refresh(); // Invalidates Router Cache for current route

// Disable prefetching
<Link href="/dashboard" prefetch={false}>Dashboard</Link>

// Next.js 15 next.config.js — opt out entirely
const nextConfig = {
  experimental: {
    staleTimes: {
      dynamic: 0,  // Already default in Next.js 15
      static: 0,   // Disable static cache too
    },
  },
};
```

### Cache Decision Flowchart (Text)

```
Is the route fully static (no dynamic functions)?
  YES -> Full Route Cache applies (HTML + RSC payload cached at build time)
  NO  -> Rendered on every request

Does the fetch use cache: 'force-cache' or next.revalidate?
  YES -> Data Cache stores the response
  NO  -> Fresh fetch on every request

Are there duplicate fetches in the same render?
  YES -> Request Memoization deduplicates them

Is the user navigating client-side?
  YES -> Router Cache may serve a stale RSC payload
```

---

## Server Actions

Server Actions are async functions that run on the server, callable from Client and Server Components. They replace API routes for mutations.

### Basic Form Handling

```tsx
// app/posts/new/page.tsx — Server Component
import { redirect } from 'next/navigation';
import { revalidatePath } from 'next/cache';

export default function NewPostPage() {
  async function createPost(formData: FormData) {
    'use server';

    const title = formData.get('title') as string;
    const content = formData.get('content') as string;

    await db.post.create({ data: { title, content } });
    revalidatePath('/posts');
    redirect('/posts');
  }

  return (
    <form action={createPost}>
      <input name="title" required />
      <textarea name="content" required />
      <button type="submit">Create Post</button>
    </form>
  );
}
```

### useActionState (Next.js 15 / React 19)

Manages form state, pending status, and progressive enhancement.

```tsx
'use client';

import { useActionState } from 'react';
import { createPost } from '@/app/actions';

export function PostForm() {
  const [state, formAction, isPending] = useActionState(createPost, {
    errors: {},
    message: '',
  });

  return (
    <form action={formAction}>
      <input name="title" />
      {state.errors?.title && <p className="text-red-500">{state.errors.title}</p>}

      <textarea name="content" />
      {state.errors?.content && <p className="text-red-500">{state.errors.content}</p>}

      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Post'}
      </button>

      {state.message && <p>{state.message}</p>}
    </form>
  );
}
```

```tsx
// app/actions.ts
'use server';

import { z } from 'zod';
import { revalidatePath } from 'next/cache';

const PostSchema = z.object({
  title: z.string().min(1, 'Title is required').max(200),
  content: z.string().min(1, 'Content is required'),
});

export async function createPost(prevState: any, formData: FormData) {
  const parsed = PostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  });

  if (!parsed.success) {
    return { errors: parsed.error.flatten().fieldErrors, message: '' };
  }

  await db.post.create({ data: parsed.data });
  revalidatePath('/posts');

  return { errors: {}, message: 'Post created successfully' };
}
```

### Optimistic Updates

```tsx
'use client';

import { useOptimistic } from 'react';
import { toggleLike } from '@/app/actions';

export function LikeButton({ liked, count }: { liked: boolean; count: number }) {
  const [optimistic, setOptimistic] = useOptimistic(
    { liked, count },
    (current, newLiked: boolean) => ({
      liked: newLiked,
      count: current.count + (newLiked ? 1 : -1),
    })
  );

  return (
    <form
      action={async () => {
        setOptimistic(!optimistic.liked);
        await toggleLike();
      }}
    >
      <button type="submit">
        {optimistic.liked ? 'Unlike' : 'Like'} ({optimistic.count})
      </button>
    </form>
  );
}
```

### Security Considerations

- Server Actions create public HTTP endpoints. Always validate input and check authentication.
- Use `zod` or similar for input validation on every action.
- Check session/auth in every action, not just in middleware.
- Use CSRF protection (Next.js includes it by default for Server Actions).
- Never pass sensitive data (IDs of resources the user should not access) solely from the client.

---

## Middleware Patterns

Middleware runs before every request at the edge. It can rewrite, redirect, set headers, and modify the response.

### Basic Structure

```tsx
// middleware.ts (project root, next to app/)
import { NextRequest, NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  // Runs on every matched path
  return NextResponse.next();
}

export const config = {
  matcher: [
    // Match all paths except static files and _next
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
};
```

### Authentication Guard

```tsx
export function middleware(request: NextRequest) {
  const token = request.cookies.get('session-token')?.value;
  const isAuthPage = request.nextUrl.pathname.startsWith('/login');

  if (!token && !isAuthPage) {
    const loginUrl = new URL('/login', request.url);
    loginUrl.searchParams.set('callbackUrl', request.nextUrl.pathname);
    return NextResponse.redirect(loginUrl);
  }

  if (token && isAuthPage) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }

  return NextResponse.next();
}
```

### Internationalization (i18n)

```tsx
const locales = ['en', 'de', 'fr', 'ja'];
const defaultLocale = 'en';

function getLocale(request: NextRequest): string {
  const acceptLang = request.headers.get('accept-language') ?? '';
  const preferred = acceptLang.split(',')[0]?.split('-')[0];
  return locales.includes(preferred ?? '') ? preferred! : defaultLocale;
}

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const hasLocale = locales.some((l) => pathname.startsWith(`/${l}/`) || pathname === `/${l}`);

  if (hasLocale) return NextResponse.next();

  const locale = getLocale(request);
  return NextResponse.redirect(new URL(`/${locale}${pathname}`, request.url));
}
```

### A/B Testing via Cookie

```tsx
export function middleware(request: NextRequest) {
  const bucket = request.cookies.get('ab-bucket')?.value;
  const response = NextResponse.next();

  if (!bucket) {
    const assigned = Math.random() < 0.5 ? 'control' : 'variant';
    response.cookies.set('ab-bucket', assigned, { maxAge: 60 * 60 * 24 * 30 });
  }

  // Rewrite to variant page
  if ((bucket ?? '') === 'variant' && request.nextUrl.pathname === '/pricing') {
    return NextResponse.rewrite(new URL('/pricing-variant', request.url));
  }

  return response;
}
```

### Rate Limiting (Simple Token Bucket)

```tsx
const rateLimit = new Map<string, { count: number; lastReset: number }>();

export function middleware(request: NextRequest) {
  if (!request.nextUrl.pathname.startsWith('/api/')) {
    return NextResponse.next();
  }

  const ip = request.headers.get('x-forwarded-for') ?? 'unknown';
  const now = Date.now();
  const window = 60_000; // 1 minute
  const maxRequests = 60;

  const entry = rateLimit.get(ip);
  if (!entry || now - entry.lastReset > window) {
    rateLimit.set(ip, { count: 1, lastReset: now });
  } else if (entry.count >= maxRequests) {
    return NextResponse.json({ error: 'Rate limit exceeded' }, { status: 429 });
  } else {
    entry.count++;
  }

  return NextResponse.next();
}
```

Note: In-memory rate limiting only works for single-instance deployments. For production, use an external store like Redis or Upstash.

### Geo-Based Routing

```tsx
export function middleware(request: NextRequest) {
  const country = request.geo?.country ?? 'US';

  // Redirect EU users to GDPR-compliant page
  const euCountries = ['DE', 'FR', 'IT', 'ES', 'NL', 'PL', 'BE', 'SE', 'AT'];
  if (euCountries.includes(country) && request.nextUrl.pathname === '/') {
    return NextResponse.rewrite(new URL('/eu-home', request.url));
  }

  const response = NextResponse.next();
  response.headers.set('x-user-country', country);
  return response;
}
```

---

## Performance and Optimization

### Image Component

```tsx
import Image from 'next/image';

// Static import — automatically gets width/height
import heroImage from '@/public/hero.jpg';

export function Hero() {
  return (
    <Image
      src={heroImage}
      alt="Hero banner"
      placeholder="blur"           // Blur-up on load
      priority                      // Preload (above-the-fold images)
      sizes="(max-width: 768px) 100vw, 50vw"
    />
  );
}

// Remote image — must specify dimensions
export function Avatar({ url }: { url: string }) {
  return (
    <Image
      src={url}
      alt="User avatar"
      width={48}
      height={48}
      className="rounded-full"
    />
  );
}
```

Configure remote patterns in `next.config.js`:

```js
const nextConfig = {
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'cdn.example.com', pathname: '/avatars/**' },
      { protocol: 'https', hostname: '*.githubusercontent.com' },
    ],
  },
};
```

### Font Optimization

```tsx
// app/layout.tsx
import { Inter, JetBrains_Mono } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
});

const jetbrainsMono = JetBrains_Mono({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-mono',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${inter.variable} ${jetbrainsMono.variable}`}>
      <body className="font-sans">{children}</body>
    </html>
  );
}
```

Tailwind CSS integration in `tailwind.config.ts`:

```ts
const config = {
  theme: {
    extend: {
      fontFamily: {
        sans: ['var(--font-inter)', 'system-ui', 'sans-serif'],
        mono: ['var(--font-mono)', 'monospace'],
      },
    },
  },
};
```

### Script Loading Strategies

```tsx
import Script from 'next/script';

// afterInteractive (default) — loads after page hydration
<Script src="https://analytics.example.com/script.js" strategy="afterInteractive" />

// lazyOnload — loads during idle time
<Script src="https://chat-widget.example.com/widget.js" strategy="lazyOnload" />

// beforeInteractive — loads before hydration (use sparingly)
<Script src="/polyfill.js" strategy="beforeInteractive" />

// Inline script with onLoad callback
<Script
  src="https://maps.googleapis.com/maps/api/js?key=KEY"
  strategy="lazyOnload"
  onLoad={() => console.log('Google Maps loaded')}
/>
```

### Bundle Analysis

```bash
# Install the analyzer
npm install @next/bundle-analyzer

# next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});
module.exports = withBundleAnalyzer(nextConfig);

# Run analysis
ANALYZE=true npm run build
```

Key strategies for reducing bundle size:
- Use dynamic imports for heavy components: `const Chart = dynamic(() => import('./Chart'), { ssr: false })`
- Move large dependencies to Server Components (they have zero client bundle cost)
- Use `next/dynamic` with `ssr: false` for client-only libraries (maps, rich text editors)
- Check for barrel file re-exports that pull in entire libraries

### Partial Prerendering (Next.js 15 — Experimental)

Combines static shell with dynamic holes. The static parts are served from CDN, while dynamic parts stream in.

```tsx
// next.config.js
const nextConfig = {
  experimental: {
    ppr: true, // Enable Partial Prerendering
  },
};
```

```tsx
// app/product/[id]/page.tsx
import { Suspense } from 'react';

export default async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const product = await getProduct(id); // Static (cached)

  return (
    <div>
      {/* Static shell — prerendered at build time */}
      <h1>{product.name}</h1>
      <p>{product.description}</p>

      {/* Dynamic hole — streams in at request time */}
      <Suspense fallback={<PriceSkeleton />}>
        <DynamicPrice productId={id} /> {/* Uses cookies() or real-time data */}
      </Suspense>

      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews productId={id} />
      </Suspense>
    </div>
  );
}
```

Architecture: PPR gives you the speed of static with the freshness of dynamic. The static shell arrives instantly from the CDN, while dynamic Suspense boundaries stream in from the server.

---

## Deployment Strategies

### Vercel (Recommended)

Zero-config deployment with full feature support.

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel

# Production deploy
vercel --prod
```

Key `vercel.json` settings:

```json
{
  "framework": "nextjs",
  "regions": ["iad1", "sfo1"],
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "s-maxage=60, stale-while-revalidate=300" }
      ]
    }
  ]
}
```

### Self-Hosted with Docker

```dockerfile
# Dockerfile
FROM node:20-alpine AS base

FROM base AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000
CMD ["node", "server.js"]
```

Required `next.config.js` for standalone output:

```js
const nextConfig = {
  output: 'standalone', // Produces minimal server bundle
};
```

### Static Export

For sites that need no server runtime (no Server Actions, no middleware, no ISR).

```js
// next.config.js
const nextConfig = {
  output: 'export',
  // Optional: trailing slashes for static hosts
  trailingSlash: true,
  // Optional: custom base path
  basePath: '/docs',
};
```

```bash
npm run build   # Generates out/ directory
# Deploy out/ to any static host (S3, Cloudflare Pages, GitHub Pages)
```

### Edge Runtime vs Node.js Runtime

```tsx
// Per-route runtime selection
export const runtime = 'edge';   // Fast cold starts, limited API surface
// OR
export const runtime = 'nodejs'; // Full Node.js APIs, heavier cold starts

// Edge-compatible route handler
export const runtime = 'edge';

export async function GET(request: NextRequest) {
  // Cannot use: fs, child_process, native Node modules
  // Can use: fetch, crypto, TextEncoder, URL, Headers
  return new Response(JSON.stringify({ time: Date.now() }));
}
```

Decision guide:
- **Edge Runtime** — Auth checks, redirects, A/B tests, geolocation, lightweight API routes. ~0ms cold start.
- **Node.js Runtime** — Database connections, file system access, heavy computation, npm packages with native bindings. 250ms+ cold start.

---

## Common Patterns Quick Reference

### Server vs Client Component Decision

| Need | Component Type |
|------|---------------|
| Fetch data | Server Component |
| Access backend resources directly | Server Component |
| Keep sensitive info on server | Server Component |
| onClick, onChange, useState, useEffect | Client Component (`'use client'`) |
| Browser APIs (localStorage, geolocation) | Client Component |
| Custom hooks with state | Client Component |

### Route Segment Configuration

```tsx
// Force dynamic rendering
export const dynamic = 'force-dynamic';

// Force static rendering
export const dynamic = 'force-static';

// Revalidation interval
export const revalidate = 3600; // seconds

// Runtime selection
export const runtime = 'edge'; // or 'nodejs'

// Maximum request duration
export const maxDuration = 30; // seconds (Vercel-specific)
```

### Metadata API

```tsx
// Static metadata
export const metadata: Metadata = {
  title: 'My App',
  description: 'App description',
  openGraph: { title: 'My App', images: ['/og.png'] },
};

// Dynamic metadata
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  const post = await getPost(slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: { images: [post.coverImage] },
  };
}
```

### Common next.config.js

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  // Redirect legacy URLs
  async redirects() {
    return [
      { source: '/old-page', destination: '/new-page', permanent: true },
    ];
  },

  // Rewrite proxy
  async rewrites() {
    return [
      { source: '/api/proxy/:path*', destination: 'https://backend.example.com/:path*' },
    ];
  },

  // Custom headers
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        ],
      },
    ];
  },

  // Enable standalone for Docker
  output: 'standalone',

  // Strict mode
  reactStrictMode: true,

  // TypeScript errors won't block build
  typescript: { ignoreBuildErrors: false },

  // ESLint errors won't block build
  eslint: { ignoreDuringBuilds: false },
};

module.exports = nextConfig;
```
