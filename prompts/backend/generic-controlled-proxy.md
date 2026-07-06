# Prompt: Implement Generic Controlled Proxy in .NET Backend

Use this prompt when you want an AI coding assistant to implement a secure generic proxy in the .NET backend for routing frontend requests to the Chatbot Service and AI Workflow Service.

---

## Prompt

You are a senior .NET backend architect and security-focused engineer.

Implement a **generic controlled proxy** in an ASP.NET Core backend.

The product is currently **single tenant and single deal**, so do not over-engineer multi-tenant abstractions yet. However, design the implementation so that tenant/deal scoping can be added later without rewriting the proxy architecture.

## Current architecture

- React frontend calls only the .NET backend.
- React frontend authenticates users with Microsoft Entra ID using Authorization Code Flow with PKCE.
- The .NET backend validates the frontend's delegated Entra access token.
- The .NET backend calls downstream services using Entra app identity / client credentials flow.
- Downstream services are:
  - Python Chatbot Service
  - Python AI Workflow Service
- Frontend must not call Chatbot or AI Workflow directly.
- Backend should expose generic route prefixes:
  - `/api/chatbot/{**path}`
  - `/api/workflow/{**path}`
- Backend should not create one controller method per downstream API method.
- Backend should act as a **controlled proxy**, not a blind open proxy.

## Main goal

Implement backend proxy routes that allow the frontend to call:

```http
/api/chatbot/*
/api/workflow/*
```

The backend should then securely forward allowed requests to the correct downstream service using app identity.

## Security requirements

The proxy must:

1. Require authenticated users for all `/api/chatbot/*` and `/api/workflow/*` routes.
2. Validate the frontend delegated token using Microsoft Entra ID.
3. Use app identity / client credentials flow when calling Chatbot and AI Workflow.
4. Never forward the frontend's bearer token to downstream services.
5. Strip any frontend-supplied trusted headers before forwarding.
6. Regenerate trusted headers in the backend.
7. Use an allowlist/policy table to decide which downstream routes are exposed.
8. Reject any method/path combination that is not explicitly allowed.
9. Add correlation ID to all downstream requests.
10. Add audit logging for proxied requests and responses.
11. Apply request size limits and timeout handling.
12. Normalise downstream errors before returning to the frontend.

## Single tenant / single deal assumptions

For now:

- There is only one tenant.
- There is only one active deal/engagement.
- Do not build full tenant membership or engagement RBAC yet.
- Use a simple permission model initially.
- The backend may use a configured default deal ID if the frontend does not send one.
- Still include `X-Deal-Id` or `X-Engagement-Id` in trusted downstream headers so the contract is future-proof.

Suggested configuration:

```json
{
  "ProductScope": {
    "TenantId": "single-tenant",
    "DealId": "single-deal"
  }
}
```

## Required route shape

Implement a generic controller that handles:

```http
/api/chatbot/{**path}
/api/workflow/{**path}
```

Supported HTTP methods:

```text
GET
POST
PUT
PATCH
DELETE
```

Do not forward routes unless they match an explicit allowlist policy.

## Proxy route policy

Create a configuration-driven policy table.

Example:

```json
{
  "ProxyRoutes": {
    "Chatbot": [
      {
        "method": "POST",
        "pathPattern": "^/conversations$",
        "requiredPermission": "chatbot.use"
      },
      {
        "method": "POST",
        "pathPattern": "^/conversations/[^/]+/messages$",
        "requiredPermission": "chatbot.use"
      },
      {
        "method": "GET",
        "pathPattern": "^/conversations/[^/]+$",
        "requiredPermission": "chatbot.read"
      }
    ],
    "Workflow": [
      {
        "method": "POST",
        "pathPattern": "^/ingestion-runs$",
        "requiredPermission": "workflow.startIngestion"
      },
      {
        "method": "POST",
        "pathPattern": "^/gap-analysis-runs$",
        "requiredPermission": "workflow.startGapAnalysis"
      },
      {
        "method": "GET",
        "pathPattern": "^/runs/[^/]+$",
        "requiredPermission": "workflow.readRun"
      }
    ]
  }
}
```

The implementation should include:

- `ProxyRoutePolicy`
- `ProxyRouteOptions`
- `IProxyRoutePolicyResolver`
- Regex-based method/path matching
- Case-insensitive service matching
- Safe default: deny if no policy matches

## Trusted downstream headers

The backend must remove these headers if supplied by the frontend:

```text
Authorization
Host
Cookie
X-Tenant-Id
X-Deal-Id
X-Engagement-Id
X-User-Id
X-User-Display-Name
X-User-Email
X-Permissions
X-Required-Permission
X-Correlation-Id
```

The backend must add its own trusted headers:

```text
X-Correlation-Id
X-Tenant-Id
X-Deal-Id
X-Engagement-Id
X-User-Id
X-User-Display-Name
X-User-Email
X-Required-Permission
```

For now, use configured tenant/deal values:

```text
X-Tenant-Id: single-tenant
X-Deal-Id: single-deal
X-Engagement-Id: single-deal
```

User values should come from validated Entra claims, for example:

```text
oid
name
preferred_username
emails
```

## Permission handling

Because this is single tenant/single deal, implement a simple permission service now:

```csharp
public interface IProxyPermissionService
{
    Task<bool> IsAllowedAsync(
        ClaimsPrincipal user,
        string requiredPermission,
        CancellationToken cancellationToken);
}
```

