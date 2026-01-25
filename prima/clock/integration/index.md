# PRIMA clock integration API Documentation

```
https://clock.madebyflow.de/api/v1/integration
```

All endpoints in this document are relative to this base URL.

---

## Authentication

All API requests must include the following HTTP header:

```
X-API-KEY: <api-key>
```
---

## Error Object Format

Every error response uses this structure:

```json
{
  "code": "ERROR_CODE",
  "message": "Human readable error message",
  "timestamp": "2026-01-25T10:15:30Z",
  "errors": [
    {
      "field": "fieldName",
      "message": "Validation message"
    }
  ]
}
```

---

# Endpoints

## GET /work-times

Exports finished work time entries in a paginated format.

This endpoint is intended for **system integrations** and is accessible **only via API key authentication**.

The API key must have the **READ** permission to access this endpoint.

---

## Query Parameters

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `page` | integer | no | `0` | Zero-based page index |
| `size` | integer | no | `100` | Number of items per page |
| `timezone` | string | no | `Europe/Rome` | IANA timezone used to convert timestamps |
| `from` | date (`YYYY-MM-DD`) | no | — | Start date (inclusive) |
| `to` | date (`YYYY-MM-DD`) | no | — | End date (inclusive) |

## Date Filter Logic (`from` / `to`)

- Both parameters are **optional**
- Dates must be provided in **ISO-8601 format**: `YYYY-MM-DD`
- **Filtering is applied on the creation timestamp** (`created`)
- The `created` timestamp is evaluated in **UTC**
- The `timezone` parameter **does not affect filtering**, only timestamp conversion in the response

### Timezone Notes
- If omitted, `Europe/Rome` is used.
- If invalid, the request fails with a validation error.

---

## Sorting

Results are sorted by:
1. `created` (descending)
2. `id` (descending)

Newest entries are returned first.

---

## Response (200 OK)

```json
{
  "result": [
    {
      "companyId": "CUST-INT-001",
      "employeeId": "EMP-INT-042",
      "docType": 1,
      "serialYear": 2026,
      "docId": 4711,
      "taskId": 987654,
      "notes": "Implementation work",
      "startTask": "2026-01-25T08:30:00+01:00",
      "endTask": "2026-01-25T12:15:00+01:00",
      "created": "2026-01-25T12:15:10Z",
      "updated": "2026-01-25T12:16:02Z",
      "projects": [
        {
          "projectId": 42,
          "projectName": "Website Relaunch"
        }
      ]
    },
    {
      "companyId": "CUST-INT-002",
      "employeeId": "EMP-INT-042",
      "docType": null,
      "serialYear": null,
      "docId": null,
      "taskId": 987655,
      "notes": null,
      "startTask": "2026-01-26T09:00:00+01:00",
      "endTask": "2026-01-26T10:30:00+01:00",
      "created": "2026-01-26T10:30:05Z",
      "updated": "2026-01-26T10:30:05Z",
      "projects": []
    }
  ],
  "pageMeta": {
    "page": 0,
    "size": 100,
    "totalElements": 2,
    "totalPages": 1
  }
}
```

---

## Error Responses

### 400 — Validation Error
Invalid parameters (e.g. invalid timezone).

### 401 — Unauthorized
Missing or invalid API key.

### 403 — Forbidden
API key does not have READ permission.


