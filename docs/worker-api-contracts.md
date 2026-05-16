# Worker API Contracts (v1)

This document defines request/response JSON contracts and auth requirements for core Worker endpoints.

## Conventions

- Base path: `/api/v1`
- Content type: `application/json`
- Auth header: `Authorization: Bearer <Auth0 access token>` when required
- Standard error envelope:

```json
{
  "error": {
    "code": "string",
    "message": "string",
    "requestId": "string"
  }
}
```

---

## 1) Requests API

### `POST /api/v1/requests`

Create a new client request.

- Auth: required (`client` role)

Request body:

```json
{
  "title": "Need onboarding support",
  "category": "onboarding",
  "description": "Detailed request description",
  "attachments": [
    {
      "objectKey": "uploads/abc123.pdf",
      "fileName": "brief.pdf"
    }
  ]
}
```

Success response `201`:

```json
{
  "requestId": "req_01J...",
  "status": "submitted",
  "createdAt": "2026-05-16T12:00:00Z"
}
```

### `GET /api/v1/requests`

List request history for authenticated client.

- Auth: required (`client` role)

Query params:

- `cursor` (optional)
- `limit` (optional, default 20)

Success response `200`:

```json
{
  "items": [
    {
      "requestId": "req_01J...",
      "title": "Need onboarding support",
      "status": "in_review",
      "updatedAt": "2026-05-16T13:00:00Z"
    }
  ],
  "nextCursor": "opaque_cursor_or_null"
}
```

### `GET /api/v1/requests/{requestId}`

Get detail for a single request.

- Auth: required (`client` role, owner-scoped)

Success response `200`:

```json
{
  "requestId": "req_01J...",
  "title": "Need onboarding support",
  "description": "Detailed request description",
  "status": "in_review",
  "events": [
    {
      "type": "status_changed",
      "at": "2026-05-16T13:10:00Z"
    }
  ]
}
```

---

## 2) Chat API

### `POST /api/v1/chat/session`

Create or resume a chat session tied to a request.

- Auth: required (`client` role)

Request body:

```json
{
  "requestId": "req_01J..."
}
```

Success response `200`:

```json
{
  "sessionId": "chat_01J...",
  "state": "active"
}
```

### `POST /api/v1/chat/message`

Append a message to an active chat session.

- Auth: required (`client` role)

Request body:

```json
{
  "sessionId": "chat_01J...",
  "message": "Any update on my request?"
}
```

Success response `200`:

```json
{
  "messageId": "msg_01J...",
  "acceptedAt": "2026-05-16T14:00:00Z"
}
```

### `GET /api/v1/chat/session/{sessionId}`

Retrieve chat session transcript.

- Auth: required (`client` role, participant-scoped)

Success response `200`:

```json
{
  "sessionId": "chat_01J...",
  "messages": [
    {
      "messageId": "msg_01J...",
      "sender": "client",
      "text": "Any update on my request?",
      "timestamp": "2026-05-16T14:00:00Z"
    }
  ]
}
```

---

## 3) Partner Upload API

### `POST /api/v1/partner/upload`

Initiate partner upload job and receive ingest instructions.

- Auth: required (`partner` or `developer/admin` role)

Request body:

```json
{
  "fileName": "partner-batch-2026-05-16.csv",
  "contentType": "text/csv",
  "sizeBytes": 202400
}
```

Success response `201`:

```json
{
  "jobId": "job_01J...",
  "upload": {
    "method": "PUT",
    "url": "https://signed-upload-url",
    "headers": {
      "content-type": "text/csv"
    },
    "expiresAt": "2026-05-16T14:15:00Z"
  }
}
```

### `POST /api/v1/partner/upload/{jobId}/complete`

Mark upload complete and queue validation/ingestion.

- Auth: required (`partner` or `developer/admin` role)

Request body:

```json
{
  "objectKey": "partner-ingest/job_01J.../partner-batch-2026-05-16.csv",
  "checksum": "sha256:..."
}
```

Success response `202`:

```json
{
  "jobId": "job_01J...",
  "status": "queued"
}
```

### `GET /api/v1/partner/upload/{jobId}`

Fetch upload job status and errors.

- Auth: required (`partner` or `developer/admin` role, tenant-scoped)

Success response `200`:

```json
{
  "jobId": "job_01J...",
  "status": "failed",
  "errors": [
    {
      "row": 42,
      "code": "INVALID_EMAIL",
      "message": "email is not valid"
    }
  ]
}
```

---

## 4) Auth Requirements Summary

- Public routes/pages may exist, but all data-bearing API endpoints above require valid Auth0 bearer tokens.
- Worker must enforce:
  - token signature/issuer/audience/time checks,
  - role/scope checks,
  - tenant ownership constraints,
  - audit logging for privileged partner/developer operations.

