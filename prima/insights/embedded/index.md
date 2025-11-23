---
title: ""
---

# Embedded API Documentation

##  Base URL


```
https://insights.madebyflow.de/api/v1/embedded
```

All endpoints in this document are relative to this base URL.

## API Error Object Structure

All error responses follow this standardized format:

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

## Authentication

All requests must include the API key header:

```
INSIGHTS-API-KEY: <api-key>
```

Embedded interface/API calls require an additional bearer token (issued via token endpoint):

```
Authorization: Bearer <embedded-token>
```


#  Endpoints


## Endpoint: /companies/{ergoId}/token

Creates a short-lived token granting access to the embedded **Company Insights** interface for the company identified by its `ergoId`.

**HTTP Request**
```http
POST /companies/{ergoId}/token
```


###  Request Body (optional)

```json
{
  "from": "2025-01-01",
  "to":   "2025-12-31"
}
```

**Notes**
- `from` and `to` are optional  
- Date format: `yyyy-MM-dd`  
- If omitted, the **last 7 days** are used  


###  Response (201 Created)

```json
{
  "embeddedUrl": "https://insights.madebyflow.de/embedded/companies/932?token=<bearer-token>",
  "bearerAuthorizationToken": "eyJhbGciOi...",
  "maxUsage": 10,
  "expires": "2025-01-20T10:15:30Z"
}
```

###  Field Description

| Field | Type | Explanation |
|-------|------|-------------|
| **embeddedUrl** | String | URL that loads the embedded Insights interface (e.g., in an iFrame). |
| **bearerAuthorizationToken** | String | Short-lived bearer token required when calling the embedded interface or API endpoints. |
| **maxUsage** | Integer | Maximum number of allowed API calls using this token. |
| **expires** | Instant (UTC) | Timestamp when the token becomes invalid. |

### Error Responses

#### 404 – Company Not Found
No company exists for the given `ergoId`.

#### 404 – No Reports Available
The company exists, but no reports match the requested date range.

#### 400 – Invalid Date Range
The provided dates are invalid (e.g. unsupported range or future dates).
