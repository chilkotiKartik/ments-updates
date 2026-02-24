# Backend — How it works (full, easy explanation)

This document explains the webapp backend in plain language: core components, how a user request travels through the system, where the code lives, common failure points at scale, and practical mitigations.

---

## 1) High-level overview

- Platform: Next.js (server + client) — app code lives in `src/`.
- Primary backend service: Supabase (Postgres for data, Supabase Auth, Supabase Storage). The app uses both client-side and server-side Supabase helpers.
- Pattern: Route Handlers (Next.js) implement HTTP endpoints that call shared `src/api/*` modules which contain business logic and talk to Supabase (or call Postgres RPCs).
- Background / async: the app uses in-process fire-and-forget HTTP calls to internal API endpoints for notifications and feed work; media processing uses FFmpeg helpers.

Files worth opening first:
- `package.json` — dependencies and scripts.
- `src/utils/supabase-server.ts` — server helpers (`createAuthClient`, `createAdminClient`).
- `src/utils/supabase.ts` — browser supabase proxy.
- `src/app/api/*` — Next.js Route Handlers.
- `src/api/*` — higher-level services (business logic) used by route handlers.

See [src/utils/supabase-server.ts](src/utils/supabase-server.ts#L1-L40) and [src/api/posts.ts](src/api/posts.ts#L1-L20) for patterns used across the codebase.

---

## 2) Simple request lifecycle (step-by-step)

Example: user opens feed → app loads posts.

1. Browser calls a client endpoint (fetch to `/api/posts?limit=20`).
2. Next.js Route Handler `src/app/api/posts/route.ts` runs. It calls `createAuthClient()` to get a server Supabase client tied to the user's cookies.
3. The handler calls a service function (e.g., `fetchPosts`) located in `src/api/posts.ts`.
4. That service uses the Supabase client to call either:
   - A Postgres RPC (fast, pre-aggregated; e.g. `get_posts_with_counts`) or
   - A composed Supabase query with joins/aggregations (fallback if RPC missing).
5. Results are returned to the handler, which wraps them in a JSON response to the browser.

For write flows (e.g., create post): handler → service function `createPost` → `supabase.from('posts').insert(...)` → on success, code triggers fire-and-forget background calls such as `/api/feed/extract-topics` or push-notification endpoints.

Important detail: server routes use two client types:
- `createAuthClient()` — respects Row Level Security (RLS) by using user cookies.
- `createAdminClient()` — uses the Supabase service role key to bypass RLS for trusted server actions. Only use in secure server contexts.

---

## 3) Data model (high level)

Main Post-related tables you will see in queries:
- `posts` — posts table.
- `post_media` — media items linked to posts.
- `post_polls`, `post_poll_options`, `post_poll_votes` — polls and votes.
- `post_likes` — likes per post.
- `users` — user profiles and metadata.
- `environments` — logical spaces/groups used for scoping feeds.

Other supporting tables and concepts: notification tables, feed-engine tables (caches), and any custom RPCs in `supabase/migrations/*.sql`.

---

## 4) Where code is (quick map)

- Route handlers (HTTP endpoints): `src/app/api/*` (Next.js Route Handlers). Example: [src/app/api/posts/route.ts](src/app/api/posts/route.ts#L1-L40).
- Business logic / DB access: `src/api/*` (e.g., `src/api/posts.ts`).
- Supabase helpers: `src/utils/supabase-server.ts` and `src/utils/supabase.ts`.
- Media & compression helpers: `utils/` and `src/utils/*` (FFmpeg usage in `package.json`).
- DB migrations and RPC SQL: `supabase/migrations/`.

---

## 5) Auth & security

- Browser: uses `NEXT_PUBLIC_SUPABASE_ANON_KEY` to create a client; this is safe for public operations controlled by RLS.
- Server: trusted operations should use `SUPABASE_SERVICE_ROLE_KEY` only in secure server code (see `createAdminClient()`).
- RLS (Row Level Security) is expected in the database to restrict row access per-user; route handlers use `createAuthClient()` so RLS rules apply.

Security pointers:
- Never leak `SUPABASE_SERVICE_ROLE_KEY` to the browser or logs.
- Validate input in route handlers before calling admin APIs.

---

## 6) Background work & notifications

- The app triggers background work by making internal HTTP requests to other route handlers (fire-and-forget). This is simple but can be fragile at scale.
- Common background tasks:
  - Feed extraction / topic generation (`/api/feed/extract-topics`).
  - Push notifications on reply/mention (`/api/push-on-reply`, `/api/push-on-mention`).
  - Media transcoding/compression (FFmpeg helpers).

At-scale recommendation: move these to explicit job queues/workers (Redis + BullMQ, or a managed queue like AWS SQS + Lambda/Fargate) so retries, visibility, and rate controls exist.

---

## 7) Where scalability can fail (common bottlenecks) and mitigations

1) Postgres connection pressure (serverless + many concurrent Next.js instances)
   - Why it fails: each serverless instance can open many DB connections; Postgres has limited connections.
   - Symptoms: connection errors, high latency, 5xx errors.
   - Mitigations: use a connection pooler (pgBouncer) or use Supabase's connection pooling offering; move heavy read traffic to read replicas; cache results in Redis or CDN.

2) Heavy synchronous RPCs or complex joins
   - Why: long-running RPCs block and increase latency; complex queries can slow overall throughput.
   - Mitigations: precompute with materialized views or scheduled jobs; simplify RPCs; paginate aggressively; move expensive aggregation to background workers and cache results.

