# ADR-012: No automatic retry on unknown outcome for non-idempotent providers

- Status: Accepted
- Date: 2026-11-03

## Context

Retrying is safe only if the target operation is idempotent. When we trigger the client's **existing WhatsApp/email automation**, the request may time out *after* the remote system has accepted it. Retrying would then send a **duplicate message to a real participant**. Whether the existing system honors an idempotency key is unknown **[CONFIRM Q1, Q2]**.

## Decision

Every action in the registry declares an **idempotency class**:

| Class | Meaning | Retry on timeout / connection reset after send |
|---|---|---|
| `natural` | Operation is idempotent by nature (GET, PUT-to-state, Calendar insert with our deterministic event ID) | ✅ auto-retry |
| `keyed` | Provider honors our `Idempotency-Key` (verified in contract tests) | ✅ auto-retry with the same key |
| `checkable` | Provider offers a lookup to check whether the previous attempt landed (e.g. list Zoom meetings by our `agenda` tag) | ✅ check first, then retry if absent |
| `unsafe` | None of the above | ❌ **No auto-retry**. Mark `needs_attention` with reason `unknown_outcome` |

- Errors **before** the request is sent (DNS failure, connection refused, 429 with `Retry-After`, explicit 5xx with a "not processed" body) are retried for every class.
- The admin sees "Outcome unknown: check the WhatsApp system, then choose **Mark as delivered** or **Resend**". Both choices are audited.
- The comms trigger contract ([whatsapp-email-integration.md](../features/whatsapp-email-integration.md#4-outbound-trigger-contract)) **asks** the client's systems to honor `Idempotency-Key`. Once verified, those triggers move to `keyed`.

## Consequences

- ✅ Avoids the worst real-world failure: duplicate messages to hundreds of participants after a network blip.
- ⚠️ More manual work during comms-system outages. This is mitigated by bulk "resolve unknown outcomes" actions in the Execution Monitor.
