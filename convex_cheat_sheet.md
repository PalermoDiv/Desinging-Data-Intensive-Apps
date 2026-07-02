# Convex Cheat Sheet

A compact reference for building a TypeScript application with Convex.

> **Mental model:** clients subscribe to backend query results; mutations change the database; Convex automatically reruns affected queries and pushes consistent updates.

---

## Quick Start

```bash
npm install convex
npx convex dev       # Develop and sync backend functions
npx convex deploy    # Deploy backend changes to production
```

Typical structure:

```text
convex/
├── _generated/      # Generated; do not edit
├── schema.ts        # Tables, validators, and indexes
├── tasks.ts         # Backend functions
├── http.ts          # Optional HTTP routes
└── crons.ts         # Optional recurring jobs
```

---

## Function Decision Table

| Need | Use |
|---|---|
| Read database data reactively | `query` |
| Write database data transactionally | `mutation` |
| Call an external API | `action` |
| Receive a webhook or raw HTTP request | `httpAction` |
| Hide a backend function from clients | `internalQuery`, `internalMutation`, or `internalAction` |
| Run work later | `ctx.scheduler.runAfter` or `runAt` |
| Run recurring work | Cron job |

| Capability | Query | Mutation | Action |
|---|---:|---:|---:|
| Direct database read | Yes | Yes | No |
| Direct database write | No | Yes | No |
| External `fetch` | No | No | Yes |
| Transactional | Yes | Yes | No |
| Reactive/cached | Yes | No | No |

---

## Schema

```ts
// convex/schema.ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  tasks: defineTable({
    ownerId: v.id("users"),
    title: v.string(),
    completed: v.boolean(),
    priority: v.optional(v.number()),
  })
    .index("by_owner", ["ownerId"])
    .index("by_owner_and_completed", ["ownerId", "completed"]),

  users: defineTable({
    name: v.string(),
    tokenIdentifier: v.string(),
  }).index("by_token", ["tokenIdentifier"]),
});
```

Common validators:

```ts
v.string()
v.number()
v.boolean()
v.null()
v.id("tasks")
v.array(v.string())
v.object({ name: v.string() })
v.optional(v.number())
v.union(v.literal("open"), v.literal("closed"))
```

---

## Query

```ts
import { query } from "./_generated/server";
import { v } from "convex/values";

export const listOpen = query({
  args: { ownerId: v.id("users") },
  handler: async (ctx, { ownerId }) => {
    return await ctx.db
      .query("tasks")
      .withIndex("by_owner_and_completed", (q) =>
        q.eq("ownerId", ownerId).eq("completed", false),
      )
      .take(50);
  },
});
```

Query helpers:

```ts
await ctx.db.get(id);                       // One document by ID
await ctx.db.query("tasks").first();       // First result or null
await ctx.db.query("tasks").unique();      // Exactly zero or one
await ctx.db.query("tasks").take(20);      // Bounded result
await ctx.db.query("tasks").collect();     // All results: use carefully
```

---

## Mutation

```ts
import { mutation } from "./_generated/server";
import { v } from "convex/values";

export const create = mutation({
  args: { ownerId: v.id("users"), title: v.string() },
  handler: async (ctx, { ownerId, title }) => {
    const normalized = title.trim();
    if (!normalized) throw new Error("Title is required");

    return await ctx.db.insert("tasks", {
      ownerId,
      title: normalized,
      completed: false,
    });
  },
});
```

Writes:

```ts
const id = await ctx.db.insert("tasks", document);
await ctx.db.patch(id, { completed: true });
await ctx.db.replace(id, replacementDocument);
await ctx.db.delete(id);
```

The entire mutation is one transaction. Throwing rolls it back.

---

## Action

```ts
import { action } from "./_generated/server";
import { api, internal } from "./_generated/api";
import { v } from "convex/values";

export const summarize = action({
  args: { taskId: v.id("tasks") },
  handler: async (ctx, { taskId }) => {
    const task = await ctx.runQuery(api.tasks.get, { taskId });
    if (task === null) throw new Error("Task not found");

    const response = await fetch("https://example.com/summarize", {
      method: "POST",
      body: JSON.stringify({ text: task.title }),
    });
    if (!response.ok) throw new Error("Summary request failed");

    const { summary } = await response.json();
    await ctx.runMutation(internal.tasks.saveSummary, { taskId, summary });
  },
});
```

Need Node.js APIs or a Node-only package? Put this at the top of the action file:

```ts
"use node";
```

Do not define queries or mutations in a `"use node"` file.

---

## Public vs. Internal Functions

```ts
import {
  internalAction,
  internalMutation,
  internalQuery,
} from "./_generated/server";
```

```ts
api.tasks.create                 // Public: callable by clients
internal.tasks.rebuildIndex      // Internal: backend only
```

Default to internal functions for scheduled jobs, privileged helpers, and workflow steps.

---

## React Client

Provider:

```tsx
import { ConvexProvider, ConvexReactClient } from "convex/react";

const client = new ConvexReactClient(import.meta.env.VITE_CONVEX_URL);

root.render(
  <ConvexProvider client={client}>
    <App />
  </ConvexProvider>,
);
```

Hooks:

