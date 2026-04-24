# Webhooks

Register webhook endpoints to receive events when tool executions complete, instead of polling.

Webhooks follow the [Standard Webhooks](https://www.standardwebhooks.com/) spec.

## Events

| Event | Fired when |
|---|---|
| `tool_execution.succeeded` | Execution completed successfully |
| `tool_execution.failed` | Execution failed |
| `tool_execution.cancelled` | Execution was cancelled |

## Create a webhook endpoint

**Endpoint:** `POST /v1/webhooks/endpoints`

| Param | Type | Required | Description |
|---|---|---|---|
| `url` | string (HTTPS) | yes | URL to receive POST requests |
| `events` | string[] | yes | Events to subscribe to |
| `description` | string | no | Description of the endpoint |

Returns a `WebhookEndpoint` with a `signingSecret`. **The signing secret is only returned once** — store it securely.

```typescript
const endpoint = await client.webhooks.createWebhookEndpoint({
  url: "https://your-server.com/webhooks/videogen",
  events: ["tool_execution.succeeded", "tool_execution.failed"],
});

// Store endpoint.signingSecret securely — it is only returned on create
```

## List webhook endpoints

**Endpoint:** `GET /v1/webhooks/endpoints`

Returns `{ endpoints: WebhookEndpoint[] }`.

```typescript
const { endpoints } = await client.webhooks.listWebhookEndpoints();
```

## Delete a webhook endpoint

**Endpoint:** `DELETE /v1/webhooks/endpoints/{endpointId}`

Returns `204 No Content`. The endpoint stops receiving events immediately.

```typescript
await client.webhooks.deleteWebhookEndpoint({ endpointId: "wh_endpoint_..." });
```

## Webhook payload

When an execution reaches a terminal status, VideoGen sends a POST to your URL:

```json
{
  "event": "tool_execution.succeeded",
  "toolExecutionId": "vg_exec_...",
  "toolType": "PROMPT_TO_IMAGE",
  "occurredAt": 1745409600,
  "results": [
    {
      "storageFileId": "vg_file_...",
      "type": "IMAGE",
      "file": {
        "storageFileId": "vg_file_...",
        "type": "IMAGE",
        "scope": "GLOBAL",
        "displayName": "A mountain at sunrise",
        "downloadSource": {
          "status": "ready",
          "url": "https://...",
          "expiresAt": 1745413200
        }
      }
    }
  ]
}
```

Webhook payloads for `succeeded` events include hydrated `file` objects with signed download URLs — no extra API call needed. For `failed` and `cancelled` events, `results` is absent and `error` may be present.

## Verify signatures

Use the `verifyWebhookSignature` SDK helper to confirm a request is authentic. It throws if the signature is invalid or the timestamp is too old.

Required headers: `webhook-id`, `webhook-timestamp`, `webhook-signature`.

```typescript
import { verifyWebhookSignature } from "@videogen/sdk";

// In your webhook handler (e.g. Express, Next.js API route):
const payload = verifyWebhookSignature(
  rawBody,         // raw request body string (not parsed JSON)
  request.headers, // must include webhook-id, webhook-timestamp, webhook-signature
  signingSecret,   // the secret returned when you created the endpoint
);

console.log(payload.event);           // "tool_execution.succeeded"
console.log(payload.toolExecutionId); // "vg_exec_..."
```

```python
from videogen import verify_webhook_signature

payload = verify_webhook_signature(
    raw_body=request.data.decode(),
    headers=dict(request.headers),
    signing_secret=signing_secret,
)
```

Under the hood this wraps the `standardwebhooks` library:

```typescript
import { Webhook } from "standardwebhooks";
const wh = new Webhook(signingSecret);
const payload = wh.verify(rawBody, headers);
```

## WebhookEndpoint shape

| Field | Type | Description |
|---|---|---|
| `endpointId` | string | Endpoint ID |
| `url` | string | Registered URL |
| `events` | string[] | Subscribed events |
| `description` | string \| null | Description |
| `createdAt` | number | Unix timestamp |
| `signingSecret` | string | HMAC secret (only on create response) |
| `signingSecretLast4` | string | Last 4 characters of the secret |
