# PRIMA Clock Integration API Documentation

```
https://clock.madebyflow.de/api/v1/integration
```

All endpoints in this document are relative to this base URL.

---

##  Authentication

All API requests must include the following HTTP header:

```
X-API-KEY: <api-key>
```

API keys have permissions. Endpoints require either:

- `PERM_READ`
- `PERM_WRITE`

---

## Error Object Format

<details>
<summary><strong>Show error object structure</strong></summary>

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

**Notes**
- `errors[]` is typically present for validation errors (HTTP 400)
- `code` values depend on the server-side error mapping

</details>

---

# Work Time Endpoints

---

<details>
<summary><strong>GET /work-times</strong></summary>

Exports finished work time entries in a paginated format.

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

- Both parameters are optional
- Dates must be provided in ISO-8601 format (`YYYY-MM-DD`)
- Filtering is applied on the creation timestamp (`created`)
- The `created` timestamp is evaluated in UTC
- The `timezone` parameter does not affect filtering

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
        },
	{
          "projectId": 20,
          "projectName": "SalesViewer"
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

- **400 — Validation Error**  
  Invalid parameters (e.g. invalid timezone)

</details>

---

#  Customer Import Endpoints

The import is a **session-based workflow** identified by `importId`.

### Workflow
1. **INIT** — announce import size
2. **PAGE** — send pages
3. **FINALIZE** — apply import
4. **STATUS** — optional poll status
5. **CANCEL** — optional abort import

---

<details>
<summary><strong>POST /customers/import/init</strong></summary>

Initializes a new customer import session.

Required permission: `PERM_WRITE`

---

### Request Body

```json
{
  "importId": "import-2026-01-26T120000Z",
  "totalPages": 15,
  "totalCustomers": 14400,
  "pageSize": 1000
}
```

---

### Fields

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `importId` | string | yes | Unique identifier of the import session |
| `totalPages` | integer | yes | Total number of pages (≥ 1) |
| `totalCustomers` | integer | yes | Expected total customers (≥ 0) |
| `pageSize` | integer | yes | Page size used (≥ 1) |

---

### Success Response
- **201 Created** — empty body

### Error Responses
- **400** Validation error
- **409** Import already active

</details>

---

<details>
<summary><strong>POST /customers/import/page</strong></summary>

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

---

### Behavior

- Pages may arrive out of order
- Duplicate page submissions are accepted (last wins)
- Status switches to `READY` when all pages are received

---

### Success Response
- **202 Accepted**

### Error Responses
- **400** Validation error
- **409** Import not accepting pages

</details>

---

<details>
<summary><strong>POST /customers/import/finalize</strong></summary>

Finalizes the import and applies changes.

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

- Import exists and status is `READY`
- `receivedPages == totalPages`
- Staged customer count equals `totalCustomers`
- Staging data exists

---

### Success Response
- **202 Accepted**

### Error Responses
- **400** Validation error
- **409** Import not ready or validation failed

</details>

---

<details>
<summary><strong>POST /customers/import/cancel</strong></summary>

Cancels an import session and deletes staging data.

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

- Cancel allowed in states:
  - RECEIVING
  - READY
  - FAILED
- Staging data is deleted

---

### Success Response
- **204 No Content**

### Error Responses
- **400** Validation error
- **409** Import cannot be cancelled

</details>

---

<details>
<summary><strong>GET /customers/import/status</strong></summary>

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
- **404** Import does not exist or has expired

</details>
