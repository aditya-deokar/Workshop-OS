# Technical Stack: Evaluation, Compatibility and Decisions

Related: [system-architecture.md](../architecture/system-architecture.md) · [nestjs-inngest-integration.md](../architecture/nestjs-inngest-integration.md) · [ADR index](../adr/README.md)

> Every version below must be **pinned exactly** in `package.json` / lockfile at project start. "Current" means the current stable release as of November 2026. Confirm at kickoff.

---

## 1. Final stack

| Layer | Choice | Status | Notes |
|---|---|---|---|
| Runtime | **Node.js 22 LTS or 24 LTS** | Decided | ≥ 22.4 keeps Inngest Connect open as an option **[VERIFIED requirement]** |
| Language | TypeScript (strict) | Decided | |
| Monorepo | pnpm workspaces + Turborepo | Decided | |
| Frontend | React 19 + Vite | Decided ([ADR-009](../adr/009-react-vite-spa-instead-of-nextjs.md)) | SPA |
| Routing | TanStack Router | Proposed | React Router v7 SPA mode acceptable |
| Server state | TanStack Query | Decided | |
| UI | Tailwind CSS v4 + shadcn/ui | Decided | shadcn/ui components are copied into `packages/ui` |
| Forms | react-hook-form + Zod resolver | Decided | |
| Timeline builder | **dnd-kit** (`@dnd-kit/core`, `@dnd-kit/sortable`) | Decided | See §3.6 |
| Workflow editor | **React Flow** (`@xyflow/react` v12) | Decided | |
| Backend | **NestJS 11** on `@nestjs/platform-express` (Express 5) | Decided | Fastify rejected (§3.2) |
| API | REST + OpenAPI 3.1 | Decided | Generated from Zod |
| Validation | **Zod** (single schema source) | Decided | §3.4 |
| ORM | **Drizzle ORM** + drizzle-kit migrations | Decided | §3.3 |
| DB | PostgreSQL 16/17 (managed) | Decided | |
| Durable execution | **Inngest** TS SDK v4, Inngest Cloud | Decided ([ADR-002](../adr/002-inngest-orchestration.md), [ADR-005](../adr/005-nestjs-inngest-express-serve.md)) | |
| Auth | **Better Auth** (Drizzle adapter) | Decided ([ADR-006](../adr/006-auth-better-auth-and-body-parsing.md)) | Mounted manually |
| Realtime | SSE + Postgres LISTEN/NOTIFY | Decided ([ADR-010](../adr/010-realtime-sse-with-postgres-notify.md)) | |
| Files | Cloudflare R2 (S3 API) via `@aws-sdk/client-s3` | Decided | |
| Time | **Luxon** (server + client) | Proposed | §3.7 |
| Logging | pino (`nestjs-pino`) | Proposed | JSON logs with trace IDs |
| Tracing/metrics | OpenTelemetry Node SDK | Decided | §3.5 |
| Errors | Sentry (Node + React) | Decided | §3.5 |
| Testing | Vitest (+ `unplugin-swc` for Nest), Testcontainers, MSW, Playwright, k6 | Decided | §3.8 |
| Containers | Docker (multi-stage), docker compose for dev | Decided | |
| CI/CD | GitHub Actions | Proposed | |

## 2. Stack evaluation summary

| Proposed in brief | Verdict | Reason |
|---|---|---|
| React + TS + Vite | ✅ Keep | Internal tool; no SSR need |
| Tailwind + shadcn/ui | ✅ Keep | |
| NestJS | ✅ Keep, with caveats | DI and modules suit the modular monolith. **Inngest and Better Auth need explicit integration work** (§3.1, §3.2) |
| REST + OpenAPI | ✅ Keep | Generate the spec from Zod, not hand-written decorators |
| PostgreSQL | ✅ Keep | |
| Drizzle | ✅ Keep | No official Nest module, but a trivial custom provider is enough |
| Inngest | ✅ Keep | Served via `inngest/express`. **Not the Next.js integration** |
| Better Auth | ✅ Keep, mount manually | Body-parser conflict resolved by pipeline ordering |
| Zod **or** class-validator | ✅ **Zod only** | One schema source for DTOs, events, the workflow DSL and the frontend |
| dnd-kit | ✅ Keep | Drag mechanics only. Time-grid logic is ours |
| React Flow | ✅ Keep | Editor only. Execution is server-side |
| R2 / S3 | ✅ R2 | S3-compatible. No egress fees **[VERIFIED: Cloudflare pricing model; re-check]** |
| WebSockets **or** SSE | ✅ SSE | One-way updates |
| OpenTelemetry + Sentry | ✅ Keep, **configure carefully** | Sentry's Node SDK uses OpenTelemetry internally (§3.5) |
| Vitest/Jest + Playwright | ✅ Vitest (with SWC) | §3.8 |
| Docker + cloud | ✅ | See [deployment.md](deployment.md) |

