# ADR-010: Server-Sent Events, fanned out with Postgres LISTEN/NOTIFY

- Status: Accepted
- Date: 2026-11-03

## Context

The Workshop Control Center and the Execution Monitor need live updates (activity status changes, attendance counters, alerts). Traffic is server→client only, and there are few concurrent viewers (fewer than 50).

## Decision

- Use **SSE** via NestJS `@Sse()` endpoints (`GET /api/v1/stream?topics=workshop:{id},alerts`). Events carry **IDs + status deltas only**. The client then refetches details with TanStack Query (cache invalidation).
- **Cross-instance fan-out**: domain modules publish change notifications with `pg_notify('wos_events', json)` **after commit**. Each `api` instance holds one dedicated `LISTEN` connection and forwards notifications to subscribed SSE clients. Postgres NOTIFY payloads are limited (8000 bytes by default), so only IDs are sent.
- Reconnect: SSE `Last-Event-ID` with a short in-memory replay buffer. On gaps, the client does a full refetch (correctness never depends on the stream).
- Proxies must disable response buffering for `text/event-stream`, and the deployment must support HTTP/2 to avoid the 6-connections-per-origin HTTP/1.1 limit.

## Alternatives

- **WebSockets** (`@nestjs/websockets`): bidirectional, which we don't need. More complex auth and proxies.
- **Redis pub/sub**: adds Redis for no other reason.
- **Polling**: acceptable fallback. The client falls back to 10 s polling if SSE fails.

## Consequences

- ✅ No new infrastructure.
- ⚠️ A `LISTEN` connection is not pooled through PgBouncer in transaction mode. It must use a direct connection.