```tsx
import { useAction, useMutation, useQuery } from "convex/react";
import { api } from "../convex/_generated/api";

const tasks = useQuery(api.tasks.listOpen, { ownerId });
const createTask = useMutation(api.tasks.create);
const summarize = useAction(api.tasks.summarize);

if (tasks === undefined) return <p>Loading...</p>;
```

`useQuery` returns `undefined` while its first result is loading.

---

## Indexes

Define:

```ts
defineTable({
  ownerId: v.id("users"),
  completed: v.boolean(),
}).index("by_owner_and_completed", ["ownerId", "completed"]);
```

Use:

```ts
ctx.db
  .query("tasks")
  .withIndex("by_owner_and_completed", (q) =>
    q.eq("ownerId", ownerId).eq("completed", false),
  );
```

Rules of thumb:

- Field order matters.
- Index the actual query prefix.
- Prefer `withIndex` over scanning and `.filter()`.
- Avoid unbounded `.collect()` on growing tables.
- Use pagination for feeds and histories.

---

## Authentication and Authorization

```ts
const identity = await ctx.auth.getUserIdentity();
if (identity === null) throw new Error("Unauthenticated");
```

Authentication is not authorization. For every protected function:

1. Get the identity.
2. Resolve the application user.
3. Load the requested document.
4. Check ownership, membership, or role.
5. Only then read or write protected data.

Never trust a client-provided `userId` without checking it against the authenticated identity.

---

## Scheduling

```ts
await ctx.scheduler.runAfter(
  0,
  internal.emails.sendWelcome,
  { userId },
);

await ctx.scheduler.runAt(
  Date.now() + 60_000,
  internal.reminders.send,
  { reminderId },
);
```

Cron:

```ts
// convex/crons.ts
import { cronJobs } from "convex/server";
import { internal } from "./_generated/api";

const crons = cronJobs();
crons.daily(
  "cleanup",
  { hourUTC: 4, minuteUTC: 0 },
  internal.cleanup.run,
);

export default crons;
```

Preferred external-work pattern:

```text
Client → mutation records intent → mutation schedules internal action
       → action calls provider → internal mutation stores result
```

---

## HTTP Action

```ts
// convex/http.ts
import { httpRouter } from "convex/server";
import { httpAction } from "./_generated/server";

const http = httpRouter();

http.route({
  path: "/webhook",
  method: "POST",
  handler: httpAction(async (_ctx, request) => {
    // Verify the provider signature before trusting the body.
    const payload = await request.json();
    return Response.json({ received: Boolean(payload) });
  }),
});

export default http;
```

---

## File Upload Flow

```text
1. Mutation: ctx.storage.generateUploadUrl()
2. Client: POST file bytes to upload URL
3. Client receives storage ID
4. Mutation stores storage ID plus owner/metadata
5. Query: ctx.storage.getUrl(storageId)
```

Keep authorization metadata in database documents; a storage ID is not an access policy.

---

## Error and Consistency Rules

- Queries are deterministic and read-only.
- Mutations are deterministic transactions.
- Never call external services from a mutation.
- Actions can have side effects and are not automatically safe to retry.
- Use idempotency keys for payments, emails, and external jobs.
- Record workflow status (`pending`, `processing`, `completed`, `failed`).
- Validate all public arguments with `v` validators.
- Use `internal*` functions for privileged backend operations.

---

## Common Mistakes

| Mistake | Better approach |
|---|---|
| Filtering an entire table | Define and use an index |
| Calling `.collect()` on an unlimited feed | Paginate or use `.take(n)` |
| Trusting a client-provided owner ID | Compare with authenticated identity |
| Calling an API from transactional logic | Move it to an action |
| Calling a public function from a scheduled job | Use an internal function |
| Editing `_generated` files | Change source files and regenerate |
| Exposing secrets to the frontend | Use backend deployment environment variables |
| Retrying a side effect blindly | Make it idempotent and track status |
| Using many separate `runQuery` calls in an action | Batch related reads in one internal query when consistency matters |

---

## Production Checklist

- [ ] Schema and runtime validators are defined.
- [ ] Growing queries use indexes and bounded results.
- [ ] Protected functions enforce server-side authorization.
- [ ] Backend-only functions use `internal*`.
- [ ] External effects are idempotent and observable.
- [ ] Secrets are stored in deployment environment variables.
- [ ] Development, preview, and production deployments are separate.
- [ ] Authentication provider production URLs are configured.
- [ ] Backups and recovery needs have been reviewed.
- [ ] Current Convex limits and pricing have been checked.

---

## Official Documentation

- [Overview](https://docs.convex.dev/understanding/overview)
- [Functions](https://docs.convex.dev/functions/overview)
- [Database](https://docs.convex.dev/database/overview)
- [Indexes](https://docs.convex.dev/database/reading-data/indexes/)
- [Real-time updates](https://docs.convex.dev/realtime)
- [Actions](https://docs.convex.dev/functions/actions)
- [HTTP actions](https://docs.convex.dev/functions/http-actions)
- [Production hosting](https://docs.convex.dev/production/hosting/)

*Checked against the official Convex documentation in July 2026. Confirm production-sensitive commands, limits, and hosted features against the current docs.*