## 3. Compatibility issues found (and resolutions)

### 3.1 Inngest × NestJS

- **Issue:** No NestJS adapter. Next.js examples don't apply. DI container vs module-scope functions. Global guards/filters don't wrap the handler.
- **Resolution:** `inngest/express` mounted in `main.ts`. In-house `InngestModule` with discovery and closures over singletons. v4 class middleware for cross-cutting concerns. Full analysis: [nestjs-inngest-integration.md](../architecture/nestjs-inngest-integration.md).
- **Open:** Express 5 compatibility → **SPIKE S1**.

### 3.2 Better Auth × NestJS × Inngest × webhook signatures (body parsing)

- **Issue:** Better Auth needs the raw stream (mount before body parsers; Nest `bodyParser: false`) **[VERIFIED]**. Inngest needs `express.json()` with a ≥ 4 MB limit **[VERIFIED]**. Zoom HMAC needs exact raw bytes **[VERIFIED]**.
- **Resolution:** Disable Nest's global parser. Apply route-scoped parsers ([ADR-006](../adr/006-auth-better-auth-and-body-parsing.md)).
- **Fastify:** Inngest has `inngest/fastify` **[VERIFIED]**, but Better Auth's Nest integration lists Fastify as beta **[VERIFIED]**. We stay on Express.

### 3.3 Drizzle × NestJS × Better Auth

- No official `@nestjs/drizzle`. Provide `DRIZZLE` as a custom provider (node-postgres `Pool` + `drizzle(pool, { schema })`). Close the pool in `onApplicationShutdown`.
- **Transactions across services:** pass a `tx` handle explicitly, or use an `AsyncLocalStorage`-based transaction context (`@nestjs-cls/transactional` has a Drizzle adapter **[ASSUMPTION: verify current support]**). **Decision:** explicit `tx` parameter in repository methods. It is simpler and works inside Inngest steps, where there's no request context.
- Better Auth's CLI generates its schema for Drizzle. Commit the generated tables into our schema folder and manage them through drizzle-kit migrations like everything else.
- Migrations: `drizzle-kit generate` (SQL files committed, reviewed). Applied by a dedicated migration job in CI/CD. **Never `drizzle-kit push` in shared environments.**

### 3.4 Zod as the single schema source

- NestJS's default validation is class-validator. We use a Zod validation pipe plus OpenAPI generation from Zod. Candidates: `nestjs-zod` or `@asteasolutions/zod-to-openapi`. **Check Zod v4 support of the chosen library before adopting** **[SPIKE S2]**. Fallback: generate the OpenAPI document directly from `@wos/contracts` with `zod-to-openapi` and serve it via `@nestjs/swagger`'s `SwaggerModule.setup` using the generated document object.
- Inngest v4 `eventType("name", { schema })`: confirm it accepts our Zod schemas (Standard Schema compatibility is not documented in the migration guide) **[SPIKE S1]**. Fallback: `staticSchema<T>()` for types plus explicit Zod parse in the first step.

### 3.5 OpenTelemetry × Sentry

