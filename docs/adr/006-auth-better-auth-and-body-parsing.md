# ADR-006: Better Auth for staff authentication, with per-route body parsing

- Status: Accepted (pending SPIKE S2)
- Date: 2026-11-03

## Context

Users are internal staff only (owner, admins, operators, viewers; possibly trainers). Requirements: email+password, optional Google sign-in, 2FA for privileged roles, sessions for a SPA, RBAC, audit.

Compatibility facts **[VERIFIED]**:

- Better Auth's Express integration must be mounted **before** `express.json()` because "body parsers consume the incoming request stream" ([Express integration](https://www.better-auth.com/docs/integrations/express)). Express 5 uses the catch-all path syntax `/api/auth/*splat`.
- Better Auth's NestJS page uses the **community-maintained** package `@thallesp/nestjs-better-auth` and requires `NestFactory.create(AppModule, { bodyParser: false })` ([NestJS integration](https://better-auth.com/docs/integrations/nestjs)).
- Inngest's Express handler requires `express.json()` with a raised limit (≥ 4 MB) ([ADR-005](005-nestjs-inngest-express-serve.md)).
- Zoom webhook signatures are an HMAC over the **exact raw request body** ([Zoom webhooks](https://developers.zoom.us/docs/api/webhooks/)). Our comms callbacks use the same model.

These requirements conflict when a single global body parser is used.

## Decision

1. **Better Auth** with the Drizzle adapter, `emailAndPassword`, optional Google social sign-in (restricted to the client's domain), the `twoFactor` plugin for admin/owner, and built-in rate limiting. Roles live in our own `staff_roles` model. We do not use Better Auth's `organization` plugin (no multi-tenancy).
2. Create Nest with **`bodyParser: false`** and build the HTTP pipeline explicitly, in this order:

| Order | Path | Parser | Handler |
|---|---|---|---|
| 1 | `/api/auth/*splat` | **none** | Better Auth `toNodeHandler(auth)` |
| 2 | `/api/inngest` (worker role only) | `express.json({ limit: '10mb' })` | Inngest `serve()` |
| 3 | `/webhooks/*` | `express.raw({ type: '*/*', limit: '2mb' })`; raw `Buffer` kept on `req.rawBody` | Nest webhook controllers (verify signature over raw bytes, then `JSON.parse`) |
| 4 | everything else (`/api/v1/*`) | `express.json({ limit: '1mb' })` | Nest controllers + `SessionGuard` |

3. Do **not** depend on `@thallesp/nestjs-better-auth` for the MVP. Mount the handler ourselves and write a ~50-line `SessionGuard` that calls `auth.api.getSession({ headers })`. This keeps full control of the pipeline order above. Re-evaluate the package if it adds route-scoped body parser support we can verify.
4. The SPA and the API are served from the **same site** (e.g. `ops.client.com` for the SPA and `ops.client.com/api` through the reverse proxy). Cookies are `HttpOnly; Secure; SameSite=Lax`. CSRF relies on Better Auth's origin checks plus `trustedOrigins`.

## SPIKE S2 acceptance criteria

- Sign-in, sign-out, session refresh and 2FA work with Express 5 and the pipeline above.
- An Inngest signed request and a Zoom signed webhook both validate in the same process (`APP_ROLE=all`).
- `SessionGuard` rejects missing/expired sessions. RBAC decorator denies under-privileged roles.

## Alternatives considered

| Option | Notes |
|---|---|
| `@thallesp/nestjs-better-auth` | Convenient decorators, but global `bodyParser: false` handling is opaque; community-maintained |
| Passport + `express-session` | Mature, official `@nestjs/passport`; more hand-written code for 2FA, rate limit, password reset |
| Managed IdP (Auth0, Clerk, WorkOS, Cognito) | Strong security features. Adds cost and an external dependency for ~10 staff users. A good choice if the client mandates SSO **[CONFIRM Q14]** |
| Google Workspace SSO only (OIDC) | Simplest if all staff have Workspace accounts. Can be layered on Better Auth's social sign-in |

## Consequences

- ✅ One explicit, testable HTTP pipeline.
- ⚠️ Any new route family must choose its parser deliberately. This is covered by a pipeline unit test that asserts parser order.
