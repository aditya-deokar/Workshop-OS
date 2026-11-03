# ADR-009: React + Vite SPA instead of Next.js

- Status: Accepted
- Date: 2026-11-03

## Context

The PRD's section 9 proposed a Next.js 16 admin dashboard with Next.js API/tRPC as the backend. The engineering brief selects **React + TypeScript + Vite** for the frontend and **NestJS** for the backend.

## Decision

- The admin UI is a **client-rendered SPA** built with Vite. It is an authenticated internal tool, so SEO and SSR have no value.
- Routing: **TanStack Router** (type-safe search params for filters) or React Router v7 in SPA mode. Pick one in Phase 1. TanStack Router is preferred.
- Server state: **TanStack Query**. Forms: **react-hook-form + Zod**.
- API contracts: shared **Zod schemas** in a workspace package `@wos/contracts`, used by NestJS (validation + OpenAPI) and the SPA (typed client).
- **tRPC is not used.** The brief requires REST + OpenAPI, and external systems (comms callbacks) need plain REST.
- Static assets are served from object storage/CDN or from the reverse proxy on the same site as the API (see [ADR-006](006-auth-better-auth-and-body-parsing.md) for cookies).

## Consequences

- ✅ Clear separation: all logic lives in NestJS, and the SPA is a thin, replaceable client.
- ✅ No Next.js server runtime to operate.
- ⚠️ Inngest's Next.js examples (`inngest/next`) do not apply anywhere in this system. See [ADR-005](005-nestjs-inngest-express-serve.md).
