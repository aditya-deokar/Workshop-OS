# ADR-005: Serve Inngest via the official `inngest/express` adapter inside NestJS

- Status: Accepted (pending [SPIKE S1](../planning/roadmap.md#phase-0-technical-spikes))
- Date: 2026-11-03
- Full investigation: [architecture/nestjs-inngest-integration.md](../architecture/nestjs-inngest-integration.md)

## Context

The PRD's original architecture assumed **Next.js API routes** (`inngest/next`). The chosen backend is **NestJS**. The Next.js integration **must not be assumed to work unchanged**:

- `inngest/next` exports route handlers shaped for the Next.js App/Pages Router request objects. NestJS does not produce those objects.
- Next.js examples assume function modules are imported at module scope. NestJS services live in a DI container and are only available after `NestFactory.create()`.
- Next.js on Vercel examples assume serverless timeouts and `checkpointing.maxRuntime` tuning. We run long-lived containers.

What the official docs say **[VERIFIED]** ([Serving Inngest functions](https://www.inngest.com/docs/learn/serving-inngest-functions)):

- There is **no dedicated NestJS adapter**. For NestJS, the docs list **`inngest/express`**.
- The documented NestJS pattern registers the handler in **`main.ts`** with `app.use("/api/inngest", serve({ client, functions }))`. It obtains services from the Nest container (`app.get(...)`) and passes them into a function factory.
- Express requires **JSON body parsing** before the handler ("otherwise your functions won't be executed"). The default 100 kb `express.json()` limit should be raised. The docs say to allow requests **up to 4 MB**, and their NestJS example uses 10 MB.
- `streaming: true` is supported for Express, for longer-running requests.
- v4 moved `signingKey`, `baseUrl`, etc. from `serve()` to the `Inngest` client. The default mode is `cloud` and requires a signing key; dev uses `INNGEST_DEV=1` **[VERIFIED]** ([v3→v4 migration](https://www.inngest.com/docs/reference/typescript/v4/migrations/v3-to-v4)).

## Decision

1. Use the **Express platform** (`@nestjs/platform-express`) for NestJS. Not Fastify, because the Better Auth NestJS integration's Fastify support is beta. See [ADR-006](006-auth-better-auth-and-body-parsing.md).
2. Mount `serve()` from **`inngest/express`** at **`/api/inngest`** in `main.ts`, **only when `APP_ROLE` ∈ {`worker`, `all`}**.
3. Build an in-house **`InngestModule`** (~150–250 LOC) that:
   - provides a singleton `Inngest` client (`INNGEST_CLIENT` token)
   - discovers function definitions from providers marked `@InngestFunctions()` using Nest's `DiscoveryService`
   - exposes `InngestRegistry.all()` for `main.ts` to pass to `serve()`
4. Function handlers get their dependencies **by closure** from **singleton** Nest providers. Request-scoped providers are forbidden inside Inngest functions. Cross-cutting concerns (tracing context, logger binding, Sentry capture, PII encryption) use **v4 class-based middleware** (`Middleware.BaseMiddleware` with `transformFunctionInput`) **[VERIFIED]** ([DI middleware](https://www.inngest.com/docs/durable-execution/guides-and-advanced/middleware/dependency-injection)).
5. The endpoint **bypasses Nest guards, interceptors, pipes, exception filters and the global prefix**, because it is mounted as raw Express middleware. This is intentional. Authentication is Inngest's request signing (`INNGEST_SIGNING_KEY`, verified by the SDK). The endpoint must **never** sit behind the session guard.
6. Do **not** adopt unofficial wrappers (`nestjs-inngest`, `@torixtv/nestjs-inngest`, `nest-inngest`) **[COMMUNITY]**. They add a dependency that may lag SDK majors (v4 shipped March 2026 with breaking changes) for little gain.
7. Use **serve (HTTP) mode** in the MVP. **Connect mode** (`inngest/connect`, outbound WebSocket worker, Node ≥ 22.4) is a documented alternative for long-lived workers that need no inbound endpoint. It supports self-hosting via a gateway on port 8289 **[VERIFIED]** ([Connect](https://www.inngest.com/docs/setup/connect)). Search results describe it as Public Beta and migration from `serve` apps is still limited, so it is **deferred** and re-evaluated in phase 6.

## SPIKE S1 acceptance criteria (must pass before Phase 2)

1. NestJS 11 (Express 5) + `inngest@4.x` `serve()` from `inngest/express` registers functions with the Inngest Dev Server and with an Inngest Cloud branch environment.
2. Function sync via PUT to the serve endpoint after deploy works from CI **[ASSUMPTION: PUT-based sync mechanism; verify in v4 docs]**.
3. Signature verification succeeds with the body parser configuration from ADR-006. It also succeeds when the global JSON parser is disabled and a route-scoped `express.json({ limit: '10mb' })` is used. Signature verification fails (401) for tampered requests.
4. A function using a Nest singleton service through closure works across `step.sleep('1m')` and a worker **restart** in the middle of the sleep.
5. `step.run` retries surface in our `action_executions` table through middleware/hooks without double-recording.
6. `streaming: true` behaves correctly behind our chosen load balancer (no buffering).
7. OpenTelemetry context propagates from the step into adapter HTTP calls.
8. Unit test a function with `@inngest/test` against v4 **[ASSUMPTION: v4-compatible release exists]**.

If any criterion fails, the fallback order is: (a) custom handler built on `InngestCommHandler` (documented for custom frameworks), (b) a separate plain-Express worker process that shares the domain modules through a NestJS *standalone application context* (`NestFactory.createApplicationContext`), (c) Connect mode.

## Consequences

- ✅ Uses only officially documented integration surfaces.
- ✅ Domain services are reused unchanged inside durable functions.
- ⚠️ Body-parser ordering is subtle (Better Auth vs Inngest vs webhook raw bodies). It is centralized in one `bootstrap/http-pipeline.ts` with tests.
- ⚠️ The Inngest endpoint must be publicly reachable over HTTPS from Inngest Cloud. It needs its own WAF/rate rules, and only the worker role exposes it.
