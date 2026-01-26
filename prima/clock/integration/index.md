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

API keys have permissions. Endpoints require either:

- `PERM_READ`
- `PERM_WRITE`

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

Notes:
- `errors[]` is typically present for validation errors (HTTP 400).
- `code` values depend on the server-side error mapping (examples in this doc use descriptive names).

---

# Endpoints

## GET /work-times

Exports finished work time entries in a paginated format.

This endpoint is intended for **system integrations** and is accessible **only via API key authentication**.

The API key must have the **READ** permission to access this endpoint.

Required permission: `PERM_READ`

---

### Query Parameters

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `page` | integer | no | `0` | Zero-based page index |
| `size` | integer | no | `100` | Number of items per page |
| `timezone` | string | no | `Europe/Rome` | IANA timezone used to convert timestamps |
| `from` | date (`YYYY-MM-DD`) | no | — | Start date (inclusive) |
| `to` | date (`YYYY-MM-DD`) | no | — | End date (inclusive) |

---

### Date Filter Logic (`from` / `to`)

- Both parameters are **optional**
- Dates must be provided in **ISO-8601 format**: `YYYY-MM-DD`
- **Filtering is applied on the creation timestamp** (`created`)
- The `created` timestamp is evaluated in **UTC**
- The `timezone` parameter **does not affect filtering**, only timestamp conversion in the response

#### Timezone Notes
- If omitted, `Europe/Rome` is used.
- If invalid, the request fails with a validation error.

---

### Sorting

Results are sorted by:
1. `created` (descending)
2. `id` (descending)

Newest entries are returned first.

---

### Response (200 OK)

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

### Error Responses

#### 400 — Validation Error
Invalid parameters (e.g. invalid timezone).


---

# Customer Import API

The import is a **session-based** workflow identified by `importId`:

1. **INIT** — announce import size (pages + total customers)
2. **PAGE** — send pages (0-based page index)
3. **STATUS** — poll the status at any time
4. **FINALIZE** — apply the import to the main `customers` table
5. **CANCEL** — abort an import and delete staging data

---


## POST /customers/import/init

Initializes a new customer import session.

Required permission: `PERM_WRITE`

---

### Request Body

```json
{
  "importId": "import-2026-01-26T120000Z",
  "totalPages": 15,
  "totalCustomers": 14400,
  "pageSize": 1000,
}
```

### Fields

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `importId` | string | yes | Unique identifier of the import session |
| `totalPages` | integer | yes | Total number of pages that will be sent (must be ≥ 1) |
| `totalCustomers` | integer | yes | Expected total number of customers (must be ≥ 0) |
| `pageSize` | integer | yes | page size that will be used (must be ≥ 1) |

### Notes

- The import is valid for a limited time (TTL) and may expire if not completed.

---

### Success Response (201 Created)

Empty body.

---

### Error Responses

#### 400 — Validation Error
Invalid JSON or invalid values (e.g. `totalPages < 1`).

#### 409 — Conflict Error
Import already active


---

## POST /customers/import/page

Uploads one page of customer data.

Required permission: `PERM_WRITE`

---

### Request Body

```json
{
  "importId": "import-2026-01-26T120000Z",
  "page": 0,
  "customers": [
    {
      "internalId": "CUST-001",
      "name": "Hotel Aurora",
      "place": "Luttach"
    },
    {
      "internalId": "CUST-002",
      "name": "Hotel Alpine"
    }
  ]
}
```

### Fields

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `importId` | string | yes | Import session identifier |
| `page` | integer | yes | Zero-based page index (must be ≥ 0) |
| `customers` | array | yes | Customers in this page (must not be empty) |

### Customer object

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `internalId` | string | yes | Unique customer identifier |
| `name` | string | yes | Customer name |
| `place` | string | no | the place where the customer is located|

---

### Behavior

- Pages may arrive out of order.
- Duplicate page submissions are accepted; Last submission is used.
- When `receivedPages == totalPages`, the server sets the import status to `READY`.

---

### Success Response (202 Accepted)

Empty body.

---

### Error Responses

#### 400 — Validation Error
Invalid JSON, missing fields, empty customers list, invalid page value.


#### 409 — Conflict
Import not initialized, cancelled, done, or otherwise not accepting pages.

---

## POST /customers/import/finalize

Finalizes the import and applies the changes to the `customers` table.

Required permission: `PERM_WRITE`

---

### Request Body

```json
{
  "importId": "import-2026-01-26T120000Z"
}
```

---

### Finalize Validation

Finalize will only run if:

- The import exists and is in status `READY`
- `receivedPages == totalPages`
- The number of staged customer records equals `totalCustomers` (expected count from init)
- Staging data exists

If any validation fails, the request returns a conflict error.

---

### Success Response (202 Accepted)

Empty body.

---

### Error Responses

#### 400 — Validation Error
Invalid JSON or missing/empty `importId`.


#### 409 — Conflict
Import not ready, incomplete, cancelled, or validation failed.

---

## POST /customers/import/cancel

Cancels an import session and deletes staging data for this import.

Required permission: `PERM_WRITE`

---

### Request Body

```json
{
  "importId": "import-2026-01-26T120000Z"
}
```

---

### Behavior

- Cancelling is allowed only in states: `RECEIVING`, `READY`, `FAILED`
- Staging records are deleted

---

### Success Response (204 No Content)

Empty body.

---

### Error Responses

#### 400 — Validation Error
Invalid JSON or missing/empty `importId`.


#### 409 — Conflict
Import cannot be cancelled (e.g. already `PROCESSING` or `DONE`).

---

## GET /customers/import/status

Returns the current status of an import.

Required permission: `PERM_READ`

---

### Query Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `importId` | string | yes | Import session identifier |

---

### Response (200 OK)

```json
{
  "importId": "import-2026-01-26T120000Z",
  "status": "RECEIVING",
  "totalPages": 14,
  "receivedPages": 7,
  "totalCustomers": 14000
}
```

---

### Error Responses

#### 404 — Not Found
Import does not exist or has expired.


---
