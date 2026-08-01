---
name: Verify and handle SafetyKit webhooks
description: Receive SafetyKit workflow webhooks, verify their HMAC-SHA256 signature, and process agent decisions idempotently.
api: SafetyKit Data API
source: https://docs.safetykit.com/webhooks/verifying-signatures
operations:
  - addEndpoint
  - verifySignature
---

# Verify and handle SafetyKit webhooks

SafetyKit delivers asynchronous agent decisions as outbound webhooks following the Standard Webhooks (Svix) convention.

## Steps
1. **Register an endpoint** in the SafetyKit dashboard (Webhooks → Adding Endpoints) and store the signing secret (`whsec_...`).
2. **Verify every delivery before trusting it.** Read three headers: `webhook-id`, `webhook-timestamp`, `webhook-signature`.
   - Reject if `webhook-timestamp` is more than 300 seconds from now (replay guard).
   - Compute `HMAC-SHA256` over `"{webhook-id}.{webhook-timestamp}.{raw-body}"` using the base64-decoded portion of the secret after `whsec_`.
   - Compare against each `v1,<sig>` entry in `webhook-signature` using a **constant-time** comparison. The raw, unmodified request body must be used.
3. **Process idempotently.** `webhook-id` is stable across resends — dedupe on it.
4. **Handle event types:**
   - `workflow.succeeded` — read `output.actions`, `output.labels`, `output.fields`, `output.label_changes`; correlate via `namespace` + `id` + `request_id`.
   - `workflow.failed` — mark the object for retry or manual review.

## Rules
- Return 2xx quickly; do heavy work off the request path.
- Never skip signature verification, even in test.