3) Feed generation work (N+1 queries or lack of caching)
   - Why: building a feed by querying many tables per post causes high DB load.
   - Mitigations: use a feed cache layer (Redis or dedicated feed tables), normalize queries into efficient RPCs, or maintain a denormalized feed table updated by background workers.

4) Media processing (FFmpeg) on server
   - Why: CPU- and memory-heavy; can block app servers or exhaust resources.
   - Mitigations: offload to dedicated workers/containers (Fargate, EC2 worker pool, or cloud functions), use pre-signed uploads to Storage + serverless processing, and limit concurrency.

5) Notifications smoke (push loops)
   - Why: naive fan-out of notifications can cause spikes and exceed provider quotas.
   - Mitigations: batch notifications, rate-limit, introduce backoff and retries, offload to async worker queue.

6) Serverless cold-starts and time limits (Next.js serverless functions)
   - Why: cold starts cause latency spikes and timeouts for longer RPCs.
   - Mitigations: use keep-alive strategies, prefer containerized long-running instances for heavy endpoints, or convert heavy endpoints to background jobs.

7) Lack of caching for static media
   - Why: serving media without CDN stresses origin storage.
   - Mitigations: use CDN (Edge) for images/videos, serve signed URLs from Supabase Storage and let CDN handle traffic.

Operational mitigations summary:
- Add Redis for caching and job queueing.
- Use pgBouncer or Supabase-provided pooling for DB connections.
- Move CPU-heavy tasks off app servers to dedicated workers.
- Use read replicas and materialized views for heavy read patterns.
- Introduce rate limits and circuit breakers for external providers.

---

## 8) Developer checklist — run locally

1. Create `.env.local` (or set env vars):
   - `NEXT_PUBLIC_SUPABASE_URL`
   - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `SUPABASE_SERVICE_ROLE_KEY`
   - Any storage keys used in `AWS_DEPLOYMENT.md` if replacing Supabase storage.
2. Install and run:
```bash
npm install
npm run dev
```
3. If you want the DB locally, run Supabase locally and apply migrations in `supabase/migrations/`.

---

## 9) Quick troubleshooting pointers

- 401/403 on server routes: verify `createAuthClient()` is correctly using cookies and that the browser sends `sb-web-auth` cookie.
- RPC missing errors: code falls back to manual queries but with reduced performance; inspect `supabase/migrations` to add the missing function.
- Connection errors: monitor DB connection counts and enable connection pooling.

---

## 10) Next recommended actions (practical)

