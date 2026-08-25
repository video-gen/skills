# Webhooks

Register webhook endpoints to receive events when tool executions, workflow runs, project exports, or file uploads complete, instead of polling.

Webhooks follow the [Standard Webhooks](https://www.standardwebhooks.com/) spec.

## Events

### Tool execution events

| Event | Fired when |
|---|---|
| `tool_execution.succeeded` | Execution completed successfully |
| `tool_execution.failed` | Execution failed |
| `tool_execution.cancelled` | Execution was cancelled |

### Workflow run events

| Event | Fired when |
|---|---|
| `workflow_run.succeeded` | Workflow run completed successfully |
| `workflow_run.failed` | Workflow run failed |
| `workflow_run.cancelled` | Workflow run was cancelled |

Payload includes `workflowRunId`, `workflowType`, `projectId`, and `projectUrl`.

### Project export events (API exports only)

| Event | Fired when |
|---|---|
| `project_export.succeeded` | Export completed successfully |
| `project_export.failed` | Export failed |
| `project_export.cancelled` | Export was cancelled |

Payload includes `exportId`, `projectId`, `occurredAt`, and `exportFileId` (`null` until succeeded). Use `GET /v1/projects/{projectId}/exports/{exportId}` for signed download URLs.

### File events (API uploads only)

| Event | Fired when |
|---|---|
| `file.upload.completed` | File upload finished and the file is stored |
| `file.upload.failed` | API file upload failed |
| `file.playback_ready` | HLS streaming is available (private `hlsSource`, plus public playback if enabled) |
| `file.download_ready` | At least one static-rendition download URL is ready |
| `file.analysis_completed` | Description, transcript, and vector embedding are ready (file is searchable) |
| `file.analysis_failed` | File analysis failed |

Each file event payload includes a hydrated `FileInfo` object. `file.analysis_completed` and `file.analysis_failed` never fire for `scope: "TEMPORARY"` files.

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
  events: [
    "tool_execution.succeeded",
    "tool_execution.failed",
    "workflow_run.succeeded",
    "workflow_run.failed",
    "project_export.succeeded",
    "project_export.failed",
    "file.upload.completed",
    "file.download_ready",
  ],
});

// Store endpoint.signingSecret securely — it is only returned on create
```

## List webhook endpoints

**Endpoint:** `GET /v1/webhooks/endpoints`

Returns `{ endpoints: WebhookEndpoint[], hasMore: boolean, nextCursor: string | null }`. Cursor-paginated; see <https://docs.videogen.io/pagination>.

```typescript
const { endpoints, hasMore, nextCursor } = await client.webhooks.listWebhookEndpoints();
```

## Delete a webhook endpoint

**Endpoint:** `DELETE /v1/webhooks/endpoints/{endpointId}`

Returns `204 No Content`. The endpoint stops receiving events immediately.

```typescript
await client.webhooks.deleteWebhookEndpoint({ endpointId: "ep_..." });
```

## Webhook payload (tool execution)

When a tool execution reaches a terminal status, VideoGen sends a POST to your URL:

```json
{
  "event": "tool_execution.succeeded",
  "toolExecutionId": "vg_tool_...",
  "toolType": "GENERATE_IMAGE",
  "occurredAt": 1745409600,
  "results": [
    {
      "fileId": "vg_file_...",
      "type": "IMAGE",
      "file": {
        "fileId": "vg_file_...",
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

## Webhook payload (workflow run)

```json
{
  "event": "workflow_run.succeeded",
  "workflowRunId": "vg_work_...",
  "workflowType": "SCRIPT_TO_VIDEO",
  "projectId": "vg_proj_...",
  "projectUrl": "https://app.videogen.io/project/...",
  "occurredAt": 1745409600
}
```

## Verify signatures

Use the `verifyWebhookSignature` SDK helper to confirm a request is authentic. It throws if the signature is invalid or the timestamp is too old.

Required headers: `webhook-id`, `webhook-timestamp`, `webhook-signature`.

```typescript
import { verifyWebhookSignature } from "@videogen/sdk";

// In your webhook handler (e.g. Express, Next.js API route):
const payload = verifyWebhookSignature({
  rawBody, // raw request body string (not parsed JSON)
  headers: request.headers, // must include webhook-id, webhook-timestamp, webhook-signature
  secret: signingSecret, // the secret returned when you created the endpoint
});

console.log(payload.event); // "tool_execution.succeeded"
console.log(payload.toolExecutionId); // "vg_tool_..."
```

```python
from videogen import verify_webhook_signature

payload = verify_webhook_signature(
    raw_body=request.data.decode(),
    headers=dict(request.headers),
    secret=signing_secret,
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
| `endpointId` | string | Endpoint ID (e.g. `ep_...`) |
| `url` | string | Registered URL |
| `events` | string[] | Subscribed events |
| `description` | string \| null | Description |
| `createdAt` | number | Unix timestamp |
| `signingSecret` | string | HMAC secret (only on create response) |
| `signingSecretLast4` | string | Last 4 characters of the secret |
