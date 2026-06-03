# PHASE 4 — Frontend Dashboard & Security

```
Owner:        Member 4 — Frontend & Security Engineer
Runs after:   Scaffolding (steps 1–3) can begin as soon as Phase 1's repo skeleton + shared/
              exist. Final data integration (steps 4–6) needs Phase 3's incident data + /api.
Enables:      Phase 5 validation (the dashboard must render incident state under auth).
Parallel:     Built alongside Phase 3.
How to use:   Open Claude Code at the repo root and paste this whole file as your message.
```

---

You are building OpsSentinel's **management & observability dashboard** in `frontend/`: a **Next.js
(App Router)** app, optimized for serverless **Cloud Run**, secured with **Google Identity
Services** (cryptographically verified ID tokens), containerized with a **multi-stage Alpine**
Dockerfile (standalone output, non-root), and fronted by **Cloud CDN** via a **Serverless NEG +
Layer-7 load balancer**. The dashboard reads the incident state and metrics that the Phase-3 agent
writes, and renders incident detail **server-side** so there are **zero unhandled client-side
exceptions**.

Work in small, reviewable steps and pause at each checkpoint. **Share types with the backend** by
generating TypeScript types from `shared/json-schema/` (do not hand-duplicate schemas).

## Context you must honor
- **Next.js App Router**, configured in **standalone output mode** (the compiler traces deps and
  bundles only what production needs — smaller image, faster cold start).
- **Docker:** multi-stage build on **Alpine Linux**, running as a **non-root** user (mitigates
  privilege escalation). Industry-standard Dockerfile over Buildpacks for granular control.
- **Auth:** Google Identity Services on the client → send the **ID token** to the backend as
  `Authorization: Bearer <id_token>`. The backend verifies it with the official Google library and
  uses the **`sub` claim** as the immutable user key. Keep authentication separate from
  infrastructure authorization scopes.
- **Delivery:** **Serverless NEG** connects the L7 load balancer to the Cloud Run service; **Cloud
  CDN (Pull strategy + TTL headers)** caches static assets after first fetch.
- **Render incident state with server-side data fetching** (React Server Components / route
  handlers) so the page never throws on the client during the Phase-5 demo.

## Tasks

### 1. Scaffold the Next.js App Router app  *(can start immediately)*
Create `frontend/` with a Next.js (App Router) + TypeScript project, `output: 'standalone'`,
ESLint/Prettier, and a clean component structure. Add it to `docker-compose.yml` (dev server) in
the Phase-1 slot. Generate TS types from `shared/json-schema/` into `frontend/lib/types/`.

### 2. Authentication with Google Identity Services  *(can start immediately)*
Implement Google sign-in on the client; obtain the **ID token**; attach it as a Bearer token on
every backend call. Add a server-side guard so unauthenticated users can't reach dashboard routes.
Document the OAuth client-id config (`GOOGLE_OAUTH_CLIENT_ID`). Provide a backend **token-verify
helper** (or coordinate with Member 1/2) that verifies the token via the official Google client
library and extracts `sub` — and make the dashboard render only after a verified session.

### 3. Build the UI shell with mock data  *(can start immediately)*
Build the layout and pages against a **typed mock** of the `/api` responses so UI work isn't blocked
on the live backend:
- **Incident list** — filter by `status` and `severity`; show severity, category, confidence,
  `autonomy_tier`, risk, age.
- **Incident detail** — root cause, correlated evidence/sources, confidence, historical match,
  proposed remediation steps, risk level, autonomy/caution note, audit timeline, and a **deep link
  to the Arize Phoenix trace** (`trace_id`).
- **Reliability/metrics dashboard** — MTTD, MTTR, triage accuracy, correlation precision,
  autonomous approval rate, **Arize calibration error**, and **autonomy coverage** per category.
- **Connected-tools / health** strip (Elastic, PagerDuty, ServiceNow, Slack, Arize).

### 4. Wire to the real backend API  *(needs Phase 3)*
Replace the mock with **server-side fetches** to `GET /api/incidents`, `GET /api/incidents/:id`,
`GET /api/metrics`, `GET /api/health`, forwarding the verified Bearer token. Render incident detail
in a Server Component (SSR) so first paint is exception-free. Add loading/error boundaries.

### 5. Containerize for Cloud Run
Write the **multi-stage Alpine** Dockerfile: a build stage, then a minimal runtime stage copying the
**standalone** output, running as a **non-root** user, exposing the Cloud Run port. Keep the image
lean. Verify it builds and runs locally.

### 6. Deploy via Serverless NEG + L7 LB + Cloud CDN
Using Member 1's `infra/networking/` and `infra/cloud-run/` scripts, deploy the container to Cloud
Run, attach the **Serverless NEG** to the **L7 HTTPS load balancer**, and enable **Cloud CDN (Pull +
TTL)** for static assets. Document the apply order and the cache headers.

## Tech specifics
Next.js (App Router) + TypeScript, `output: 'standalone'` · Google Identity Services + official
Google token-verification library · multi-stage **Alpine** Dockerfile, non-root · Cloud Run ·
Serverless NEG · external HTTPS (L7) load balancer · Cloud CDN (Pull, TTL). Reuse `shared/json-schema`.

## Definition of Done (verify before handoff)
- App runs locally via docker-compose; unauthenticated users are blocked; Google sign-in yields a
  verified session.
- With Phase 3 live, the incident list and **server-rendered** incident detail show real data with
  **zero unhandled client-side exceptions**; the Arize trace deep-link works.
- The metrics page renders the success-criteria metrics.
- The Alpine multi-stage image builds, runs as non-root, and deploys to Cloud Run behind the NEG +
  L7 LB with Cloud CDN serving static assets.

## Handoff to Phase 5
A deployed, authenticated dashboard that renders live incident state and metrics server-side — ready
for the end-to-end demo.

## Guardrails — do NOT build here
No backend reasoning, no MCP servers, no Slack approval logic, no `localStorage`/browser-storage for
app state (use server state / React state). None of the deferred features.
