# Async patterns

All VideoGen tool endpoints are asynchronous. `POST /v1/tools/...` returns HTTP 202 with `{ toolExecutionId }`. You choose how to get the result: poll for it, or receive it via webhook.

## Execution lifecycle

```
POST /v1/tools/prompt-to-image → { "toolExecutionId": "vg_exec_..." }
```

Statuses progress through:

| Status | Terminal? | Meaning |
|---|---|---|
| `pending` | no | Queued, waiting to start |
| `running` | no | Generation in progress |
| `succeeded` | yes | Done — `results` array is populated |
| `failed` | yes | Something went wrong — `error` is populated |
| `cancelled` | yes | Cancelled by you |

## Option 1: Polling

Use the `pollExecutedTool` SDK helper. It calls `GET /v1/tools/executions/{toolExecutionId}` in a loop until a terminal status is reached.

```typescript
import { VideoGenClient, pollExecutedTool } from "@videogen/sdk";

const client = new VideoGenClient({ token: process.env.VIDEOGEN_API_KEY });

const { toolExecutionId } = await client.tools.promptToImage({
  prompt: "A mountain at sunrise",
});

const response = await pollExecutedTool(client, toolExecutionId);

if (response.status === "succeeded") {
  console.log("File ID:", response.results[0].fileId);
}
```

```python
from videogen import VideoGenApi, poll_executed_tool

client = VideoGenApi(token=os.environ["VIDEOGEN_API_KEY"])

response = client.tools.prompt_to_image(prompt="A mountain at sunrise")
execution = poll_executed_tool(client, response.tool_execution_id)

if execution.status == "succeeded":
    print("File ID:", execution.results[0].storage_file_id)
```

### Options

| Option | Type | Default | Description |
|---|---|---|---|
| `pollIntervalMs` / `poll_interval_ms` | number | 1500 | Milliseconds between polls |
| `signal` | AbortSignal | — | Abort signal to cancel polling (TypeScript only) |

### Manual polling

```bash
# Repeat until status is "succeeded", "failed", or "cancelled"
curl https://api.videogen.io/v1/tools/executions/vg_exec_... \
  -H "Authorization: Bearer $VIDEOGEN_API_KEY"
```

**When to use polling:** Scripts, CLI tools, or any situation where you can block and wait.

## Option 2: Webhooks

For production systems, register a webhook endpoint and VideoGen will POST to your URL when an execution reaches a terminal status. See [webhooks.md](webhooks.md) for full setup.

**When to use webhooks:** Production backends, serverless functions, anywhere you don't want to hold a connection open.

## Cancellation

Cancel an in-progress execution at any time:

```typescript
await client.tools.cancelToolExecution({ toolExecutionId: "vg_exec_..." });
```

```bash
curl -X POST https://api.videogen.io/v1/tools/executions/vg_exec_.../cancel \
  -H "Authorization: Bearer $VIDEOGEN_API_KEY"
```

If the execution hasn't completed yet, its status transitions to `cancelled`.

## Typical flow

```
1. POST /v1/tools/prompt-to-image  →  { toolExecutionId: "vg_exec_..." }
2. Poll GET /v1/tools/executions/vg_exec_...  →  { status: "pending" }
3. Poll GET /v1/tools/executions/vg_exec_...  →  { status: "running" }
4. Poll GET /v1/tools/executions/vg_exec_...  →  { status: "succeeded", results: [...] }
5. GET /v1/files/{fileId}  →  file metadata
6. POST /v1/files/{fileId}/hydrate  →  file with signed download URLs
```

Steps 5–6 can be replaced with the `getHydratedFile` or `downloadFile` SDK helpers.
