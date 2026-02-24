 # SCALING GUIDE — Design-grade, action-oriented plan for `ments`

 This file is written to be read top-to-bottom by engineers and managers. It includes a short executive summary, clear diagrams, prioritized actions with commands and code examples, an implementation timeline, owners, risks, and acceptance criteria.

 Table of contents
 1. Executive summary
 2. Target architecture (diagram)
 3. Key replacements (what, why, how)
 4. Code examples (queue producer + worker)
 5. Migration plan & timeline
 6. Validation, SLOs, and runbook
 7. Risks, mitigations, and owners
 8. Appendices: GitHub Actions, k6 scenario, Dockerfile

 --------------------------------------------------------------------------------

 1) Executive summary (one paragraph)
 The `ments` app should move to a stateless web tier with a small, resilient backend stack: PgBouncer for DB pooling, Redis for queues and short caches, a dedicated worker fleet for CPU-bound media tasks (ffmpeg), and a CDN-fronted storage layer for media. Add Sentry and tracing for observability, and implement CI to prevent regressions. This reduces single-node strain, allows independent scaling of expensive workloads, and keeps Postgres healthy.

 --------------------------------------------------------------------------------


 2) Target architecture (visual)

 ```mermaid
 graph TD
   Users -->|HTTP| CDN["CDN — Edge Cache"]
   CDN -->|edge routing| Next["Next.js — Edge / Static"]
   Next -->|API call| API["Stateless API"]
   API -->|cache & enqueue| Redis["Redis — cache & queue"]
   API -->|db requests| PgBouncer["PgBouncer (pooler)"]
   PgBouncer -->|pooled connections| Postgres["Primary Postgres"]
   Postgres -->|replication| Replicas["Read Replicas"]
   Redis -->|jobs| Workers["Worker pool — ffmpeg, jobs"]
   Workers -->|store renditions| Storage["S3 / Supabase Storage"]
   Storage -->|served via CDN| CDN
 ```

 Short explanation: serve static content at the CDN/edge, keep API stateless, use Redis for short caches and durable queues, and isolate media processing into a worker fleet that uploads renditions to storage which is served via CDN.

 Diagram legend (arrow and node meanings)

 - Nodes:
   - `CDN`: edge cache and global delivery network.
   - `Next.js`: edge/static renderer and client-facing entrypoints.
   - `API`: stateless server-side functions or server instances.
   - `Redis`: short-lived caches and durable job queues.
   - `PgBouncer`: DB connection pooler for server-side connections.
   - `Postgres` / `Replicas`: primary DB and read-only replicas.
   - `Workers`: background processors handling CPU-bound tasks.
   - `Storage`: object storage for originals and renditions.

 - Arrows (flow semantics):
   - `-->` or `|HTTP|`: synchronous HTTP request/response or edge routing.
   - `|API call|`: backend API invocation from the web tier.
   - `|cache & enqueue|`: read/write to Redis for caching or enqueueing jobs.
   - `|db requests|`: SQL queries routed through PgBouncer.
   - `|jobs|`: job dispatch from Redis to workers (asynchronous).
   - `|store renditions|`: workers upload processed assets to storage.
   - `|served via CDN|`: storage objects are delivered via CDN to users.


 --------------------------------------------------------------------------------

 3) Key replacements — what, why, how (detailed)

 A. Web hosting: Managed Next / Containers
 - What: Move from local Next standalone to Vercel or containerized Next app with edge features.
 - Why: Edge caching reduces latency; managed platforms handle autoscaling and TLS.
 - How (quick): Create a Vercel project, set env vars, deploy staging, smoke test.
 - Acceptance: landing page TTFB < 150ms in target region.

 B. DB pooling: PgBouncer or Supabase pooling
 - What: Introduce a pooling layer between API and Postgres.
 - Why: Prevent "too many connections" and stabilize DB under bursts.
 - How (quick): Deploy PgBouncer, change server-side `DATABASE_URL` to point at pooler, leave client/browser code using Supabase client as-is.
 - Acceptance: under load, active Postgres connections remain below 70% of max.

 C. Durable queue: Redis + BullMQ (or SQS)
 - What: Replace fire-and-forget background HTTP calls with a queue.
 - Why: Ensures retries, visibility, and decoupled processing.
 - How (quick): Add Redis, implement `enqueueTranscode()` in server API to push job, add worker service to consume.
 - Acceptance: jobs complete or land in DLQ with error details; retries observed.

 D. Media workers (ffmpeg)
 - What: Move media processing to a worker fleet (Docker image with ffmpeg).
 - Why: Avoid CPU/memory spikes in web processes, control concurrency and resource cost.
 - How (quick): Build a small worker image that reads from Redis and runs ffmpeg; deploy to Cloud Run / ECS.
 - Acceptance: worker processes N concurrent jobs with stable CPU usage and queue lag within target.

 E. Storage + CDN
 - What: Store originals and renditions in S3 or Supabase Storage; serve via CDN.
 - Why: scalability, cost-efficiency, and global delivery.
 - How (quick): configure bucket + CDN origin; use signed URLs for private assets.
 - Acceptance: CDN hit rate > 90% for media assets.

 F. Observability & CI
 - What: Add Sentry, basic application tracing, and a CI workflow (lint+build).
 - Why: reduce MTTR and prevent shipping breaking changes.
 - How (quick): add SENTRY_DSN env, initialize SDK, add `.github/workflows/ci.yml`.
 - Acceptance: errors captured in Sentry; CI runs on PRs.

 --------------------------------------------------------------------------------

 4) Code examples — minimal, copyable

 A. Enqueue producer (server API) — TypeScript example

 ```ts
 // src/utils/queue.ts (producer)
 import { Queue } from 'bullmq';

 const connection = process.env.REDIS_URL;
 export const mediaQueue = new Queue('media', connection ? { connection } as any : undefined);

 export async function enqueueTranscode(key: string, opts: Record<string, unknown> = {}) {
   await mediaQueue.add('transcode', { key, opts }, { attempts: 3, backoff: { type: 'exponential', delay: 2000 } });
 }
```

 Usage in an API route after upload webhook or post creation:
 ```ts
 import { enqueueTranscode } from '@/utils/queue';

 // after upload completes:
 await enqueueTranscode('posts/123/original.mp4', { postId: '123' });
```

 B. Worker consumer (Node) — minimal

 ```ts
 // worker/src/index.ts
 import { Worker } from 'bullmq';
 const connection = process.env.REDIS_URL;

 const worker = new Worker('media', async job => {
   if (job.name === 'transcode') {
     const { key } = job.data as { key: string };
     // download from storage, run ffmpeg, upload renditions, update DB
   }
 }, { connection } as any);

 worker.on('failed', (job, err) => console.error('job failed', job?.id, err));
```

 C. Example ffmpeg command (H.264 + thumbnail)
 ```bash
 ffmpeg -i input.mp4 -c:v libx264 -preset veryfast -crf 23 -vf scale=-2:720 output_720.mp4
 ffmpeg -i input.mp4 -ss 00:00:01.000 -vframes 1 thumbnail.jpg
 ```

 --------------------------------------------------------------------------------

 5) Migration plan & timeline (practical)

 Week 0 (prep)
 - Add `.env.example`, install Redis locally (`docker run -d --name ments-redis -p 6379:6379 redis:7`).
 - Add `src/utils/queue.ts` (producer) and `worker/` scaffold.

 Week 1 (safe changes)
 - Enable PgBouncer; point server-side processes at it; run smoke tests.
 - Deploy worker scaffold to staging and run sample jobs.

 Week 2 (media+queue)
 - Implement direct signed uploads and webhook to enqueue jobs.
 - Wire worker to upload renditions and update DB.

 Week 3 (caching & CDN)
 - Add Redis caches for feed endpoints and precompute trending.
 - Configure CDN for storage and test cache-control headers.

 Week 4 (observability & hardening)
 - Add Sentry and basic tracing; add CI to run lint+build on PRs.
 - Run load-test and iterate on indexes and PgBouncer settings.

 Notes: adjust timeline to team bandwidth; each step should be validated in staging before production rollout.

 --------------------------------------------------------------------------------

 6) Validation, SLOs, runbook (what to check)

 Smoke tests (must pass before deploy):
 - Post creation, media upload, job enqueued, worker completes, DB updated, CDN serves rendition.
 - Auth flows (login redirect) work end-to-end.

 Load tests (targets):
 - `/api/feed` 1000 RPS for 1 minute, p95 < 500ms.
 - Worker queue depth < threshold (e.g., < 500) under steady state.

 Runbook: if queue depth spikes > SLA threshold
 1. Scale workers (increase replicas).
 2. Check failed jobs and DLQ.
 3. Inspect storage/network errors.

 --------------------------------------------------------------------------------

 7) Risks, mitigations, and owners

 - Risk: Postgres connections spike and DB becomes unavailable.
   - Mitigation: PgBouncer + connection limits; monitor connections; add alerts.
   - Owner: Infra.

 - Risk: Worker failures cause media backlog.
   - Mitigation: DLQ, retries, alerting, and autoscale workers.
   - Owner: Backend.

 - Risk: CDN misconfiguration serving stale or private content.
   - Mitigation: use signed URLs for private content, set correct Cache-Control, test invalidation flows.
   - Owner: DevOps + Backend.

 --------------------------------------------------------------------------------

 8) Appendices (copyable snippets)

 A. Minimal CI (GitHub Actions) — lint + build
 ```yaml
 name: CI
 on: [push, pull_request]
 jobs:
   build:
     runs-on: ubuntu-latest
     steps:
       - uses: actions/checkout@v4
       - uses: actions/setup-node@v4
         with:
           node-version: '18'
       - run: npm ci
       - run: npm run lint
       - run: npm run build
 ```

 B. k6 basic scenario (feed load)
 ```js
 import http from 'k6/http';
 import { sleep } from 'k6';

 export let options = { vus: 50, duration: '1m' };

 export default function () {
   http.get('https://your-staging.example.com/api/feed');
   sleep(1);
 }
 ```

 C. Worker Dockerfile (example)
 ```dockerfile
 FROM node:18
 WORKDIR /usr/src/app
 COPY package.json package-lock.json ./
 RUN npm ci --only=production
 COPY . .
 RUN npm run build
 CMD ["node", "dist/index.js"]
 ```




