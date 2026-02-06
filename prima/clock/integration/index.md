# PRIMA Clock Integration API Documentation
![API Version](https://img.shields.io/badge/API-v1--stable-success)
![Last Updated](https://img.shields.io/badge/Last%20Updated-2026--01--26-blue)


```
https://clock.madebyflow.de/api/v1/integration
```

All endpoints in this document are relative to this base URL.

# Table of Contents

- [Authentication](#authentication)
- [Error Object Format](#error-object-format)
- [Work Time Endpoints](#work-time-endpoints)
  - [GET /work-times](#get-work-times)
- [Customer Import Endpoints](#customer-import-endpoints)
  - [POST /customers/import/init](#post-customersimportinit)
  - [POST /customers/import/page](#post-customersimportpage)
  - [POST /customers/import/finalize](#post-customersimportfinalize)
  - [POST /customers/import/cancel](#post-customersimportcancel)
  - [GET /customers/import/status](#get-customersimportstatus)


##  Authentication

All API requests must include the following HTTP header:

```
X-API-KEY: <api-key>
```

API keys have permissions. Endpoints require either:

- `PERM_READ`
- `PERM_WRITE`


## Error Object Format



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



# Work Time Endpoints


## GET /work-times

Exports finished work time entries in a paginated format.

Required permission: `PERM_READ`


### Query Parameters

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `page` | integer | no | `0` | Zero-based page index |
| `size` | integer | no | `100` | Number of items per page |
| `timezone` | string | no | `Europe/Rome` | IANA timezone used to convert timestamps |
| `from` | date (`YYYY-MM-DD`) | no | — | Start date (inclusive) |
| `to` | date (`YYYY-MM-DD`) | no | — | End date (inclusive) |


### Date Filter Logic (`from` / `to`)

- Both parameters are optional
- Dates must be provided in ISO-8601 format (`YYYY-MM-DD`)
- Filtering is applied on the creation timestamp (`created`)
- The `created` timestamp is evaluated in UTC
- The `timezone` parameter does not affect filtering


### Sorting

Results are sorted by:
1. `created` (descending)
2. `id` (descending)

Newest entries are returned first.


### Response Example (200 OK)

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

### Response Model

Represents one finished work time entry returned by the export.

| Field | Type | Nullable | Description |
|----------------|----------------------|-----------|----------------------------------------------|
| companyId | string | no | Customer / company identifier |
| employeeId | string | no | Internal employee identifier |
| docType | integer | yes | ERP document type |
| serialYear | integer | yes | ERP document year |
| docId | integer | yes | ERP document number |
| taskId | integer | no | Unique task or work entry ID |
| notes | string | yes | Optional free text notes |
| startTask | datetime (ISO-8601) | no | Task start timestamp with timezone |
| endTask | datetime (ISO-8601) | no | Task end timestamp with timezone |
| created | datetime (UTC ISO-8601) | no | Creation timestamp in UTC |
| updated | datetime (UTC ISO-8601) | no | Last update timestamp in UTC |
| projects | array<Project> | no | Associated projects (may be empty) |

### PageMeta

Pagination information for the current result set.

| Field | Type | Nullable | Description |
|--------------|-----------|-----------|-------------------------------|
| page | integer | no | Current page index (zero-based) |
| size | integer | no | Page size |
| totalElements | integer | no | Total number of records |
| totalPages | integer | no | Total number of pages |

#### Notes

- All timestamps use ISO-8601 format  
- `created` and `updated` are always UTC  
- `startTask` and `endTask` respect the requested `timezone` parameter  
- `projects` can be an empty array but is never null  



### Error Responses

- **400 — Validation Error**  
  Invalid parameters (e.g. invalid timezone)


# Customer Import Endpoints

The import is a **session-based workflow** identified by `importId`.

### Required Workflow
1. **INIT** — announce import size  
2. **PAGE** — send pages  
3. **FINALIZE** — apply import  

### Optional Endpoints
- **STATUS** — poll import status  
- **CANCEL** — abort import session


## POST /customers/import/init

Initializes a new customer import session.

Required permission: `PERM_WRITE`


### Request Body

```json
{
  "importId": "import-2026-01-26T120000Z",
  "totalPages": 15,
  "totalCustomers": 14400,
  "pageSize": 1000
}
```


| Field | Type | Required | Description |
|------|------|----------|-------------|
| `importId` | string | yes | Unique identifier of the import session |
| `totalPages` | integer | yes | Total number of pages (≥ 1) |
| `totalCustomers` | integer | yes | Expected total customers (≥ 0) |
| `pageSize` | integer | yes | Page size used (≥ 1) |

### Success Response
- **201 Created** — empty body

### Error Responses
- **400** Validation error
- **409** Import already active



## POST /customers/import/page

Uploads one page of customer data.

Required permission: `PERM_WRITE`



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

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `importId` | string | yes | Import session identifier |
| `page` | integer | yes | Zero-based page index (must be ≥ 0) |
| `customers` | array | yes | Customers in this page (must not be empty) |

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `internalId` | string | yes | Unique customer identifier |
| `name` | string | yes | Customer name |
| `place` | string | no | The place where the customer is located |


### Behavior

- Pages may arrive out of order
- Duplicate page submissions are accepted (last wins)
- Status switches to `READY` when all pages are received


### Success Response
- **202 Accepted**

### Error Responses
- **400** Validation error
- **409** Import not accepting pages




## POST /customers/import/finalize

Finalizes the import and applies changes.

Required permission: `PERM_WRITE`



### Request Body

```json
{
  "importId": "import-2026-01-26T120000Z"
}
```

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `importId` | string | yes | Import session identifier |



### Finalize Validation

- Import exists and status is `READY`
- `receivedPages == totalPages`
- Staged customer count equals `totalCustomers`
- Staging data exists



### Success Response
- **202 Accepted**

### Error Responses
- **400** Validation error
- **409** Import not ready or validation failed


## POST /customers/import/cancel

Cancels an import session and deletes staging data.

Required permission: `PERM_WRITE`



### Request Body

```json
{
  "importId": "import-2026-01-26T120000Z"
}
```

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `importId` | string | yes | Import session identifier |

### Behavior

- Cancel allowed in states:
  - RECEIVING
  - READY
  - FAILED
- Staging data is deleted



### Success Response
- **204 No Content**

### Error Responses
- **400** Validation error
- **409** Import cannot be cancelled

## GET /customers/import/status

Returns the current status of an import.

Required permission: `PERM_READ`

### Query Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `importId` | string | yes | Import session identifier |


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


### Error Responses
- **404** Import does not exist or has expired



