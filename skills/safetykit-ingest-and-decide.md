---
name: Ingest an object and receive a risk decision
description: Submit an object to a SafetyKit namespace and retrieve the AI agent's labels, risk score, and enforcement actions, either by webhook or by polling.
api: SafetyKit Data API
source: https://docs.safetykit.com/using-data-api/copy-and-paste-quickstart
operations:
  - add
  - getStatus
  - getDownloadUrl
---

# Ingest an object and receive a risk decision

Use this to send a user, listing, or content object to SafetyKit for automated risk review and get back the agent decision.

## Auth
Every call sends `Authorization: Bearer sk_...` (a server-side key from the Team API Keys page). Never expose the key to browsers or mobile clients.

## Steps
1. **Add data** — `POST /v1/data/{namespace}` with a body `{ "data": [ { "id": "<stable-id>", ... } ] }`. Each object MUST carry a stable `id`; resubmitting the same `id` upserts and re-evaluates it (this is the dedup contract — there is no Idempotency-Key header). The call returns a `request_id` immediately; processing is asynchronous.
2. **Receive the result.** Preferred: register a webhook and handle `workflow.succeeded` (payload `output` carries `actions`, `labels`, `fields`, `label_changes`) — verify the `webhook-signature` header first (see the handle-webhooks skill). Alternative: **poll** `GET /v1/data/{namespace}/requests/{requestId}` until `status` is `succeeded` (states: queued → ingesting → succeeded/failed).
3. **Fetch large results** — if a status response would exceed 5 MiB it returns HTTP 413; call `GET /v1/data/{namespace}/requests/{requestId}/results/download-url` and download from the returned URL instead.

## Rules
- Retry `429`, `500`, `503` with exponential backoff.
- Persist both your object `id` and the `request_id` for reconciliation.
- Treat `output` as potentially null and `metadata` as optional when parsing.
