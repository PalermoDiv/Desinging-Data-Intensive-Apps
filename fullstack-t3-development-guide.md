# The Complete Guide to Building Full-Stack Web Apps with the T3 Stack

A step-by-step, beginner-friendly yet technically deep guide to developing, deploying, and scaling modern full-stack web applications using the T3 stack and its ecosystem.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [What Is the T3 Stack?](#2-what-is-the-t3-stack)
3. [Prerequisites](#3-prerequisites)
4. [Project Initialization](#4-project-initialization)
5. [Project Structure Explained](#5-project-structure-explained)
6. [Environment Configuration](#6-environment-configuration)
7. [Database Setup with Prisma](#7-database-setup-with-prisma)
8. [Type-Safe APIs with tRPC](#8-type-safe-apis-with-trpc)
9. [Frontend with Next.js and Tailwind CSS](#9-frontend-with-nextjs-and-tailwind-css)
10. [Authentication with NextAuth.js](#10-authentication-with-nextauthjs)
11. [State Management and Data Fetching](#11-state-management-and-data-fetching)
12. [Forms and Validation](#12-forms-and-validation)
13. [Testing Strategy](#13-testing-strategy)
14. [Deployment Options](#14-deployment-options)
15. [Cloud Database Options](#15-cloud-database-options)
16. [CI/CD Pipeline](#16-cicd-pipeline)
17. [Monitoring and Observability](#17-monitoring-and-observability)
18. [Best Practices](#18-best-practices)
19. [Common Patterns and Example Project](#19-common-patterns-and-example-project)
20. [Troubleshooting](#20-troubleshooting)
21. [Suggested Learning Path](#21-suggested-learning-path)
22. [Conclusion](#22-conclusion)

---

## 1. Introduction

### What This Guide Covers

This guide walks you through the entire lifecycle of building a full-stack web application using a **T3-style stack**: a modern, type-safe, full-stack TypeScript architecture.

You will learn how to:

- Scaffold and configure a new project.
- Design a database schema with Prisma.
- Build type-safe APIs with tRPC.
- Create reactive UIs with Next.js and Tailwind CSS.
- Add authentication with NextAuth.js.
- Validate forms and manage server state.
- Test your application.
- Deploy to production with CI/CD.
- Monitor, scale, and secure your app.

### Who Is This For?

- Beginners who want a structured path to full-stack development.
- Intermediate developers moving from separate frontend/backend projects to unified TypeScript stacks.
- Teams evaluating the T3 stack for production use.

---

## 2. What Is the T3 Stack?

The **T3 Stack** is an opinionated full-stack framework built around TypeScript. It was created by Theo Browne and the T3 community. The acronym stands for:

| Letter | Technology | Role |
|--------|-----------|------|
| **T** | **TypeScript** | Type-safe JavaScript across the entire stack. |
| **3** | (Three core technologies) | Next.js, tRPC, Tailwind CSS |
| Plus | **Prisma** | Type-safe ORM for database access. |
| Plus | **NextAuth.js** | Authentication for Next.js applications. |

### T3 Stack Architecture

```mermaid
flowchart TB
    A[Browser] -->|HTTP / Server-Side Render| B[Next.js App Router]
    B --> C[Tailwind CSS UI]
    B --> D[tRPC Router]
    D --> E[Prisma ORM]
    E --> F[(PostgreSQL Database)]
    B --> G[NextAuth.js]
    G --> H[OAuth Providers]
```

### Why Use the T3 Stack?

| Benefit | Explanation |
|--------|-------------|
| **End-to-End Type Safety** | Share TypeScript types between frontend, backend, and database. |
| **Rapid Development** | Scaffold a production-ready app in minutes with `create-t3-app`. |
| **Modern Tooling** | Uses the latest React, Next.js, and TypeScript patterns. |
| **Flexibility** | Choose only the pieces you need: drop NextAuth, add Zod, switch databases, etc. |
| **Strong Community** | Active Discord, extensive templates, and best-practice guidance. |

### When to Use It

Use the T3 stack when:

- You want a single language across the entire application.
- You are building data-driven web applications.
- You value developer experience and type safety.
- You want SSR/SSG/ISR capabilities via Next.js.

> **Note:** The T3 stack is particularly strong for CRUD-heavy applications, dashboards, SaaS products, and internal tools.

---

## 3. Prerequisites

### Required Knowledge

Before starting, you should be comfortable with:

- JavaScript fundamentals
- Basic React (components, hooks, state)
- Command line basics
- Git and GitHub

### Required Tools

| Tool | Purpose | Installation |
|------|---------|--------------|
| **Node.js** | JavaScript runtime | `node -v` should show 18.x or higher |
| **npm / pnpm / yarn** | Package manager | pnpm is recommended |
| **Git** | Version control | `git --version` |
| **VS Code** | Code editor | Recommended with extensions below |
| **A database** | Data persistence | Local Postgres or cloud provider |

### Recommended VS Code Extensions

- ESLint
- Prettier
- Tailwind CSS IntelliSense
- Prisma
- Thunder Client or REST Client (optional)
- GitLens

### Verify Your Environment

```bash
node -v       # Should be v18 or higher
npm -v        # Or pnpm -v / yarn -v
git --version
```

> **Tip:** Use a Node version manager like `nvm` (macOS/Linux) or `fnm` (cross-platform) to switch Node versions easily.

---

## 4. Project Initialization

### Install create-t3-app

`create-t3-app` is the official CLI for scaffolding T3 projects.

```bash
npx create-t3-app@latest my-app
```

You can also use `pnpm`:

```bash
pnpm dlx create-t3-app@latest my-app
```

### CLI Options

During initialization, you will be asked to choose:

| Option | Recommendation |
|--------|---------------|
| **App Router or Pages Router?** | Choose **App Router** for new projects (Next.js 13+). |
| **Include NextAuth.js?** | Yes, if you need user accounts. |
| **Include Prisma?** | Yes, for database access. |
| **Include tRPC?** | Yes, this is the core of the stack. |
| **Include Tailwind CSS?** | Yes, for styling. |
| **Initialize Git?** | Yes. |
| **Install dependencies?** | Yes. |

### Start the Development Server

```bash
cd my-app
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000) to see your app.

---

## 5. Project Structure Explained

After initialization, your project will look like this:

```text
my-app/
├── prisma/
│   └── schema.prisma          # Database schema
├── public/                    # Static assets
├── src/
│   ├── app/                   # Next.js App Router
│   │   ├── api/
│   │   │   ├── auth/
│   │   │   │   └── [...nextauth]/route.ts
│   │   │   └── trpc/
│   │   │       └── [trpc]/route.ts
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── globals.css
│   ├── env.js                 # Validated environment variables
│   ├── server/
│   │   ├── api/
│   │   │   ├── routers/       # tRPC routers
│   │   │   │   └── post.ts
│   │   │   ├── trpc.ts        # tRPC context and helpers
│   │   │   └── root.ts        # Root tRPC router
│   │   ├── auth.ts            # NextAuth configuration
│   │   └── db.ts              # Prisma client singleton
│   └── styles/
│       └── globals.css
├── .env                       # Environment variables (not committed)
├── .env.example               # Example environment variables
├── next.config.js
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

### Key Files

| File | Purpose |
|------|---------|
| `prisma/schema.prisma` | Defines your database tables and relations. |
| `src/server/api/routers/*.ts` | tRPC route handlers grouped by domain. |
| `src/server/api/trpc.ts` | tRPC initialization, context, and middleware. |
| `src/server/auth.ts` | NextAuth providers and callbacks. |
| `src/env.js` | Runtime and build-time environment validation with T3 Env. |

---

## 6. Environment Configuration

### What Is T3 Env?

The T3 stack uses **T3 Env** to validate environment variables at build time and runtime. This catches missing or invalid configuration early.

### env.js Example

```javascript
import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

export const env = createEnv({
  server: {
    DATABASE_URL: z.string().url(),
    NEXTAUTH_SECRET: z.string().min(1),
    NEXTAUTH_URL: z.string().url(),
    DISCORD_CLIENT_ID: z.string().min(1),
    DISCORD_CLIENT_SECRET: z.string().min(1),
  },
  client: {
    NEXT_PUBLIC_APP_NAME: z.string().min(1),
  },
  runtimeEnv: {
    DATABASE_URL: process.env.DATABASE_URL,
    NEXTAUTH_SECRET: process.env.NEXTAUTH_SECRET,
    NEXTAUTH_URL: process.env.NEXTAUTH_URL,
    DISCORD_CLIENT_ID: process.env.DISCORD_CLIENT_ID,
    DISCORD_CLIENT_SECRET: process.env.DISCORD_CLIENT_SECRET,
    NEXT_PUBLIC_APP_NAME: process.env.NEXT_PUBLIC_APP_NAME,
  },
});
```

### .env.example

```env
DATABASE_URL="postgresql://user:password@localhost:5432/mydb"
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="your-secret-key"
DISCORD_CLIENT_ID=""
DISCORD_CLIENT_SECRET=""
NEXT_PUBLIC_APP_NAME="My T3 App"
```

> **Warning:** Never commit `.env` to Git. It is already in `.gitignore` by default.

### Generating NEXTAUTH_SECRET

```bash
openssl rand -base64 32
```

Copy the output into your `.env` file.

---

## 7. Database Setup with Prisma

### What Is Prisma?

Prisma is a modern ORM for Node.js and TypeScript. It provides:

- A declarative schema language.
- Type-safe database queries.
- Automatic migrations.
- A visual database management tool (Prisma Studio).

### Define Your Schema

Edit `prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Post {
  id        String   @id @default(cuid())
  title     String
  content   String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  authorId  String
  author    User     @relation(fields: [authorId], references: [id])
}

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  emailVerified DateTime?
  name          String?
  image         String?
  accounts      Account[]
  sessions      Session[]
  posts         Post[]
}

model Account {
  // NextAuth-required fields
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?
  user              User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

### Run Migrations

```bash
pnpm prisma migrate dev --name init
```

This creates the database tables and generates the Prisma Client.

### Generate the Client

After schema changes:

```bash
pnpm prisma generate
```

### Open Prisma Studio

```bash
pnpm prisma studio
```

Prisma Studio provides a GUI for browsing and editing your database.

### Querying with Prisma

Example tRPC router using Prisma:

```typescript
import { z } from "zod";
import { createTRPCRouter, protectedProcedure, publicProcedure } from "@/server/api/trpc";

export const postRouter = createTRPCRouter({
  getAll: publicProcedure.query(({ ctx }) => {
    return ctx.db.post.findMany({
      orderBy: { createdAt: "desc" },
      include: { author: true },
    });
  }),

  create: protectedProcedure
    .input(z.object({ title: z.string().min(1), content: z.string().optional() }))
    .mutation(({ ctx, input }) => {
      return ctx.db.post.create({
        data: {
          title: input.title,
          content: input.content,
          authorId: ctx.session.user.id,
        },
      });
    }),
});
```

> **Tip:** Use `protectedProcedure` for routes that require authentication. Use `publicProcedure` for open routes.

---

## 8. Type-Safe APIs with tRPC

### What Is tRPC?

tRPC lets you build fully type-safe APIs without writing schemas or manual fetch calls. You define procedures on the server, and call them directly from React components.

### tRPC Data Flow

```mermaid
flowchart LR
    A[React Component] -->|api.post.getAll.useQuery| B[tRPC Client]
    B --> C[tRPC Router]
    C --> D[Prisma ORM]
    D --> E[(Database)]
```

### Define a Router

```typescript
// src/server/api/routers/post.ts
import { z } from "zod";
import { createTRPCRouter, publicProcedure } from "@/server/api/trpc";

export const postRouter = createTRPCRouter({
  hello: publicProcedure
    .input(z.object({ text: z.string() }))
    .query(({ input }) => {
      return { greeting: `Hello ${input.text}` };
    }),
});
```

### Use tRPC in a Component

```tsx
// src/app/page.tsx
import { api } from "@/trpc/react";

export default function HomePage() {
  const { data, isLoading } = api.post.hello.useQuery({ text: "World" });

  if (isLoading) return <div>Loading...</div>;

  return <div>{data?.greeting}</div>;
}
```

### tRPC Procedures Explained

| Procedure | Use Case |
|-----------|----------|
| `query` | Read data (like GET). |
| `mutation` | Write data (like POST/PUT/DELETE). |
| `publicProcedure` | No authentication required. |
| `protectedProcedure` | Requires a valid session. |

### Input Validation with Zod

Every tRPC procedure can validate inputs with Zod:

```typescript
.create(
  protectedProcedure
    .input(z.object({ title: z.string().min(3).max(100) }))
    .mutation(async ({ ctx, input }) => { ... })
)
```

If the input is invalid, tRPC returns a typed error automatically.

> **Tip:** Keep routers organized by domain: `post.ts`, `user.ts`, `comment.ts`, etc.

---

## 9. Frontend with Next.js and Tailwind CSS

### Next.js App Router

The T3 stack uses the Next.js App Router. Each file in `src/app/` corresponds to a route.

| File | Route |
|------|-------|
| `src/app/page.tsx` | `/` |
| `src/app/about/page.tsx` | `/about` |
| `src/app/posts/[id]/page.tsx` | `/posts/:id` |
| `src/app/layout.tsx` | Root layout |

### Server Components vs. Client Components

| Type | Use For | Can Use tRPC? |
|------|---------|---------------|
| **Server Component** (default) | Fetching data on the server, SEO, static rendering | Yes, via `api.post.getAll()` directly |
| **Client Component** | Interactivity, browser APIs, hooks | Yes, via `api.post.getAll.useQuery()` |

Mark a client component with `"use client"` at the top.

```tsx
"use client";

import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

### Styling with Tailwind CSS

Tailwind provides utility classes for rapid UI development.

```tsx
export function Button({ children }: { children: React.ReactNode }) {
  return (
    <button className="rounded-lg bg-blue-600 px-4 py-2 text-white hover:bg-blue-700">
      {children}
    </button>
  );
}
```

### Full Page Example

```tsx
// src/app/page.tsx
import { api } from "@/trpc/server";
import { CreatePost } from "@/app/_components/create-post";

export default async function Home() {
  const posts = await api.post.getAll();

  return (
    <main className="container mx-auto p-4">
      <h1 className="text-3xl font-bold">My Posts</h1>
      <CreatePost />
      <ul className="mt-4 space-y-2">
        {posts.map((post) => (
          <li key={post.id} className="rounded border p-4">
            <h2 className="text-xl font-semibold">{post.title}</h2>
            <p className="text-gray-600">{post.content}</p>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

---

## 10. Authentication with NextAuth.js

### What Is NextAuth.js?

NextAuth.js (now Auth.js) is a complete authentication solution for Next.js. It supports OAuth (Google, GitHub, Discord), email magic links, and credentials-based login.

### Configure NextAuth

```typescript
// src/server/auth.ts
import { PrismaAdapter } from "@auth/prisma-adapter";
import { type GetServerSidePropsContext, type NextApiRequest, type NextApiResponse } from "next";
import { getServerSession, type NextAuthOptions } from "next-auth";
import DiscordProvider from "next-auth/providers/discord";
import { env } from "@/env";
import { db } from "@/server/db";

export const authOptions: NextAuthOptions = {
  callbacks: {
    session: ({ session, user }) => ({
      ...session,
      user: {
        ...session.user,
        id: user.id,
      },
    }),
  },
  adapter: PrismaAdapter(db),
  providers: [
    DiscordProvider({
      clientId: env.DISCORD_CLIENT_ID,
      clientSecret: env.DISCORD_CLIENT_SECRET,
    }),
  ],
};

export const getServerAuthSession = () => getServerSession(authOptions);
```

### Add Auth Route

The `create-t3-app` CLI creates this automatically:

```typescript
// src/app/api/auth/[...nextauth]/route.ts
import NextAuth from "next-auth";
import { authOptions } from "@/server/auth";

const handler = NextAuth(authOptions);
export { handler as GET, handler as POST };
```

### Protect Routes in tRPC

```typescript
// src/server/api/trpc.ts
import { initTRPC, TRPCError } from "@trpc/server";
import { getServerAuthSession } from "@/server/auth";

const t = initTRPC.create();

const createContext = async () => {
  const session = await getServerAuthSession();
  return { session, db };
};

export const router = t.router;
export const publicProcedure = t.procedure;

const enforceUserIsAuthed = t.middleware(({ ctx, next }) => {
  if (!ctx.session || !ctx.session.user) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return next({
    ctx: {
      session: { ...ctx.session, user: ctx.session.user },
    },
  });
});

export const protectedProcedure = t.procedure.use(enforceUserIsAuthed);
```

### Use Authentication in Components

```tsx
"use client";

import { signIn, signOut, useSession } from "next-auth/react";

export function AuthButton() {
  const { data: session } = useSession();

  if (session) {
    return <button onClick={() => signOut()}>Sign out</button>;
  }

  return <button onClick={() => signIn("discord")}>Sign in with Discord</button>;
}
```

> **Tip:** Wrap your app in the `SessionProvider` in `src/app/layout.tsx` for client-side session access.

---

## 11. State Management and Data Fetching

### Server State with tRPC

For data that lives on the server, use tRPC hooks:

```tsx
const { data, isLoading, error } = api.post.getAll.useQuery();
const createPost = api.post.create.useMutation();
```

### Client State with React Hooks

For local UI state, use `useState` and `useReducer`:

```tsx
const [isOpen, setIsOpen] = useState(false);
```

### When to Use a Global State Library

Consider Zustand, Jotai, or Redux only when:

- Multiple unrelated components share the same state.
- Prop drilling becomes unmanageable.
- You need complex state logic.

For most T3 apps, server state via tRPC and local state via hooks are enough.

### Data Fetching Patterns

| Pattern | Use Case |
|---------|----------|
| Server Component + direct tRPC call | Initial page load, SEO-critical data. |
| Client Component + `useQuery` | Interactive dashboards, polling, mutations. |
| `useSuspenseQuery` | Loading boundaries and streaming. |
| Prefetching | Faster navigation between pages. |

---

## 12. Forms and Validation

### Recommended Libraries

| Library | Purpose |
|---------|---------|
| **React Hook Form** | Performant form handling. |
| **Zod** | Schema validation shared with tRPC. |
| **@hookform/resolvers** | Connect Zod to React Hook Form. |

### Install

```bash
pnpm add react-hook-form zod @hookform/resolvers
```

### Example Form

```tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { api } from "@/trpc/react";

const schema = z.object({
  title: z.string().min(3, "Title must be at least 3 characters"),
  content: z.string().optional(),
});

type FormData = z.infer<typeof schema>;

export function CreatePostForm() {
  const { register, handleSubmit, reset, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  const createPost = api.post.create.useMutation({
    onSuccess: () => {
      reset();
    },
  });

  const onSubmit = (data: FormData) => {
    createPost.mutate(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <input {...register("title")} placeholder="Title" className="border p-2" />
        {errors.title && <p className="text-red-500">{errors.title.message}</p>}
      </div>
      <textarea {...register("content")} placeholder="Content" className="border p-2" />
      <button type="submit" className="rounded bg-blue-600 px-4 py-2 text-white">
        Create Post
      </button>
    </form>
  );
}
```

> **Tip:** Share Zod schemas between forms and tRPC to guarantee frontend and backend validation stay in sync.

---

## 13. Testing Strategy

### Types of Tests

| Type | Tool | Purpose |
|------|------|---------|
| **Unit Tests** | Vitest or Jest | Test individual functions and utilities. |
| **Component Tests** | React Testing Library | Test React components in isolation. |
| **E2E Tests** | Playwright | Test complete user flows in a real browser. |

### Install Vitest

```bash
pnpm add -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom
```

### Example Unit Test

```typescript
// src/utils/sum.test.ts
import { expect, test } from "vitest";
import { sum } from "./sum";

test("adds two numbers", () => {
  expect(sum(1, 2)).toBe(3);
});
```

### Example E2E Test with Playwright

```typescript
// e2e/home.spec.ts
import { test, expect } from "@playwright/test";

test("homepage loads", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveTitle(/My T3 App/);
});
```

### Add Test Scripts

```json
{
  "scripts": {
    "test": "vitest",
    "test:e2e": "playwright test"
  }
}
```

> **Tip:** Run E2E tests against a real database or use isolated test databases to avoid polluting development data.

---

## 14. Deployment Options

### Recommended Deployment Architecture

```mermaid
flowchart LR
    A[GitHub] -->|Push to main| B[CI/CD Pipeline]
    B --> C[Build Next.js App]
    C --> D[Vercel]
    D --> E[Edge Network / CDN]
    D --> F[tRPC API Routes]
    F --> G[(PostgreSQL Database)]
```

### Vercel (Recommended for T3)

Vercel is the natural home for Next.js applications and the T3 stack.

**Steps:**

1. Push your project to GitHub.
2. Import the repository in Vercel.
3. Add environment variables in the Vercel dashboard.
4. Deploy.

**Pros:**

- Zero-config Next.js support.
- Automatic preview deployments.
- Edge network for static assets.
- Serverless functions for API routes.

### Railway / Render

Use these for more backend-heavy apps or when you want a persistent server.

**Best for:**

- Long-running background jobs.
- Websockets.
- Larger databases.

### Fly.io

Deploy Dockerized T3 apps globally close to users.

**Best for:**

- Apps needing low latency in multiple regions.
- Container-based deployments.

### Self-Hosted / VPS

For full control, deploy to a VPS with Docker.

```bash
# Build and run with Docker
docker build -t my-t3-app .
docker run -p 3000:3000 --env-file .env my-t3-app
```

### Deployment Comparison

| Platform | Difficulty | Cost | Best For |
|----------|-----------|------|----------|
| **Vercel** | Very Easy | Free tier + usage | Most T3 apps |
| **Railway** | Easy | Medium | Full-stack apps with persistent backend |
| **Render** | Easy | Medium | Predictable pricing, production apps |
| **Fly.io** | Medium | Medium | Global low-latency apps |
| **VPS + Docker** | Hard | Low | Full control, learning, cost optimization |

---

## 15. Cloud Database Options

### Recommended Databases for T3

| Provider | Type | Best For |
|----------|------|----------|
| **Vercel Postgres** | Managed PostgreSQL | Tight Vercel integration, serverless scale. |
| **Neon** | Serverless PostgreSQL | Branching, scale-to-zero, generous free tier. |
| **Supabase** | Managed PostgreSQL + BaaS | Auth, realtime, storage, and database together. |
| **Railway** | Managed PostgreSQL | Simple setup alongside your Railway app. |
| **PlanetScale** | MySQL-compatible | Branching schema, deploy requests, great DX. |
| **AWS RDS** | Managed PostgreSQL/MySQL | Enterprise, compliance, complex integrations. |

### Database Connection String

Your `DATABASE_URL` typically looks like:

```env
DATABASE_URL="postgresql://user:password@host:5432/dbname?sslmode=require"
```

### Migration Strategy for Production

1. Run migrations in CI/CD before deploying the app, or
2. Use a separate migration job.
3. Never run `prisma migrate dev` in production; use `prisma migrate deploy`.

```bash
pnpm prisma migrate deploy
```

> **Warning:** Always back up your production database before running migrations.

---

## 16. CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 8

      - name: Install dependencies
        run: pnpm install

      - name: Run linter
        run: pnpm lint

      - name: Run type check
        run: pnpm typecheck

      - name: Run tests
        run: pnpm test

      - name: Build
        run: pnpm build
```

### Vercel Auto-Deploy

Vercel automatically deploys on every push to `main`. Preview deployments are created for pull requests.

### Manual Production Checklist

Before deploying:

- [ ] Environment variables are set in production.
- [ ] Database migrations are applied.
- [ ] Tests pass.
- [ ] Linting and type checking pass.
- [ ] Auth provider redirect URLs are configured.

---

## 17. Monitoring and Observability

### What to Monitor

| Metric | Tool Examples |
|--------|--------------|
| **Errors** | Sentry, LogRocket |
| **Performance** | Vercel Analytics, Datadog |
| **Uptime** | UptimeRobot, Pingdom |
| **Logs** | Vercel Logs, Datadog, Logtail |
| **Database** | Prisma Optimize, database provider dashboards |

### Add Sentry for Error Tracking

```bash
pnpm dlx @sentry/wizard@latest -i nextjs
```

Sentry captures frontend and backend errors automatically.

### Performance Monitoring

Enable Vercel Analytics:

```bash
pnpm add @vercel/analytics
```

Add to `src/app/layout.tsx`:

```tsx
import { Analytics } from "@vercel/analytics/react";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
      <Analytics />
    </html>
  );
}
```

---

## 18. Best Practices

### Code Organization

- Group by feature when the app grows: `src/features/posts/`.
- Keep server-only code inside `src/server/`.
- Keep shared UI components in `src/components/`.

### Type Safety

- Share Zod schemas between client forms and tRPC routers.
- Avoid `any`. Use `unknown` when type is uncertain.
- Generate Prisma types after every schema change.

### Security

- Validate all user input with Zod.
- Use `protectedProcedure` for sensitive operations.
- Store secrets in environment variables, never in the frontend.
- Enable Row Level Security (RLS) if using Supabase directly.
- Keep dependencies updated.

### Performance

- Use Next.js Server Components by default.
- Optimize images with `next/image`.
- Use database indexes for frequently queried columns.
- Cache tRPC queries when appropriate.

### Database

- Use migrations in production (`prisma migrate deploy`).
- Add indexes on foreign keys and search fields.
- Use transactions for multi-step operations.
- Regular backups.

> **Tip:** Review the [T3 collection](https://create.t3.gg/) and community examples for real-world patterns.

---

## 19. Common Patterns and Example Project

### Building a Simple Blog

**Features:**

- Users sign in with Discord.
- Authenticated users create posts.
- Public feed shows all posts.

**Schema:**

```prisma
model Post {
  id        String   @id @default(cuid())
  title     String
  content   String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  authorId  String
  author    User     @relation(fields: [authorId], references: [id])
}
```

**tRPC Router:**

```typescript
export const postRouter = createTRPCRouter({
  getAll: publicProcedure.query(({ ctx }) => {
    return ctx.db.post.findMany({ orderBy: { createdAt: "desc" }, include: { author: true } });
  }),

  create: protectedProcedure
    .input(z.object({ title: z.string().min(3), content: z.string().optional() }))
    .mutation(({ ctx, input }) => {
      return ctx.db.post.create({
        data: { ...input, authorId: ctx.session.user.id },
      });
    }),
});
```

**Homepage:**

```tsx
import { api } from "@/trpc/server";
import { CreatePostForm } from "@/app/_components/create-post-form";

export default async function Home() {
  const posts = await api.post.getAll();

  return (
    <main className="container mx-auto p-4">
      <h1 className="mb-4 text-3xl font-bold">Blog</h1>
      <CreatePostForm />
      <div className="mt-6 space-y-4">
        {posts.map((post) => (
          <article key={post.id} className="rounded border p-4">
            <h2 className="text-xl font-semibold">{post.title}</h2>
            <p className="text-gray-600">{post.content}</p>
            <p className="text-sm text-gray-400">By {post.author.name}</p>
          </article>
        ))}
      </div>
    </main>
  );
}
```

---

## 20. Troubleshooting

### Common Issues

| Problem | Solution |
|---------|----------|
| `DATABASE_URL` is not defined | Check `.env` and restart the dev server. |
| Prisma Client not found | Run `pnpm prisma generate`. |
| tRPC query returns `undefined` | Check that the router is registered in `root.ts`. |
| NextAuth session is `null` | Verify `NEXTAUTH_SECRET` and provider credentials. |
| Build fails on Vercel | Ensure all env vars are set in Vercel dashboard. |
| Type errors after schema change | Regenerate Prisma Client and restart TypeScript server. |
| CORS errors | Ensure `NEXTAUTH_URL` matches your deployed domain. |

### Reset Local Database

```bash
pnpm prisma migrate reset
```

> **Warning:** This deletes all local data. Use only in development.

### Debug tRPC Procedures

Add logging middleware:

```typescript
const logger = t.middleware(async ({ path, type, next }) => {
  const start = Date.now();
  const result = await next();
  const duration = Date.now() - start;
  console.log(`[tRPC] ${type} ${path} - ${duration}ms`);
  return result;
});

export const publicProcedure = t.procedure.use(logger);
```

---

## 21. Suggested Learning Path

1. **Learn TypeScript basics** — types, interfaces, generics.
2. **Master React and Next.js** — components, hooks, App Router.
3. **Understand relational databases** — tables, relations, SQL.
4. **Learn Prisma** — schema design, migrations, queries.
5. **Learn tRPC** — routers, procedures, middleware.
6. **Add authentication** — OAuth flow, sessions, protected routes.
7. **Style with Tailwind CSS** — utility-first design.
8. **Build a complete project** — blog, dashboard, or SaaS MVP.
9. **Deploy to Vercel** — add database, env vars, CI/CD.
10. **Scale and monitor** — error tracking, analytics, optimizations.

---

## 22. Conclusion

The T3 stack offers one of the best developer experiences for building modern full-stack web applications. By combining TypeScript, Next.js, tRPC, Prisma, Tailwind CSS, and NextAuth.js, you get:

- End-to-end type safety.
- Fast development cycles.
- A clear project structure.
- Excellent deployment options.
- A strong, growing community.

Start small, deploy early, and iterate. Your first T3 app does not need to use every feature — add complexity only when your project demands it.

---

*Happy building!*
