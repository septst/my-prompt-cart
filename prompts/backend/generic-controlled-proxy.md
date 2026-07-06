# Prompt: Implement Generic Backend Proxy for Chatbot and AI Workflow

Use this prompt with GitHub Copilot to implement a simple, secure backend proxy in the .NET backend.

---

## Prompt

You are a senior .NET backend engineer.

Implement a **generic proxy** in the ASP.NET Core backend so the React frontend can call downstream AI services through the backend only.

## Context

- React frontend calls only the .NET backend.
- React frontend already uses Microsoft Entra ID Authorization Code Flow with PKCE to call the backend.
- Backend validates the user's delegated token.
- Backend calls downstream services using Microsoft Entra app identity / client credentials flow.
- Downstream services are:
  - Python Chatbot Service
  - Python AI Workflow Service
- Product is currently **single tenant / single deal**.
- Do not implement complex multi-tenant RBAC yet.

## What to build

Create generic backend proxy routes for:

```http
/api/chatbot/{**path}
/api/workflow/{**path}
```

These route names are examples/prefixes. Keep the implementation flexible so exact downstream paths can evolve without creating one backend method per downstream API method.

The backend should forward requests to the configured downstream service:

```text
/api/chatbot/*  -> Chatbot Service
/api/workflow/* -> AI Workflow Service
```

## Required behaviour

1. Require authenticated users for all proxy routes.
2. Do not forward the frontend bearer token downstream.
3. Use backend app identity / client credentials to call Chatbot and AI Workflow.
4. Forward the original request method, path, query string, and body.
5. Wrap the forwarded payload with only:
   - `correlationId`
   - `dealId`
   - `userId`
   - `payload`
6. Keep this wrapping contract the same for both Chatbot and AI Workflow.
7. Strip any frontend-supplied trusted headers or identity fields.
8. Add basic logging/audit using correlation ID.
9. Return downstream status code and response body to the frontend.
10. Keep the implementation simple and Copilot-friendly.

## Payload wrapping contract

If the frontend sends this body:

```json
{
  "message": "What are the open risks?"
}
```

The backend should forward this to the downstream service:

```json
{
  "correlationId": "generated-or-existing-correlation-id",
  "dealId": "configured-single-deal-id",
  "userId": "entra-user-object-id",
  "payload": {
    "message": "What are the open risks?"
  }
}
```

For requests without a JSON body, use:

```json
{
  "correlationId": "generated-or-existing-correlation-id",
  "dealId": "configured-single-deal-id",
  "userId": "entra-user-object-id",
  "payload": null
}
```

## Configuration

Use app settings like:

```json
{
  "ProductScope": {
    "DealId": "single-deal"
  },
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

## Suggested implementation

Create:

```text
Controllers/DownstreamProxyController.cs
Proxy/DownstreamProxyClient.cs
Proxy/IDownstreamProxyClient.cs
Auth/DownstreamTokenProvider.cs
Auth/IDownstreamTokenProvider.cs
Auth/DownstreamAuthHandler.cs
Options/ProductScopeOptions.cs
Options/DownstreamApiOptions.cs
Models/DownstreamProxyEnvelope.cs
```

Use named `HttpClient`s:

```text
Chatbot
Workflow
```

Use a delegating handler to attach the backend app-identity token for the correct downstream API.

Use `DefaultAzureCredential` for token acquisition so managed identity works in Azure and developer credentials work locally.

## Controller behaviour

The proxy controller should:

1. Accept `/api/chatbot/{**path}` and `/api/workflow/{**path}`.
2. Map `chatbot` to the Chatbot named `HttpClient`.
3. Map `workflow` to the Workflow named `HttpClient`.
4. Reject any other service name.
5. Get or create a correlation ID.
6. Get `userId` from the validated Entra claim, preferably `oid`.
7. Get `dealId` from configuration.
8. Read the frontend request body as JSON when present.
9. Create `DownstreamProxyEnvelope`.
10. Forward to the downstream path with the original query string.
11. Return downstream status code and response body.

## Model

```csharp
public sealed record DownstreamProxyEnvelope(
    string CorrelationId,
    string DealId,
    string UserId,
    object? Payload);
```

## Important security notes

- The frontend must not call Chatbot or AI Workflow directly.
- The backend must not forward the user's bearer token.
- The backend must call downstream services using its own app identity.
- Downstream services should trust `correlationId`, `dealId`, and `userId` only because the request came from the backend app identity.
- Keep route examples as examples only. Do not hard-code assumptions about Chatbot or Workflow business APIs beyond the `/api/chatbot/*` and `/api/workflow/*` proxy prefixes.

## Acceptance criteria

The implementation is done when:

1. Authenticated frontend users can call backend proxy routes.
2. `/api/chatbot/*` forwards to Chatbot Service.
3. `/api/workflow/*` forwards to AI Workflow Service.
4. Downstream calls use app identity / client credentials.
5. Forwarded request body is wrapped as `{ correlationId, dealId, userId, payload }`.
6. Frontend bearer token is never forwarded downstream.
7. The implementation does not require one backend method per downstream API method.