1. Add a Redis instance and a job queue for feed extraction and push notifications.
2. Add pgBouncer or enable Supabase pooling if using serverless Next.js.
3. Move media processing to dedicated worker instances.
4. Create materialized views or denormalized feed tables updated by workers for fast feed reads.
5. Add observability: metrics (DB latency, request latency), alerts, and request tracing.

---

If you want, I can now:
- produce an endpoint-by-endpoint map (Route Handler → service function → DB tables/RPCs), or
- extract and summarize the DB migrations in `supabase/migrations/`, or
- write a short migration plan to add Redis + job queue.

Choose one and I’ll proceed.

---

## 12) Big-step flows (end-to-end, high-level but ordered)

These show the major systems touched for each common user action. Each item is a single flow broken into main phases — useful for architecture reviews and planning work.

A) Load user feed (big steps)
1. Client requests `/api/posts` with auth cookie.
2. Route handler validates identity with `createAuthClient()` and reads user id.
3. Service `fetchPosts()` runs: preferred path is RPC `get_posts_with_counts`; fallback is composed queries with joins and batch counts.
4. Response JSON returned to client; UI renders list of posts with likes/replies counts and media URLs.
5. (Optional) Feed cache is read/updated in Redis or in a denormalized table by background jobs.

B) Create a post (big steps)
1. Client POSTs to `/api/posts` (or calls client helper that invokes the API).
2. Route handler uses `createAuthClient()` to verify user and calls `createPost()` in `src/api/posts.ts`.
3. `createPost()` inserts into `posts` table and returns the created post row.
4. Background hooks: enqueue feed extraction, notification jobs, and media processing if applicable.
5. UI optimistically updates feed and polls backend for final state.

C) Upload media (big steps)
1. Client requests a signed upload URL from an API or uses direct Supabase Storage upload via browser client.
2. Client uploads file to Supabase Storage (signed URL or client lib).
3. On upload completion, worker (or an API route) triggers FFmpeg processing: create thumbnails, transcode video, store processed assets.
4. Worker writes media metadata to `post_media` and updates `posts.image`/`media_thumbnail` fields as needed.

D) Notifications (big steps)
1. Event occurs (reply, mention, like) and route handler or service enqueues a notification job.
2. Worker processes job, writes notification to DB, and calls push providers (APNs/FCM) or web push endpoints.
3. Worker handles retries, batching, and backoff.

E) Feed engine / recommendation (big steps)
1. Post creation or engagement events enqueue feature-extraction jobs.
2. Workers extract features (topics, embeddings) and write to feed cache tables or a vector store.
3. Recommendation jobs run candidate generation and scoring (periodically or on-demand) and store results in a denormalized feed cache.
4. Client requests read from the cached feed for low-latency responses.

---

## 13) Small-step recipes (detailed copy-paste commands and checks)

These are the exact commands and minimal checks you can run locally or in prod to perform common tasks or debug issues.

A) Start dev environment (exact)
```bash
# 1. Install deps
npm install

# 2. Export env (example Windows PowerShell)
$env:NEXT_PUBLIC_SUPABASE_URL="https://xyz.supabase.co"
$env:NEXT_PUBLIC_SUPABASE_ANON_KEY="anon_key"
$env:SUPABASE_SERVICE_ROLE_KEY="service_role_key"

# 3. Run dev server
npm run dev
```

B) Reproduce auth-protected request with curl (exact)
```bash
# First extract sb-web-auth cookie from browser devtools, then:
curl -v -H "Cookie: sb-web-auth=<COOKIE_VALUE>" "http://localhost:3000/api/posts?limit=1"
```

C) Check for RPC functions in Postgres (psql)
```sql
-- Connect with psql, then:
\df+ get_posts_with_counts
\df+ like_post
```

D) Run a migration file manually (psql)
```bash
# From repo root - example (replace host/user/db/port)
psql "postgresql://user:password@localhost:5432/postgres" -f supabase/migrations/002_feed_rpc_functions.sql
```

