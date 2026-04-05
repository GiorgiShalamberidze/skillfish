---
name: supabase-architect
description: Supabase development: Row-Level Security policies, Edge Functions, Realtime subscriptions, Storage, Auth configuration, and migration from Firebase.
---

# Supabase Architect

Design and implement production-grade Supabase backends covering database schema design, Row-Level Security, authentication, Edge Functions, Realtime subscriptions, Storage, client SDK patterns, and Firebase migration strategies.

---

## Table of Contents

- [Database and Schema](#1-database-and-schema)
- [Row-Level Security](#2-row-level-security)
- [Authentication](#3-authentication)
- [Edge Functions](#4-edge-functions)
- [Realtime](#5-realtime)
- [Storage](#6-storage)
- [Client SDK Patterns](#7-client-sdk-patterns)
- [Migration from Firebase](#8-migration-from-firebase)

---

## When to Use

- Designing or reviewing a Supabase database schema
- Writing or auditing Row-Level Security policies
- Configuring Supabase Auth with email, OAuth, or magic links
- Building Deno Edge Functions for server-side logic
- Setting up Realtime subscriptions (Postgres Changes, Broadcast, Presence)
- Managing file uploads with Supabase Storage and signed URLs
- Integrating supabase-js into React, Next.js, or other frontend frameworks
- Migrating an existing Firebase project to Supabase

---

## 1. Database and Schema

### Table Design Principles

- Use `uuid` primary keys via `gen_random_uuid()` for all user-facing tables.
- Always include `created_at` and `updated_at` timestamptz columns.
- Prefer `text` over `varchar(n)` unless a hard length constraint is required.
- Normalize to third normal form by default; denormalize deliberately for query performance.

### Standard Table Template

```sql
create table public.projects (
  id          uuid primary key default gen_random_uuid(),
  owner_id    uuid not null references auth.users(id) on delete cascade,
  name        text not null,
  description text,
  status      text not null default 'draft'
    check (status in ('draft', 'active', 'archived')),
  metadata    jsonb not null default '{}',
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);

create index idx_projects_owner on public.projects(owner_id);
create index idx_projects_status on public.projects(status);
```

### Foreign Keys and Cascades

```sql
create table public.tasks (
  id          uuid primary key default gen_random_uuid(),
  project_id  uuid not null references public.projects(id) on delete cascade,
  assignee_id uuid references auth.users(id) on delete set null,
  title       text not null,
  completed   boolean not null default false,
  position    integer not null default 0,
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);

-- Many-to-many junction table
create table public.project_members (
  project_id uuid not null references public.projects(id) on delete cascade,
  user_id    uuid not null references auth.users(id) on delete cascade,
  role       text not null default 'member'
    check (role in ('owner', 'admin', 'member', 'viewer')),
  joined_at  timestamptz not null default now(),
  primary key (project_id, user_id)
);
```

### Enums vs Check Constraints

Prefer check constraints for values that may change. Use Postgres enums only when the set is truly fixed and shared across multiple tables.

```sql
-- Check constraint (preferred)
status text not null check (status in ('draft', 'active', 'archived'))

-- Postgres enum (use sparingly)
create type subscription_tier as enum ('free', 'pro', 'enterprise');
```

### Generated Columns and Full-Text Search

```sql
alter table public.projects
  add column fts tsvector
    generated always as (
      setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
      setweight(to_tsvector('english', coalesce(description, '')), 'B')
    ) stored;

create index idx_projects_fts on public.projects using gin(fts);
```

### Database Functions and Triggers

```sql
create or replace function public.handle_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger set_updated_at
  before update on public.projects
  for each row execute function public.handle_updated_at();

-- Atomic business logic
create or replace function public.archive_project(p_project_id uuid)
returns void as $$
begin
  update public.projects set status = 'archived' where id = p_project_id;
  update public.tasks set completed = true
    where project_id = p_project_id and completed = false;
end;
$$ language plpgsql security definer;
```

### Anti-Patterns: Database

- **Serial integers for public IDs.** They leak row counts and are guessable. Use UUIDs.
- **Arrays of IDs in a `jsonb` column instead of a junction table.** You lose referential integrity.
- **Skipping indexes on foreign key columns.** Every FK used in WHERE/JOIN needs an index.
- **`security definer` functions without access control.** Always pair with RLS or permission checks.

---

## 2. Row-Level Security

### Enabling RLS

RLS must be enabled on every table that clients can access. Without it, `anon` and `authenticated` roles have full access.

```sql
alter table public.projects enable row level security;
alter table public.tasks enable row level security;
alter table public.project_members enable row level security;
```

### SELECT Policies

```sql
create policy "Users read own projects"
  on public.projects for select
  using (owner_id = auth.uid());

create policy "Members read projects"
  on public.projects for select
  using (
    exists (
      select 1 from public.project_members
      where project_id = id and user_id = auth.uid()
    )
  );
```

### INSERT / UPDATE / DELETE Policies

```sql
create policy "Users create projects"
  on public.projects for insert
  with check (owner_id = auth.uid());

create policy "Admins create tasks"
  on public.tasks for insert
  with check (
    exists (
      select 1 from public.project_members
      where project_id = tasks.project_id
        and user_id = auth.uid()
        and role in ('owner', 'admin')
    )
  );

create policy "Owners update projects"
  on public.projects for update
  using (owner_id = auth.uid())
  with check (owner_id = auth.uid());

create policy "Owners delete projects"
  on public.projects for delete
  using (owner_id = auth.uid());
```

### Service Role Bypass

The `service_role` key bypasses RLS entirely. Use only in trusted server-side environments.

```typescript
import { createClient } from "@supabase/supabase-js"

const supabaseAdmin = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
)
```

### Testing RLS Policies

```sql
set request.jwt.claims = '{"sub": "user-uuid-here", "role": "authenticated"}';
set role authenticated;
select * from public.projects;  -- should return only the user's rows
reset role;
set request.jwt.claims = '';
```

### RLS with Custom Claims

```sql
create policy "Org admins manage members"
  on public.project_members for all
  using ((auth.jwt() -> 'app_metadata' ->> 'org_role') = 'admin');
```

### Anti-Patterns: RLS

- **Forgetting to enable RLS on a new table.** Every public table needs it before any data is inserted.
- **SELECT policies with subqueries on tables that lack their own RLS.** The subquery runs under the same role, leaking data.
- **Using `for all` when different operations need different rules.** Write separate policies per operation.
- **Not testing policies after writing them.** Always verify with `set role authenticated` and a test JWT.

---

## 3. Authentication

### Email/Password and OAuth

```typescript
// Sign up
const { data, error } = await supabase.auth.signUp({
  email: "user@example.com",
  password: "securePassword123!",
  options: { data: { full_name: "Jane Doe" } },
})

// Sign in
const { data, error } = await supabase.auth.signInWithPassword({
  email: "user@example.com",
  password: "securePassword123!",
})

// Google OAuth
const { data, error } = await supabase.auth.signInWithOAuth({
  provider: "google",
  options: {
    redirectTo: "https://myapp.com/auth/callback",
    scopes: "openid profile email",
  },
})
```

### OAuth Callback (Next.js App Router)

```typescript
// app/auth/callback/route.ts
import { createServerClient } from "@supabase/ssr"
import { cookies } from "next/headers"
import { NextResponse } from "next/server"

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url)
  const code = searchParams.get("code")

  if (code) {
    const cookieStore = await cookies()
    const supabase = createServerClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
      {
        cookies: {
          getAll() { return cookieStore.getAll() },
          setAll(cookiesToSet) {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options))
          },
        },
      }
    )
    const { error } = await supabase.auth.exchangeCodeForSession(code)
    if (!error) return NextResponse.redirect(`${origin}/`)
  }
  return NextResponse.redirect(`${origin}/auth/error`)
}
```

### Magic Links and OTP

```typescript
await supabase.auth.signInWithOtp({
  email: "user@example.com",
  options: { emailRedirectTo: "https://myapp.com/auth/callback" },
})

// Phone OTP
await supabase.auth.signInWithOtp({ phone: "+15551234567" })
const { data } = await supabase.auth.verifyOtp({
  phone: "+15551234567", token: "123456", type: "sms",
})
```

### Custom Claims via Auth Hook

```sql
create or replace function public.custom_access_token_hook(event jsonb)
returns jsonb as $$
declare
  claims jsonb;
  org_role text;
begin
  claims := event -> 'claims';
  select role into org_role from public.organization_members
    where user_id = (event ->> 'user_id')::uuid limit 1;
  claims := jsonb_set(claims, '{app_metadata, org_role}',
    to_jsonb(coalesce(org_role, 'member')));
  return jsonb_set(event, '{claims}', claims);
end;
$$ language plpgsql stable security definer;

grant execute on function public.custom_access_token_hook to supabase_auth_admin;
revoke execute on function public.custom_access_token_hook from authenticated, anon, public;
```

Register the hook in Dashboard > Authentication > Hooks > Customize Access Token.

### Session Management

```typescript
const { data: { subscription } } = supabase.auth.onAuthStateChange(
  (event, session) => {
    if (event === "SIGNED_IN") console.log("Signed in:", session?.user.id)
    if (event === "TOKEN_REFRESHED") console.log("Token refreshed")
  }
)
subscription.unsubscribe()  // cleanup
```

### Anti-Patterns: Auth

- **Storing `service_role` key in client-side code.** It bypasses RLS and grants full database access.
- **Not handling `TOKEN_REFRESHED`.** Stale tokens cause silent auth failures.
- **Disabling email confirmation in production.** Unverified emails create security gaps.
- **Custom session storage instead of `@supabase/ssr`.** Use the official SSR helpers for server-rendered apps.

---

## 4. Edge Functions

### Basic Edge Function with Auth

```typescript
// supabase/functions/create-checkout/index.ts
import { serve } from "https://deno.land/std@0.177.0/http/server.ts"
import { createClient } from "https://esm.sh/@supabase/supabase-js@2"

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
}

serve(async (req: Request) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders })
  }

  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_ANON_KEY")!,
    { global: { headers: { Authorization: req.headers.get("Authorization")! } } }
  )

  const { data: { user }, error } = await supabase.auth.getUser()
  if (error || !user) {
    return new Response(JSON.stringify({ error: "Unauthorized" }),
      { status: 401, headers: { ...corsHeaders, "Content-Type": "application/json" } })
  }

  const { priceId } = await req.json()
  return new Response(
    JSON.stringify({ userId: user.id, priceId }),
    { status: 200, headers: { ...corsHeaders, "Content-Type": "application/json" } }
  )
})
```

### Secrets and Invocation

```bash
supabase secrets set STRIPE_SECRET_KEY=sk_live_...
supabase secrets list
```

```typescript
// Invoke from client
const { data, error } = await supabase.functions.invoke("create-checkout", {
  body: { priceId: "price_abc123" },
})
```

### Scheduled Functions (pg_cron)

```sql
create extension if not exists pg_cron;

select cron.schedule('daily-cleanup', '0 0 * * *', $$
  select net.http_post(
    url := 'https://your-project.supabase.co/functions/v1/daily-cleanup',
    headers := jsonb_build_object(
      'Authorization', 'Bearer ' || current_setting('app.settings.service_role_key')
    ),
    body := '{}'::jsonb
  );
$$);
```

### Anti-Patterns: Edge Functions

- **Not handling CORS preflight.** Every browser-called function must respond to OPTIONS.
- **Mixing user JWT and service role on the same client.** Use one or the other per client instance.
- **Direct npm imports instead of esm.sh.** Deno requires URL imports: `https://esm.sh/package@version`.
- **Heavy imports causing cold-start latency.** Keep imports minimal; lazy-load heavy dependencies.

---

## 5. Realtime

### Postgres Changes

```sql
-- Enable Realtime on tables
alter publication supabase_realtime add table public.tasks;
alter publication supabase_realtime add table public.projects;
```

```typescript
// Listen to inserts with a filter
const channel = supabase
  .channel("project-tasks")
  .on("postgres_changes",
    { event: "INSERT", schema: "public", table: "tasks",
      filter: `project_id=eq.${projectId}` },
    (payload) => console.log("New task:", payload.new)
  )
  .on("postgres_changes",
    { event: "UPDATE", schema: "public", table: "tasks",
      filter: `project_id=eq.${projectId}` },
    (payload) => console.log("Updated:", payload.new)
  )
  .subscribe()
```

### Broadcast

Ephemeral messages between clients without database writes. Use for cursors, typing indicators, collaborative editing.

```typescript
const channel = supabase.channel("room-1")

channel.on("broadcast", { event: "cursor-move" }, (payload) => {
  console.log("Cursor:", payload.payload)
})

channel.subscribe((status) => {
  if (status === "SUBSCRIBED") {
    channel.send({
      type: "broadcast",
      event: "cursor-move",
      payload: { x: 120, y: 340, userId: "user-123" },
    })
  }
})
```

### Presence

Track online users and share state between connected clients.

```typescript
const channel = supabase.channel("room-1")

channel.on("presence", { event: "sync" }, () => {
  const onlineUsers = Object.values(channel.presenceState()).flat()
  console.log("Online:", onlineUsers)
})

channel.subscribe(async (status) => {
  if (status === "SUBSCRIBED") {
    await channel.track({
      user_id: currentUser.id,
      name: currentUser.name,
      online_at: new Date().toISOString(),
    })
  }
})
```

### Cleanup

```typescript
supabase.removeChannel(channel)   // single channel
supabase.removeAllChannels()       // all channels
```

### Anti-Patterns: Realtime

- **Not adding the table to `supabase_realtime` publication.** Changes will not fire without it.
- **Subscribing to all changes on high-traffic tables without a filter.** Use the `filter` parameter.
- **Not unsubscribing on component unmount.** Leaked subscriptions cause memory leaks.
- **Using Postgres Changes for ephemeral data.** Use Broadcast for cursors and typing indicators.

---

## 6. Storage

### Buckets and Upload/Download

```sql
insert into storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
values ('avatars', 'avatars', true, null, null),
       ('documents', 'documents', false, 10485760, array['application/pdf', 'image/png']);
```

```typescript
// Upload
await supabase.storage.from("avatars")
  .upload(`${userId}/avatar.png`, file, { cacheControl: "3600", upsert: true })

// Download
const { data } = await supabase.storage.from("documents")
  .download("reports/q4-summary.pdf")

// Public URL
const { data } = supabase.storage.from("avatars")
  .getPublicUrl(`${userId}/avatar.png`)

// Image transformation
const { data } = supabase.storage.from("avatars")
  .getPublicUrl(`${userId}/avatar.png`, {
    transform: { width: 200, height: 200, resize: "cover", quality: 80 },
  })
```

### RLS on Storage

```sql
create policy "Users upload own avatars"
  on storage.objects for insert
  with check (
    bucket_id = 'avatars'
    and auth.uid()::text = (storage.foldername(name))[1]
  );

create policy "Public avatar access"
  on storage.objects for select
  using (bucket_id = 'avatars');

create policy "Users delete own avatars"
  on storage.objects for delete
  using (
    bucket_id = 'avatars'
    and auth.uid()::text = (storage.foldername(name))[1]
  );
```

### Signed URLs and Resumable Uploads

```typescript
// Signed URL for private files (1 hour expiry)
const { data } = await supabase.storage.from("documents")
  .createSignedUrl("reports/q4-summary.pdf", 3600)

// Resumable upload with tus-js-client
import * as tus from "tus-js-client"

const upload = new tus.Upload(file, {
  endpoint: `${supabaseUrl}/storage/v1/upload/resumable`,
  headers: { authorization: `Bearer ${session?.access_token}` },
  metadata: {
    bucketName: "documents",
    objectName: "large-file.zip",
    contentType: "application/zip",
  },
  chunkSize: 6 * 1024 * 1024,
  onProgress: (bytesUploaded, bytesTotal) => {
    console.log(`${((bytesUploaded / bytesTotal) * 100).toFixed(1)}%`)
  },
})
upload.start()
```

### Anti-Patterns: Storage

- **No `file_size_limit` or `allowed_mime_types` on buckets.** Users can upload arbitrarily large files of any type.
- **Public buckets for sensitive documents.** Use private buckets with signed URLs for confidential content.
- **Hardcoded file paths without user-scoping.** Namespace by user/project ID to prevent collisions and simplify RLS.
- **Signed URLs with long expiration.** Keep expiry short (minutes to hours) for sensitive content.

---

## 7. Client SDK Patterns

### Typed Client Setup

```typescript
import { createClient } from "@supabase/supabase-js"
import type { Database } from "./database.types"

export const supabase = createClient<Database>(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)
```

```bash
# Generate types from your schema
npx supabase gen types typescript --project-id your-ref > database.types.ts
```

### Query Builder

```typescript
// Select with joins, filters, pagination
const { data, count } = await supabase
  .from("projects")
  .select("id, name, status, owner:owner_id(id, email)", { count: "exact" })
  .eq("status", "active")
  .order("created_at", { ascending: false })
  .range(0, 19)

// Insert and return
const { data } = await supabase.from("projects")
  .insert({ name: "New Project", owner_id: userId })
  .select().single()

// Upsert
await supabase.from("project_members")
  .upsert({ project_id: pid, user_id: uid, role: "admin" },
    { onConflict: "project_id,user_id" })

// Call a database function
await supabase.rpc("archive_project", { p_project_id: projectId })
```

### Error Handling

```typescript
const { data, error } = await supabase.from("projects").select("*").eq("status", "active")
if (error) {
  if (error.code === "PGRST116") return []  // no rows — not an error
  throw new Error(`Fetch failed: ${error.message}`)
}
```

### Next.js Server Components

```typescript
// lib/supabase/server.ts
import { createServerClient } from "@supabase/ssr"
import { cookies } from "next/headers"

export async function createSupabaseServer() {
  const cookieStore = await cookies()
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return cookieStore.getAll() },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options))
          } catch { /* read-only in Server Components */ }
        },
      },
    }
  )
}
```

### Auth Middleware (Next.js)

```typescript
// middleware.ts
import { createServerClient } from "@supabase/ssr"
import { NextResponse, type NextRequest } from "next/server"

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request })
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return request.cookies.getAll() },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value))
          response = NextResponse.next({ request })
          cookiesToSet.forEach(({ name, value, options }) =>
            response.cookies.set(name, value, options))
        },
      },
    }
  )
  const { data: { user } } = await supabase.auth.getUser()
  if (!user && request.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", request.url))
  }
  return response
}

export const config = { matcher: ["/dashboard/:path*", "/settings/:path*"] }
```

### Anti-Patterns: Client SDK

- **New client on every request/render.** Initialize once and reuse. In SSR, create one per request via helpers.
- **Not generating types.** Run `supabase gen types` after every migration for autocomplete and safety.
- **Ignoring the `error` field.** Always check it; a null `data` without error checking hides failures.
- **`select("*")` when you need three columns.** Specify columns to reduce payload and avoid leaking data.

---

## 8. Migration from Firebase

### Firestore to Postgres Mapping

| Firestore | Postgres / Supabase |
|---|---|
| Collection | Table |
| Document | Row |
| Subcollection | Related table with foreign key |
| Document ID | `uuid primary key default gen_random_uuid()` |
| Nested map | `jsonb` column or normalized table |
| Array field | `jsonb` array or junction table |
| Timestamp | `timestamptz` |
| GeoPoint | PostGIS `geography(Point, 4326)` |
| Reference | `uuid` foreign key |
| Security Rules | Row-Level Security policies |
| `onSnapshot` | Supabase Realtime |
| Cloud Functions | Edge Functions |

### Auth Migration

```typescript
// 1. Export Firebase users
import * as admin from "firebase-admin"
admin.initializeApp()

async function exportUsers() {
  const users: admin.auth.UserRecord[] = []
  let pageToken: string | undefined
  do {
    const result = await admin.auth().listUsers(1000, pageToken)
    users.push(...result.users)
    pageToken = result.pageToken
  } while (pageToken)

  return users.map((u) => ({
    email: u.email,
    display_name: u.displayName,
    firebase_uid: u.uid,
    email_verified: u.emailVerified,
  }))
}

// 2. Import into Supabase
const supabaseAdmin = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY)

for (const user of exportedUsers) {
  const { data } = await supabaseAdmin.auth.admin.createUser({
    email: user.email,
    email_confirm: user.email_verified,
    user_metadata: { full_name: user.display_name, firebase_uid: user.firebase_uid },
  })
  await supabaseAdmin.from("profiles").insert({
    id: data.user.id, name: user.display_name ?? "User",
  })
}
```

### Storage Migration

```typescript
import * as admin from "firebase-admin"
const bucket = admin.storage().bucket()

async function migrateStorage(prefix: string, supabaseBucket: string) {
  const [files] = await bucket.getFiles({ prefix })
  for (const file of files) {
    const [buffer] = await file.download()
    const { error } = await supabaseAdmin.storage
      .from(supabaseBucket)
      .upload(file.name, buffer, {
        contentType: file.metadata.contentType, upsert: true,
      })
    if (error) console.error(`Failed: ${file.name}`, error.message)
  }
}
```

### Realtime Migration

```typescript
// Firebase (before)
const unsubscribe = onSnapshot(
  query(collection(db, "tasks"), where("projectId", "==", pid)),
  (snapshot) => {
    snapshot.docChanges().forEach((change) => {
      console.log(change.type, change.doc.data())
    })
  }
)

// Supabase (after)
const channel = supabase.channel("project-tasks")
  .on("postgres_changes",
    { event: "*", schema: "public", table: "tasks",
      filter: `project_id=eq.${pid}` },
    (payload) => console.log(payload.eventType, payload.new))
  .subscribe()
```

### Incremental Migration Approach

**Phase 1 — Database (weeks 1-2):** Design Postgres schema from Firestore collections. Write migration scripts per collection. Verify row counts and data integrity. Dual-write to both systems.

**Phase 2 — Auth (weeks 2-3):** Export and import users. Configure matching OAuth providers in Supabase Dashboard. Switch login flows. Keep a Firebase UID mapping table for cross-referencing.

**Phase 3 — Storage (weeks 3-4):** Create buckets with RLS policies. Run the migration script. Update client code to Supabase Storage URLs. Verify file integrity with checksums.

**Phase 4 — Realtime and Functions (weeks 4-5):** Replace `onSnapshot` with Supabase Realtime. Rewrite Cloud Functions as Edge Functions. Migrate scheduled functions to `pg_cron`.

**Phase 5 — Cutover (weeks 5-6):** Stop dual-writing. Remove Firebase SDK dependencies. Monitor error rates for two weeks. Decommission the Firebase project after stability period.

### Anti-Patterns: Migration

- **Big-bang migration without dual-write.** Run both systems in parallel for a rollback path.
- **Not preserving `firebase_uid`.** Store it in `user_metadata` for cross-referencing during transition.
- **Assuming Firestore rules translate to RLS.** Write and test every policy from scratch.
- **Keeping Firestore's denormalized patterns.** Normalize in Postgres to avoid update anomalies.