- Recent Sentry Node SDKs (v8+) are built on OpenTelemetry and register their own tracer provider by default **[ASSUMPTION based on Sentry docs; verify version]**. Running a separate OTel SDK naively causes double instrumentation or lost spans.
- **Resolution:** Use Sentry's documented "custom OpenTelemetry setup" path, where the app owns the OTel `NodeSDK` and Sentry plugs in its span processor/sampler. Or choose one of them as the primary tracing backend. **Decision:** OTel owns tracing (exports OTLP to the chosen backend). Sentry is used for errors, with tracing integration through its OTel hooks. This is validated in **SPIKE S4**.
- Instrumentations: `@opentelemetry/instrumentation-http`, `-express`, `-nestjs-core`, `-pg`, `-undici` (for `fetch`).
- Inngest: propagate trace context into steps via our tracing middleware. Inngest also offers its own tracing features. Evaluate them in SPIKE S4 **[ASSUMPTION]**.

### 3.6 dnd-kit and React Flow

- dnd-kit has a stable `@dnd-kit/core` line and a newer rewrite (`@dnd-kit/react`). **Use the stable `@dnd-kit/core` + `@dnd-kit/sortable`** unless the rewrite is declared stable at kickoff.
- dnd-kit handles drag interactions only. The timeline's time axis, snapping (5-minute grid), collision rules and keyboard accessibility are custom.
- React Flow (`@xyflow/react`) is used only as an editor. The persisted model is **our DSL** (`@wos/contracts/workflow`), not React Flow's node JSON. A mapper converts between them, which keeps execution independent of the UI library.

### 3.7 Time handling

- `timestamptz` in Postgres. All instants are UTC on the wire (ISO 8601 with `Z`). The workshop has an IANA `timezone` (e.g. `Asia/Kolkata`).
- Luxon for zone math on both server and client, so both use identical DST logic. Native `Temporal` is a future replacement once it is available in our Node LTS.
- Never use `new Date(localString)` without a zone. A lint rule bans it.

### 3.8 Testing toolchain

- NestJS's default scaffolding uses Jest. Vitest needs **SWC** (`unplugin-swc`) because esbuild doesn't emit `emitDecoratorMetadata`, which Nest DI relies on **[VERIFIED: known esbuild limitation]**.
- Decision: **Vitest everywhere** (one runner for monorepo), with `unplugin-swc` in `apps/server`. Fallback: Jest for server only.

### 3.9 Inngest v4 breaking changes vs older tutorials

Most blog posts and community wrappers target v3. Under v4: triggers go in the options, `EventSchemas` is removed, `serveHost` → `serveOrigin`, `logLevel` is removed, `streaming: "force"` → `true`, and checkpointing is on by default **[VERIFIED]**. Engineers must use v4 docs only. A PR checklist item covers this.

## 4. Alternatives considered (by layer)

| Layer | Alternative | Why not chosen |
|---|---|---|
| Backend | Next.js API / tRPC (PRD original) | Brief mandates NestJS + REST/OpenAPI. Mixing UI and orchestration runtime is undesirable |
| Backend | Fastify-based Nest | Better Auth Nest Fastify support beta |
| Backend | Hono / plain Express | Lose Nest DI/modules that help structure a large monolith |
| ORM | Prisma | Heavier runtime. Drizzle gives SQL-level control for conditional updates, `SKIP LOCKED` and advisory locks |
| ORM | Kysely | Good alternative. Drizzle chosen for schema-as-code + migrations |
| Workflow | Temporal / pg-boss / BullMQ / Trigger.dev / DBOS | See [ADR-002](../adr/002-inngest-orchestration.md#alternatives-considered) |
| Auth | Passport / managed IdP | See [ADR-006](../adr/006-auth-better-auth-and-body-parsing.md#alternatives-considered) |
| Realtime | WebSockets / Redis | See [ADR-010](../adr/010-realtime-sse-with-postgres-notify.md) |
| Validation | class-validator | Two schema systems (backend vs frontend/events). Zod unifies them |
| Workflow editor | Custom canvas / Rete.js | React Flow is mature and fits our editor-only use |

## 5. Version pinning policy

- Exact versions (`"inngest": "4.x.y"`, no `^`), with Renovate PRs weekly.
- Major upgrades of Inngest, NestJS, Better Auth, Drizzle or Zod require: changelog review, a full integration test run, and a staging soak of ≥ 3 days with a synthetic workshop.
