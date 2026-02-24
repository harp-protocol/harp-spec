# HARP Gateway HTTP Binding (REST + SSE + WebSocket) — Draft v0.2

**Status:** Draft  
**Date:** 2026-02-24  
**Applies to:** HARP-GW v0.2

This document defines the HTTP wire protocol binding for the HARP Gateway. It is **Envelope-mandatory**: every message body sent to, or emitted by, the Gateway MUST be a HARP Envelope.

## 1. Media Type and Encoding

- `Content-Type: application/harp+json`
- UTF-8 encoded JSON.
- All timestamps MUST be RFC3339.
- All IDs MUST be case-sensitive strings unless otherwise specified.

## 2. Envelope (wire requirement)

All HTTP request bodies, SSE `data:` payloads, and WebSocket messages MUST be:
- an **Envelope** object
- with `msgType` set per message types below

See `schemas/harp-gateway-envelope.schema.json` for the wire schema of the envelope + gateway-required fields.

## 3. REST Endpoints

### 3.1 Submit Artifact (Enforcer → Gateway)
`POST /v1/artifacts`

**Request body (Envelope):**
- `msgType`: `artifact.submit`
- `requestId`: REQUIRED
- `sender.enforcerId`: REQUIRED
- `body`: MUST conform to `schemas/harp-gateway-artifact-submit.schema.json`

**Responses:**
- `202 Accepted` with an Envelope `artifact.accepted`
- `409 Conflict` for AlreadyExistsConflict
- `400 Bad Request` for validation errors

### 3.2 Submit Decision (Approver → Gateway)
`POST /v1/decisions`

**Request body (Envelope):**
- `msgType`: `decision.submit`
- `requestId`: REQUIRED
- `sender.approverId`: REQUIRED
- `body`: MUST conform to `schemas/harp-gateway-decision-submit.schema.json`

**Responses:**
- `200 OK` with Envelope `decision.accepted`
- `409 Conflict` for AlreadyDecidedConflict
- `400 Bad Request` for validation errors

### 3.3 List Approver Inbox (manual refresh)
`GET /v1/approvers/{approverId}/inbox?cursor=&limit=`

**Response body (Envelope):**
- `msgType`: `inbox.page`
- `body`: MUST conform to `schemas/harp-gateway-inbox-page.schema.json`

### 3.4 Exchange Status
`GET /v1/exchanges/{requestId}`

**Response body (Envelope):**
- `msgType`: `exchange.status`
- `body`: MUST conform to `schemas/harp-gateway-exchange-status.schema.json`

### 3.5 Enforcer Wait (long-poll fallback)
`GET /v1/exchanges/{requestId}/wait?timeout=30`

- `timeout` is seconds (1..60 RECOMMENDED).
- If a decision is available within timeout, returns `200 OK` with `decision.deliver` envelope.
- Otherwise returns `204 No Content`.

## 4. SSE Endpoints

### 4.1 Approver SSE stream
`GET /v1/sse/approvers/{approverId}`

The server emits events where `data:` is an Envelope.

**Event types (SSE `event:` field):**
- `approval.request` — Envelope msgType `approval.request`
- `ping` — keepalive (OPTIONAL; data MAY be empty)

At minimum, the Gateway MUST ensure all pending approval requests are retrievable via the inbox listing API, regardless of SSE connectivity.

### 4.2 Enforcer SSE stream (OPTIONAL)
`GET /v1/sse/enforcers/{enforcerId}`

Events:
- `decision.deliver` — Envelope msgType `decision.deliver`

## 5. WebSocket Endpoint

`GET /v1/ws?role={enforcer|approver}&id={enforcerId|approverId}`

All WS frames MUST be JSON Envelopes.

**Gateway emitted msgTypes:**
- `approval.request`
- `decision.deliver`
- `error`

**Client emitted msgTypes:**
- `artifact.submit` (OPTIONAL if REST is used for submit)
- `decision.submit` (OPTIONAL if REST is used for submit)
- `ack.submit`

## 6. Ack Endpoint / Message Ack

### 6.1 REST Ack
`POST /v1/acks`

**Request body (Envelope):**
- `msgType`: `ack.submit`
- `body`: MUST conform to `schemas/harp-gateway-ack-submit.schema.json`

**Response:**
- `200 OK` with `ack.accepted`

## 7. Deterministic routing rule (default)

When multiple active channels exist for the same Enforcer identity, the Gateway MUST deliver `decision.deliver` to the **most recent active channel**. If no channel is active, the decision MUST be retained for inbox/poll retrieval until expiration/retention.

## 8. Minimal error envelope

On error, the Gateway MUST return an Envelope:
- `msgType`: `error`
- `body`: conforming to `schemas/harp-gateway-error.schema.json`
