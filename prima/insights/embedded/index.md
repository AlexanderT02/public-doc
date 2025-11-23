# Embedded API Documentation

##  Base URL


```
https://insights.madebyflow.de/api/v1/embedded
```

All endpoints in this document are relative to this base URL.


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


##  POST /companies/{ergoId}/token


Generates a short-lived token that grants access to the embedded Company Insights interface for the company identified by their ergoId.


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
