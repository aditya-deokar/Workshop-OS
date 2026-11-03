# Security and Permissions

Related: [ADR-006](../adr/006-auth-better-auth-and-body-parsing.md) · [reliability-patterns.md §4, §11, §14](reliability-patterns.md) · [integration-architecture.md §8](../architecture/integration-architecture.md#8-credentials-and-oauth-lifecycle) · [api-specification.md](api-specification.md)

---

## 1. Threat model (summary)

| Asset | Threats | Key controls |
|---|---|---|
| Participant PII (names, emails, phones, join links) | Exfiltration, over-sharing to third parties, logging leaks | RBAC, masking, IDs-only events to Inngest, encryption at rest, export audit |
| Integration credentials (Zoom S2S, Google refresh tokens, comms secrets) | Theft → impersonation of the business on Zoom/Google/WhatsApp | Envelope encryption + KMS, never returned by API, least scopes, rotation |
| Ability to message hundreds of participants | Malicious/accidental mass sends | RBAC, previews, emergency stop, throttles, audit, unknown-outcome safety |
| Webhook endpoints | Forged events (fake attendance, fake "meeting ended") | HMAC signatures, timestamp windows, dedupe |
| Inngest endpoint | Forged step calls | Inngest signing key verification. Only on worker role |
| Admin accounts | Credential stuffing, session hijack | 2FA, rate limits, secure cookies, short sessions |
| Audit trail | Tampering | Append-only DB role, hash chain (P2) |

## 2. Authentication

- **Better Auth** (email + password with strong policy, optional Google sign-in restricted to the client's domain), `twoFactor` plugin (TOTP) **required** for `owner`/`admin`, optional for others.
- Sessions: HttpOnly, Secure, SameSite=Lax cookies. Idle timeout 12 h, absolute 7 d (configurable). Session revocation on role change and password reset.
- Rate limiting on auth routes (Better Auth built-in) plus WAF rules. Lockout/backoff after repeated failures.
- No self-sign-up: users are invited by owner/admin.
- API keys (registration intake only): random 32 bytes, shown once, stored as SHA-256, scoped (`intake:registrations`), revocable, `last_used_at` tracked.

## 3. RBAC

Roles are hierarchical: `owner > admin > operator > viewer`. `trainer` is a scoped role.

| Permission | owner | admin | operator | viewer | trainer |
|---|---|---|---|---|---|
| `users.manage` | ✅ | ✅ (not owner/admin roles) | | | |
| `integrations.manage` (connect, secrets, triggers) | ✅ | ✅ | | | |
| `workshops.read` | ✅ | ✅ | ✅ | ✅ | own sessions |
| `workshops.write` | ✅ | ✅ | | | |
| `workshops.publish` / cancel / archive | ✅ | ✅ | | | |
| `timeline.write` | ✅ | ✅ | | | |
| `timeline.reschedule` | ✅ | ✅ | ✅ (configurable) | | |
| `automation.write` / `automation.publish` | ✅ | ✅ | | | |
| `automation.control` (start/pause/resume/stop) | ✅ | ✅ | ✅ | | |
| `automation.emergency` | ✅ | ✅ | ✅ | | |
| Global emergency stop | ✅ | | | | |
| `executions.read` | ✅ | ✅ | ✅ | ✅ | |
| `executions.retry` / `execute_now` | ✅ | ✅ | ✅ | | |
| `executions.override` / `resolve` | ✅ | ✅ | ✅ (configurable) | | |
| `participants.read` | ✅ | ✅ | ✅ | masked PII | own sessions, masked |
| `participants.write` | ✅ | ✅ | ✅ | | |
| `participants.export` | ✅ | ✅ | | | |
| `sessions.host` (fresh Zoom start URL) | ✅ | ✅ | | | own sessions |
| Manual task confirm | ✅ | ✅ | ✅ | | own sessions |
| `audit.read` | ✅ | ✅ | read own actions | | |

Implementation: a `@RequirePermission('timeline.reschedule')` decorator + `RbacGuard`. Resource-scoped checks (trainer → own sessions) are done in services via a `PolicyService.can(user, action, resource)`. Permission matrix unit-tested exhaustively.

## 4. Secrets and credential storage

| Secret | Storage | Rotation |
|---|---|---|
| App secrets (DB URL, Better Auth secret, Inngest signing/event keys, R2 keys, Sentry DSN, KMS key ref) | Cloud secrets manager → env at container start | Per provider. Inngest supports `signingKeyFallback` for rotation **[VERIFIED option on client]** |
| Integration credentials | `integration_credentials` table, **envelope-encrypted** (AES-256-GCM data key per record, wrapped by KMS KEK) | Re-wrap on KEK rotation (background job). Provider secrets rotated via UI with dual-validity windows |
| Webhook HMAC secrets (ours, for comms/Apps Script) | Same as integration credentials | Dual active secrets for 7 days during rotation |
| Zoom webhook secret token | Same | Rotate in Zoom app + UI simultaneously (dual verification window) |

Rules: never log secrets (pino redaction paths + tests). Never return secrets via API (`hasSecret: true`). Decrypt only in adapters.

## 5. Webhook security

See [reliability-patterns.md §4](reliability-patterns.md#4-webhook-authentication-and-replay-protection). Additionally:

- Raw-body verification only. Constant-time comparisons.
- Per-integration URLs (`/webhooks/zoom/{integrationId}`), so a leaked secret affects one integration.
- WAF rate limit per path. Payload size limit 2 MB.
- Signature failures → 401 + security event metric. Never process the payload.

## 6. Data protection

- **In transit:** TLS 1.2+ everywhere, HSTS. Internal DB connections over TLS.
- **At rest:** managed Postgres storage encryption. R2 server-side encryption. Column-level encryption for credentials, per-registrant Zoom join URLs and meeting passcodes.
- **Minimization:**
  - Events to Inngest carry IDs only. Step outputs return IDs/statuses (no PII), so data stored in Inngest Cloud stays non-identifying. If PII must pass through a step, use Inngest encryption middleware **[ASSUMPTION: available for v4; verify]**.
  - Comms payloads include only schema-required variables.
  - Logs and attempt summaries are masked.
- **Data residency:** primary region near the client (e.g. India) **[CONFIRM Q16]**. Inngest Cloud processes metadata outside our infra, so the client's DPO should review **[CONFIRM Q11]**.
- **Regulatory:** India's Digital Personal Data Protection Act 2023 likely applies to Indian participants (consent, purpose limitation, erasure on request, breach notification) **[CONFIRM with client's legal; not legal advice]**. The design supports consent flags, erasure (anonymization) and audit.
- **Retention:** see [data-flow.md §7](../architecture/data-flow.md#7-data-retention-proposed-confirm-q15).

## 7. Application security controls

| Control | Implementation |
|---|---|
| Input validation | Zod on every API boundary, webhook payload and jsonb config |
| SQL injection | Drizzle parameterized queries only. No raw SQL from user input. Workflow `query` nodes are predefined |
| SSRF | Comms base URLs validated and allow-listed. Block private IP ranges by default. No user-provided URLs fetched server-side except allow-listed providers |
| XSS | React escaping. CSP (`default-src 'self'`, no inline scripts). Sanitize any rendered rich text |
| CSRF | SameSite cookies + Better Auth origin checks + `trustedOrigins`. Mutations require JSON content type |
| Clickjacking | `frame-ancestors 'none'` |
| File uploads | Presigned PUT with content-length/type constraints. Post-upload verification. Downloads via short-lived presigned GET |
| Dependency security | Renovate + `pnpm audit` / OSV scanner in CI. Lockfile pinned |
| Container | Distroless/slim base, non-root user, read-only FS where possible |
| Headers | `helmet` defaults + custom CSP |

## 8. Audit

- Every administrative mutation and every manual control writes `audit_logs` (actor, action, entity, before/after, reason, IP, UA, correlation ID).
- System actions on behalf of automation are attributed to `system:inngest` with the causing user (if any) in `causation`.
- Append-only enforcement (DB grants). Hash chain (P2). Retention 7 years **[CONFIRM]**.
- Security events (login failures, 2FA changes, role changes, secret rotations, signature failures) are also audited.

## 9. Testing

- Permission matrix tests (every endpoint × role).
- Webhook signature tests (valid, invalid, replayed, stale timestamp).
- Secret redaction tests (log snapshot scanning for known secret patterns).
- Dependency/container scanning in CI. OWASP ZAP baseline scan on staging (P2).
- Pen test before go-live **[recommended]**.

## 10. Acceptance criteria

1. No endpoint is reachable without auth except `/api/auth/*`, `/webhooks/*` (signed) and `/api/inngest` (Inngest-signed), plus `/healthz`.
2. Credentials can't be read back through any API or log.
3. A viewer can't trigger any mutation (tested).
4. Inngest Cloud receives no participant PII in event payloads or step outputs (tested by a payload scanner in integration tests).
