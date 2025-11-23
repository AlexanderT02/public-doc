# Embedded API Documentation

```
https://insights.madebyflow.de/api/v1/embedded
```

All endpoints in this document are relative to this base URL.
## Authentication

All API requests must include the following http header:

```
INSIGHTS-API-KEY: <api-key>
```


## Error Object Format

Every error response uses this structure:

```json
{
  "timestamp": "2025-11-23T01:26:57.412456288Z",
  "status": 404,
  "error": "Not Found",
  "message": "No company found for ERGO ID: 700222905",
  "path": "/api/v1/embedded/companies/700222905/token",
  "details": []
}
```




# Endpoints

## POST /companies/{ergoId}/token

Creates a short‑lived embedded token granting access to the **Company Insights** interface for the company identified by `ergoId`.

### Optional Request Body

```json
{
  "from": "2025-01-01",
  "to":   "2025-12-31"
}
```

### Notes
- `from` and `to` are optional  
- Format: `yyyy-MM-dd`  
- If omitted, the API automatically selects the most recent complete weekly period, starting on Monday and ending on Sunday. 


## Response (201 Created)

```json
{
  "embeddedUrl": "https://insights.madebyflow.de/embedded/companies/932?token=<bearer-token>",
  "meta": {
    "companyIds": ["932"],
    "bearerAuthorizationToken": "eyJhbGciOi...",
    "maxUsage": 10,
    "expires": "2025-01-20T10:15:30Z"
  }
}
```

## Response Fields

### embeddedUrl — *String*
**Primary response field.**  
A fully signed (“magic link”) URL used to load the Company Insights interface directly (e.g., inside an iFrame).


### meta — *Object*
Additional metadata describing token access limitations, visibility scope, and validity.

#### meta.companyIds — *Array[String]*
List of company IDs that can be accessed with this token. Defines the visibility scope.

#### meta.bearerAuthorizationToken — *String*
Short-lived bearer token for API requests in the embedded flow.  
Access is limited to the companies listed in `meta.companyIds`.

#### meta.maxUsage — *Integer*
Maximum number of allowed API requests using this token.

#### meta.expires — *Instant (UTC)*
Exact timestamp when the token and embedded URL expire and can no longer be used.


## Error Responses

### 404 — Company Not Found
No company exists for the provided `ergoId`.

### 404 — No Reports Available
Company exists, but no company insights match the requested date range.

### 400 — Invalid Date Range
Provided dates are not allowed (bad format, unsupported range, future dates, etc.).

---