E) Add a Redis-backed worker (quick template)
1. Install:
```bash
npm install bullmq ioredis
```
2. Minimal worker `src/workers/feedWorker.ts` (example):
```ts
import { Worker } from 'bullmq';
import IORedis from 'ioredis';
const connection = new IORedis({ host:'127.0.0.1', port:6379 });
const worker = new Worker('extract-topics', async job => {
   const payload = job.data;
   // create admin supabase client and write results
}, { connection });
worker.on('failed', (job, err) => console.error('job failed', job.id, err));
```
3. Enqueue job from server route instead of fire-and-forget fetch:
```ts
import { Queue } from 'bullmq';
const queue = new Queue('extract-topics', { connection: { host:'127.0.0.1', port:6379 } });
await queue.add('job', { post_id: '...' });
```

F) Debug slow DB queries (quick)
1. Enable `pg_stat_statements` or use Supabase query insights.
2. Re-run slow RPC manually in psql and `EXPLAIN (ANALYZE, BUFFERS)` to inspect.

G) Validate `createAuthClient()` reads cookies (tiny test)
1. In `src/app/api/posts/route.ts` add temporary log:
```ts
const supabase = await createAuthClient();
console.log('user check', await supabase.auth.getUser());
```
2. Restart server and call endpoint with cookie; inspect terminal logs.

H) Quick rollback of a bad migration (exact)
1. Use psql to revert the SQL or restore from DB backup snapshot.
2. If using Supabase, use project backups/Point-in-Time-Recovery.

I) Add monitoring & alerts (minimum)
1. Add an APM (Datadog, NewRelic) or use Cloud provider metrics.
2. Track these alerts: DB connection count, 95th/99th latency for API, job queue failure rate, worker CPU/memory.

---

Progress update: I appended big-step flows and precise small-step recipes to `BACKEND_EXPLAINED.md`. Next I can:
- produce a full endpoint map (Route Handlers → service functions → DB tables/RPCs), or
- scaffold a runnable worker (`src/workers/*`) and update one route to enqueue jobs, or
- extract and summarize `supabase/migrations/` SQL functions.


---

## 11) Micro step-by-step tasks (small, exact steps)

These are tiny, copy-paste steps for common developer and operator tasks. Follow each numbered list in order.

A) Local developer setup
1. Create a project copy and open the repo.
2. Create `.env.local` with at least these env vars:
   - `NEXT_PUBLIC_SUPABASE_URL`
   - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `SUPABASE_SERVICE_ROLE_KEY`
3. Install dependencies:
```bash
npm install
```
4. (Optional) Run Supabase locally: install supabase CLI and run `supabase start` in a dev DB folder, then apply migrations in `supabase/migrations/`.
5. Start the app in dev:
```bash
npm run dev
```
6. Open `http://localhost:3000` and sign up / sign in. If auth fails, inspect cookies and logs.

B) Apply DB migrations (small steps)
1. Ensure `supabase` CLI is installed: `npm install -g supabase` or follow Supabase docs.
2. Login: `supabase login` and point to your project or local instance.
3. Apply migrations:
```bash
supabase db push --project-ref <your-ref>  # or use `supabase db reset` for local dev
```
4. Confirm RPCs exist by listing functions in `psql`:
```sql
\df+ get_posts_with_counts
\df+ like_post
```

C) Simple auth request trace (how a signed-in request flows)
1. Browser makes request: `GET /api/posts?limit=20`.
2. Route handler `src/app/api/posts/route.ts` calls `createAuthClient()` which reads cookies (name `sb-web-auth`).
3. Server calls `supabase.auth.getUser()` — if no user, return 401.
4. Service function `fetchPosts()` runs RPC or query and returns JSON.

D) Create post flow (tiny steps)
1. Client submits POST to `/api/posts` (or uses internal service function `createPost`).
2. Handler validates body, calls `createAuthClient()` to get user, then calls `createPost(authorId, ...)` in `src/api/posts.ts`.
3. `createPost` inserts into `posts` and returns the inserted row.
4. On success the code fires `/api/feed/extract-topics` and notification endpoints (fire-and-forget). Check network logs for these requests.