Initial implementation may allow all authenticated users for MVP, but structure the code so it can later check app roles, groups, database roles, or per-deal permissions.

Suggested MVP behaviour:

- Authenticated user + allowed proxy route = permitted.
- Log the required permission for audit.
- Keep the interface ready for future RBAC.

## Downstream authentication

Use typed/named `HttpClient`s:

```text
Chatbot
Workflow
```

Use a delegating handler to acquire and attach app identity tokens.

Use `DefaultAzureCredential` for Azure-hosted environments, supporting managed identity.

Use these configuration values:

```json
{
  "DownstreamApis": {
    "Chatbot": {
      "BaseUrl": "https://chatbot-service.example.com",
      "Scope": "api://chatbot-api/.default"
    },
    "Workflow": {
      "BaseUrl": "https://workflow-service.example.com",
      "Scope": "api://workflow-api/.default"
    }
  }
}
```

The backend must call downstream services using its own app token, not the user's token.

## Proxy implementation details

Create:

```text
DownstreamProxyController
IDownstreamProxyClient
DownstreamProxyClient
IDownstreamTokenProvider
DownstreamTokenProvider
DownstreamAuthHandler
IProxyRoutePolicyResolver
ProxyRoutePolicyResolver
IProxyPermissionService
ProxyPermissionService
ProxyAuditService
```

The controller should:

1. Match service: `chatbot` or `workflow`.
2. Extract downstream path.
3. Resolve route policy from method + path.
4. Return `404` or `403` for non-allowed routes.
5. Check permission using `IProxyPermissionService`.
6. Build trusted header context.
7. Forward the request body and query string.
8. Attach backend app token for the target service.
9. Return downstream response body, status code, and safe response headers.
10. Avoid forwarding unsafe downstream headers.

## Safe request forwarding

When copying request headers, do not copy:

```text
Authorization
Host
Cookie
Content-Length
Transfer-Encoding
Connection
X-Tenant-Id
X-Deal-Id
X-Engagement-Id
X-User-Id
X-User-Display-Name
X-User-Email
X-Permissions
X-Required-Permission
X-Correlation-Id
```

When copying response headers, do not copy unsafe hop-by-hop headers:

```text
Transfer-Encoding
Connection
Keep-Alive
Proxy-Authenticate
Proxy-Authorization
TE
Trailer
Upgrade
```

## Error handling

Return normalised errors to frontend:

```json
{
  "errorCode": "PROXY_ROUTE_NOT_ALLOWED",
  "message": "This route is not exposed through the backend proxy.",
  "correlationId": "...",
  "retryable": false
}
```

Suggested error codes:

```text
PROXY_ROUTE_NOT_ALLOWED
PROXY_PERMISSION_DENIED
DOWNSTREAM_AUTH_FAILED
DOWNSTREAM_TIMEOUT
DOWNSTREAM_UNAVAILABLE
DOWNSTREAM_ERROR
REQUEST_TOO_LARGE
INVALID_PROXY_TARGET
```

Do not leak internal stack traces or raw Python exception details.

## Audit logging

Audit each proxied call with:

```text
correlationId
userId
userEmail
service
method
path
requiredPermission
permissionDecision
responseStatusCode
startedAtUtc
completedAtUtc
durationMs
errorCode
```

For MVP, console/Application Insights logging is acceptable. Structure it so it can later write to a database audit table.

## Timeouts and resilience

Configure separate timeout values:

```json
{
  "DownstreamApis": {
    "Chatbot": {
      "TimeoutSeconds": 120
    },
    "Workflow": {
      "TimeoutSeconds": 60
    }
  }
}
```

Use sensible retry policy only for safe transient failures. Do not blindly retry non-idempotent POST requests unless an idempotency key is present.

## Expected files/classes

Create or modify these files:

```text
Program.cs
Controllers/DownstreamProxyController.cs
Proxy/ProxyRouteOptions.cs
Proxy/ProxyRoutePolicy.cs
Proxy/ProxyRoutePolicyResolver.cs
Proxy/DownstreamProxyClient.cs
Proxy/IDownstreamProxyClient.cs
Proxy/ProxyPermissionService.cs
Proxy/IProxyPermissionService.cs
Proxy/ProxyAuditService.cs
Proxy/IProxyAuditService.cs
Auth/DownstreamTokenProvider.cs
Auth/IDownstreamTokenProvider.cs
Auth/DownstreamAuthHandler.cs
Options/ProductScopeOptions.cs
Models/ProxyErrorResponse.cs
```

Use clear namespaces matching the existing project.

## Acceptance criteria

The implementation is complete when:

1. Authenticated frontend users can call `/api/chatbot/*` and `/api/workflow/*`.
2. Unauthenticated requests are rejected.
3. Unknown downstream service names are rejected.
4. Method/path combinations not present in the policy table are rejected.
5. Frontend-supplied trusted headers are stripped.
6. Backend-generated trusted headers are added.
7. Backend calls Chatbot and Workflow using app identity tokens.
8. Frontend bearer tokens are never forwarded downstream.
9. Proxy requests are audited.
10. Errors are normalised.
11. The code supports single tenant/single deal now but is easy to extend later.
12. The implementation avoids creating one backend method per downstream service method.

## Important design rule

`/api/chatbot/*` and `/api/workflow/*` are generic route prefixes, not blind proxy tunnels.

Only configured and authorised downstream method/path combinations should be forwarded.

The backend remains the only frontend-facing entry point for Chatbot and AI Workflow.
