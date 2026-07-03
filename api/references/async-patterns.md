# Async patterns

All VideoGen workflow and tool endpoints are asynchronous. Workflows (`POST /v1/workflows/*`) return HTTP 202 with `{ workflowRunId, projectId, projectUrl }`; tools (`POST /v1/tools/...`) return HTTP 202 with `{ toolExecutionId }`. You choose how to get the result: poll for it, or receive it via webhook.

## Workflow runs

Workflows are the primary surface. Use the `pollWorkflowRun` SDK helper to wait for a run to reach a terminal status (`succeeded`, `failed`, `cancelled`).

```typescript
import { VideoGenClient, pollWorkflowRun } from "@videogen/sdk";

const client = new VideoGenClient({ token: process.env.VIDEOGEN_API_KEY });

const { workflowRunId, projectUrl } = await client.workflows.scriptToVideo({
  script:
    "Staying hydrated keeps your body and mind running at their best. Drinking enough water boosts your energy, focus, and mood. Keep a water bottle nearby and sip throughout the day.",
  visualStyle: {
    type: "AI_IMAGE",
    aiStyle: "loose watercolor illustration with visible brushstrokes and soft color bleeds",
  },
  quality: "HIGH",
  remixActions: [
    { type: "ENABLE_CAPTIONS" },
    { type: "SET_BACKGROUND_MUSIC", fileId: "vg_file_...", volume: 0.25 },
  ],
});

const run = await pollWorkflowRun(client, workflowRunId);

if (run.status === "succeeded") {
  console.log("Project URL:", projectUrl);
}
```

```python
import os
from videogen import VideoGenApi, poll_workflow_run

client = VideoGenApi(token=os.environ["VIDEOGEN_API_KEY"])

response = client.workflows.add_visuals_narrations_and_captions_to_script(
    script="Staying hydrated keeps your body and mind running at their best. Drinking enough water boosts your energy, focus, and mood. Keep a water bottle nearby and sip throughout the day.",
    visual_style={
        "type": "AI_IMAGE",
        "ai_style": "loose watercolor illustration with visible brushstrokes and soft color bleeds",
    },
    quality="HIGH",
    remix_actions=[
        {"type": "ENABLE_CAPTIONS"},
        {"type": "SET_BACKGROUND_MUSIC", "file_id": "vg_file_...", "volume": 0.25},
    ],
)
run = poll_workflow_run(client, response.workflow_run_id)

if run.status == "succeeded":
    print("Project URL:", response.project_url)
```

Subscribe to `workflow_run.succeeded`, `workflow_run.failed`, and `workflow_run.cancelled` webhooks to receive runs instead of polling. Export the finished project to MP4 with `POST /v1/projects/{projectId}/export` (poll with `pollProjectExport`).

## Tool execution lifecycle

```
POST /v1/tools/generate-image → { "toolExecutionId": "vg_tool_..." }
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

const { toolExecutionId } = await client.tools.generateImage({
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

response = client.tools.generate_image(prompt="A mountain at sunrise")
execution = poll_executed_tool(client, response.tool_execution_id)

if execution.status == "succeeded":
    print("File ID:", execution.results[0].file_id)
```

### Options

| Option | Type | Default | Description |
|---|---|---|---|
| `pollIntervalMs` / `poll_interval_ms` | number | 1500 | Milliseconds between polls |
| `signal` | AbortSignal | — | Abort signal to cancel polling (TypeScript only) |

### Manual polling

```bash
# Repeat until status is "succeeded", "failed", or "cancelled"
curl https://api.videogen.io/v1/tools/executions/vg_tool_... \
  -H "Authorization: Bearer $VIDEOGEN_API_KEY"
```

**When to use polling:** Scripts, CLI tools, or any situation where you can block and wait.

## Option 2: Webhooks

For production systems, register a webhook endpoint and VideoGen will POST to your URL when an execution reaches a terminal status. See [webhooks.md](webhooks.md) for full setup.

**When to use webhooks:** Production backends, serverless functions, anywhere you don't want to hold a connection open.

## Cancellation

Cancel an in-progress execution at any time:

```typescript
await client.tools.cancelToolExecution({ toolExecutionId: "vg_tool_..." });
```

```bash
curl -X POST https://api.videogen.io/v1/tools/executions/vg_tool_.../cancel \
  -H "Authorization: Bearer $VIDEOGEN_API_KEY"
```

If the execution hasn't completed yet, its status transitions to `cancelled`.

## Typical flow

```
1. POST /v1/tools/generate-image  →  { toolExecutionId: "vg_tool_..." }
2. Poll GET /v1/tools/executions/vg_tool_...  →  { status: "pending" }
3. Poll GET /v1/tools/executions/vg_tool_...  →  { status: "running" }
4. Poll GET /v1/tools/executions/vg_tool_...  →  { status: "succeeded", results: [...] }
5. GET /v1/files/{fileId}  →  file metadata
6. POST /v1/files/{fileId}/hydrate  →  file with signed download URLs
```

Steps 5–6 can be replaced with the `getHydratedFile` or `downloadFile` SDK helpers.