E) Like / unlike / vote flows (tiny checks)
- To like: client calls supabase RPC `like_post` or POST to an API route that calls `supabase.rpc('like_post', {...})`.
- If RPC missing, the code handles it gracefully (logs warning). To verify, run the RPC query in psql or check `supabase/migrations/` for function SQL.

F) Add Redis + job queue (minimal migration steps)
1. Provision Redis (local: `docker run -p 6379:6379 redis`).
2. Add a worker package (example stack: `bullmq` + `ioredis`). Install:
```bash
npm install bullmq ioredis
```
3. Create a new worker file `src/workers/feedWorker.ts` that connects to Redis and processes jobs (use `createAdminClient()` for DB writes).
4. Replace fire-and-forget internal calls with enqueue calls to Redis queue (e.g., `queue.add('extract-topics', payload)`).
5. Run worker(s) separately: `node src/workers/feedWorker.js` (or run in a container / process manager).

G) Deploy checklist (small steps)
1. Set env vars in your hosting provider: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` (service-role in secrets manager).
2. Ensure DB migrations and RPCs are applied in production DB.
3. If using serverless, enable connection pooling (pgBouncer or Supabase pooling) and add Redis for background jobs.

H) Observability & quick debugging steps
1. Add request logging around route handlers (log request IDs, durations).
2. To inspect slow queries: enable Postgres slow query log or use Supabase query insights.
3. To check a failing request quickly: capture full request headers and body, then reproduce with `curl`:
```bash
curl -v -H "Cookie: sb-web-auth=<cookie>" "http://localhost:3000/api/posts?limit=1"
```

I) Common troubleshooting micro-steps
- 401/403: check that `sb-web-auth` cookie is present and `createAuthClient()` reads cookies (see [src/utils/supabase-server.ts](src/utils/supabase-server.ts#L1-L40)).
- Missing RPC: check `supabase/migrations/` and run the SQL in psql to create functions.
- Connection errors: check DB connection count, enable pooling, or scale DB.
- Media failures: ensure FFmpeg deps are installed and workers have sufficient CPU/memory.

---

## Diagrams (visuals)

Below are two Mermaid diagrams you can view in any editor that supports Mermaid or render via mermaid.live.

### Architecture (component) diagram

```mermaid
flowchart LR
   Browser(Browser / Client)
   NextJS(Next.js App\nRoute Handlers\nServer Components)
   Supabase(Supabase\nPostgres / Auth / Storage)
   Workers(Workers / Job Queue\nRedis + BullMQ)
   EdgeFns(Supabase Edge Functions)
   CDN(CDN / Edge Cache)
   FF(FFmpeg Media Workers)

   Browser -->|HTTP| NextJS
   NextJS -->|Auth & Data| Supabase
   NextJS -->|Enqueue jobs| Workers
   Workers -->|Writes / Reads| Supabase
   Workers -->|Media processing| FF
   Supabase -->|Public assets| CDN
   NextJS -->|Call Edge Fns| EdgeFns
   Browser -->|Media GET| CDN

   style NextJS fill:#f9f,stroke:#333,stroke-width:1px
   style Supabase fill:#efe,stroke:#333
   style Workers fill:#eef,stroke:#333
```

### Request flow: GET /api/posts (sequence)

```mermaid
sequenceDiagram
   participant B as Browser
   participant N as Next.js Route
   participant A as AuthClient
   participant S as Supabase (RPC)
   participant DB as Postgres

   B->>N: GET /api/posts?limit=20
   N->>A: createAuthClient() (reads sb-web-auth cookie)
   A->>S: auth.getUser()
   alt user authenticated
      N->>S: call RPC get_posts_with_counts(env_id, limit, offset)
      S->>DB: execute RPC
      DB-->>S: rows
      S-->>N: posts JSON
      N-->>B: 200 OK + JSON
   else unauthorized
      N-->>B: 401 Unauthorized
   end
```
