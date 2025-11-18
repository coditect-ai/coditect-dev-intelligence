# ADR-006: React + Next.js Frontend - Project Intelligence Platform

```yaml
Document: ADR-006-project-intelligence-react-nextjs-frontend
Version: 1.0.0
Purpose: Select Next.js (React) for server-side rendering and API routes
Audience: Engineering teams, frontend developers, UX designers
Date Created: 2025-11-17
Status: ACCEPTED
Related ADRs: ADR-005 (FastAPI Backend)
```

---

## Executive Summary

**Decision**: Use **Next.js 14 (React 18)** for the frontend.

**Why Next.js**:
- ✅ **Server-Side Rendering (SSR)**: SEO-friendly, fast initial load
- ✅ **API Routes**: Built-in backend for proxying FastAPI (BFF pattern)
- ✅ **React 18**: Modern component model, hooks, suspense
- ✅ **TypeScript**: Type-safe frontend development
- ✅ **Vercel Deployment**: Zero-config hosting (or self-host on GCP)

**Use Case**: Interactive project timeline, semantic search UI, organization management

**Alternatives Rejected**: Vue (smaller ecosystem), Angular (too heavy), SvelteKit (immature)

---

## Context and Problem Statement

### Requirements

The Project Intelligence Platform frontend must:

1. **Interactive Timeline**: Visualize 49 checkpoints across time
2. **Semantic Search**: Real-time search with ChromaDB integration
3. **Multi-Tenant UI**: Role-based access control (RBAC)
4. **SEO-Friendly**: Server-side rendering for public pages
5. **Type-Safe**: TypeScript for frontend-backend contracts

---

## Decision Drivers

1. **Server-Side Rendering** - SEO, fast initial load
2. **API Routes** - BFF pattern (Backend-for-Frontend)
3. **React Ecosystem** - Large community, component libraries
4. **TypeScript** - Type safety across frontend/backend
5. **Deployment** - Easy hosting (Vercel, GCP Cloud Run)

---

## Considered Options

### Option 1: **Next.js 14 (React 18)** (SELECTED ✅)

**Key Features**:
```typescript
// app/timeline/page.tsx (Next.js 14 App Router)
import { Suspense } from 'react';

// Server Component (SSR)
async function Timeline() {
  const checkpoints = await fetch('https://api.coditect.ai/checkpoints', {
    cache: 'no-store' // Always fresh data
  }).then(res => res.json());

  return (
    <div>
      {checkpoints.map(checkpoint => (
        <CheckpointCard key={checkpoint.id} data={checkpoint} />
      ))}
    </div>
  );
}

// API Route (BFF pattern)
// app/api/search/route.ts
export async function POST(request: Request) {
  const { query } = await request.json();

  // Proxy to FastAPI backend
  const results = await fetch('https://api.coditect.ai/search/semantic', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query })
  });

  return Response.json(results);
}
```

**Pros**:
- ✅ **SSR Built-In**: SEO-friendly, fast initial load
- ✅ **API Routes**: Backend-for-Frontend (BFF) pattern
- ✅ **React 18**: Modern features (Suspense, Server Components)
- ✅ **TypeScript**: Type-safe development
- ✅ **Zero-Config**: Deploy to Vercel in 30 seconds
- ✅ **Image Optimization**: Automatic image resizing, lazy loading
- ✅ **Code Splitting**: Automatic route-based code splitting

**Cons**:
- ⚠️ **Learning Curve**: App Router (Next.js 14) is new
- ⚠️ **Vendor Lock-In**: Optimized for Vercel (but can self-host)

**Performance**:
- **Lighthouse Score**: 95+ (SEO, Performance, Accessibility)
- **Time to Interactive (TTI)**: <2 seconds

---

### Option 2: Vue 3 + Nuxt 3

**Pros**:
- ✅ **Simple**: Easier learning curve than React
- ✅ **SSR**: Nuxt provides SSR like Next.js
- ✅ **Composition API**: Modern Vue 3 features

**Cons**:
- ❌ **Smaller Ecosystem**: Fewer component libraries than React
- ❌ **Team Familiarity**: Team has React experience
- ❌ **TypeScript**: Less mature than React ecosystem

**Why Rejected**: Smaller ecosystem, team prefers React

---

### Option 3: Angular

**Pros**:
- ✅ **TypeScript Native**: Built with TypeScript
- ✅ **Batteries Included**: Router, HTTP, Forms built-in
- ✅ **Enterprise**: Used by large enterprises

**Cons**:
- ❌ **Heavy**: Large framework, steep learning curve
- ❌ **SSR Complexity**: Angular Universal is complex
- ❌ **Declining Popularity**: React/Vue gaining market share

**Why Rejected**: Too heavy, complex SSR setup

---

### Option 4: SvelteKit

**Pros**:
- ✅ **Performance**: Compiled away (no virtual DOM)
- ✅ **Simple**: Less boilerplate than React
- ✅ **SSR**: Built-in like Next.js

**Cons**:
- ❌ **Immature Ecosystem**: Fewer libraries, smaller community
- ❌ **Team Unfamiliarity**: Learning curve
- ❌ **Unproven at Scale**: Less production usage

**Why Rejected**: Immature ecosystem, team prefers React

---

## Decision Outcome

**Chosen Option**: **Next.js 14 (React 18)** (Option 1)

### Rationale

1. **SSR Built-In**: SEO-friendly, fast initial load
2. **API Routes**: BFF pattern simplifies frontend-backend integration
3. **React Ecosystem**: Largest component library ecosystem (Chakra UI, Material-UI, shadcn/ui)
4. **Team Familiarity**: Team has React experience
5. **TypeScript**: Best-in-class type safety

### Implementation

```typescript
// app/layout.tsx (Root Layout)
import { Inter } from 'next/font/google';
import { ChakraProvider } from '@chakra-ui/react';

const inter = Inter({ subsets: ['latin'] });

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <ChakraProvider>
          {children}
        </ChakraProvider>
      </body>
    </html>
  );
}

// app/projects/[id]/page.tsx (Dynamic Route)
interface PageProps {
  params: { id: string };
}

async function ProjectPage({ params }: PageProps) {
  const project = await fetch(`https://api.coditect.ai/projects/${params.id}`)
    .then(res => res.json());

  return (
    <div>
      <h1>{project.name}</h1>
      <Timeline projectId={project.id} />
    </div>
  );
}
```

---

## Consequences

### Positive ✅

1. **SEO-Friendly**: SSR ensures search engines index content
2. **Fast Initial Load**: Server-rendered HTML sent to client
3. **Type-Safe**: TypeScript prevents frontend-backend contract bugs
4. **Developer Experience**: Hot module reload, auto-complete

### Negative ⚠️

1. **Learning Curve**: Next.js 14 App Router is new
   - **Mitigation**: Team training, documentation

2. **Vercel Optimization**: Optimized for Vercel (but can self-host)
   - **Mitigation**: Deploy to GCP Cloud Run

---

## Related Decisions

- **ADR-005**: [FastAPI Backend](ADR-005-fastapi-over-flask-django.md)
- **ADR-007**: [GCP Cloud Run Deployment](ADR-007-gcp-cloud-run-deployment.md)

---

## Approval

**Status**: ACCEPTED
**Decision Date**: 2025-11-17
**Approved By**: Engineering Leadership

---

**Made with ❤️ by CODITECT Engineering**
