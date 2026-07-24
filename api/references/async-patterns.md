# Async patterns

All VideoGen workflow and tool endpoints are asynchronous. Workflows (`POST /v1/workflows/*`) return HTTP 202 with `{ workflowRunId, projectId, projectUrl }`; tools (`POST /v1/tools/...`) return HTTP 202 with `{ toolExecutionId }`. You choose how to get the result: poll for it, or receive it via webhook.

## Workflow runs

Workflows are the primary surface. Prefer `scriptToVideoAndWait` (and the other `*AndWait` workflow methods) when you can block. They start the run and wait until a terminal status (`succeeded`, `failed`, `cancelled`). For start-then-poll yourself, use `pollWorkflowRun`.

```typescript
import { VideoGen } from "@videogen/sdk";

const vg = new VideoGen({ apiKey: "sk_videogen_live_..." });

const run = await vg.workflows.scriptToVideoAndWait({
  script:
    "Staying hydrated keeps your body and mind running at their best. Drinking enough water boosts your energy, focus, and mood. Keep a water bottle nearby and sip throughout the day.",
  visualStyle: {
    type: "AI_IMAGE",
    aiStyle: "loose watercolor illustration with visible brushstrokes and soft color bleeds",
  },
  quality: "HIGH",
  remixActions: [
    { type: "ENABLE_CAPTIONS" },
    {
      type: "CONVERT_IMAGES_TO_VIDEOS",
      motionPrompt: "slow cinematic push-in",
      muteOutputVideos: true,
      quality: "HIGH",
    },
  ],
});

if (run.status === "succeeded") {
  console.log("Project ID:", run.projectId);
}
```

```python
from videogen import VideoGen

vg = VideoGen(api_key="sk_videogen_live_...")

run = vg.workflows.script_to_video_and_wait(
    script="Staying hydrated keeps your body and mind running at their best. Drinking enough water boosts your energy, focus, and mood. Keep a water bottle nearby and sip throughout the day.",
    visual_style={
        "type": "AI_IMAGE",
        "ai_style": "loose watercolor illustration with visible brushstrokes and soft color bleeds",
    },
    quality="HIGH",
    remix_actions=[
        {"type": "ENABLE_CAPTIONS"},
        {
            "type": "CONVERT_IMAGES_TO_VIDEOS",
            "motion_prompt": "slow cinematic push-in",
            "mute_output_videos": True,
            "quality": "HIGH",
        },
    ],
)

if run["status"] == "succeeded":
    print("Project ID:", run.get("project_id"))
```

Subscribe to `workflow_run.succeeded`, `workflow_run.failed`, and `workflow_run.cancelled` webhooks to receive runs instead of polling. Export the finished project to MP4 with `POST /v1/projects/{projectId}/export` (poll with `pollProjectExport`, or use `exportAndWait`).

### Cancel a workflow run

```typescript
await vg.workflows.cancelWorkflowRun({ workflowRunId: "vg_work_..." });
```

```bash
curl -X POST https://api.videogen.io/v1/workflows/runs/vg_work_.../cancel \
  -H "Authorization: Bearer sk_videogen_live_..."
```

## Remix actions

After a workflow succeeds, apply edits with `POST /v1/projects/{projectId}/remix`. The response includes `remixActionIds` (one per action, in order). Prefer `remixAndWait`, or poll with `pollRemixActions` / `GET /v1/projects/{projectId}/remix-actions` until each action reaches a terminal status.

```typescript
const { remixActionIds } = await vg.projects.remixProject({
  projectId: "vg_proj_...",
  remixActions: [
    { type: "ENABLE_CAPTIONS" },
    {
      type: "CONVERT_IMAGES_TO_VIDEOS",
      motionPrompt: "slow cinematic push-in",
      muteOutputVideos: true,
      quality: "HIGH",
    },
  ],
});

const { remixActions } = await vg.projects.listProjectRemixActions({
  projectId: "vg_proj_...",
});
// remixActions[0].status → "succeeded"
```

Remix action statuses: `pending`, `running`, `succeeded`, `failed`, `cancelled`.

## Tool execution lifecycle

```
POST /v1/tools/generate-image → { "toolExecutionId": "vg_tool_..." }
```

Statuses progress through:

| Status | Terminal? | Meaning |
|---|---|---|
| `pending` | no | Queued, waiting to start |
| `running` | no | Generation in progress |
| `succeeded` | yes | Done: `results` array is populated |
| `failed` | yes | Something went wrong: `error` is populated |
| `cancelled` | yes | Cancelled by you |

## Option 1: Polling

Prefer `generateImageAndWait` (and the other tool `*AndWait` methods) when you can block. For start-then-poll yourself, use `pollExecutedTool` (it calls `GET /v1/tools/executions/{toolExecutionId}` until a terminal status).

```typescript
import { VideoGen, createPublicPreview } from "@videogen/sdk";

const vg = new VideoGen({ apiKey: "sk_videogen_live_..." });

const execution = await vg.tools.generateImageAndWait({
  prompt: "A mountain at sunrise",
});

if (execution.status === "succeeded") {
  const fileId = execution.results?.[0]?.fileId;
  const preview = await createPublicPreview({ client: vg, fileId });
  console.log("File ID:", fileId);
}
```

```python
from videogen import VideoGen, create_public_preview

vg = VideoGen(api_key="sk_videogen_live_...")

execution = vg.tools.generate_image_and_wait(prompt="A mountain at sunrise")

if execution["status"] == "succeeded":
    file_id = execution["results"][0]["file_id"]
    preview = create_public_preview(vg, file_id)
    print("File ID:", file_id)
```

### Options

| Option | Type | Default | Description |
|---|---|---|---|
| `pollIntervalMs` / `poll_interval_ms` | number | 1500 | Milliseconds between polls |
| `signal` | AbortSignal | (none) | Abort signal to cancel polling (TypeScript) |
| `cancel_event` | `threading.Event` / `asyncio.Event` | (none) | Cancel polling early (Python; raises `PollCancelledError`) |

### Manual polling

```bash
# Repeat until status is "succeeded", "failed", or "cancelled"
curl https://api.videogen.io/v1/tools/executions/vg_tool_... \
  -H "Authorization: Bearer sk_videogen_live_..."
```

**When to use polling:** Scripts, CLI tools, or any situation where you can block and wait.

## Option 2: Webhooks

For production systems, register a webhook endpoint and VideoGen will POST to your URL when an execution reaches a terminal status. See [webhooks.md](webhooks.md) for full setup.

**When to use webhooks:** Production backends, serverless functions, anywhere you don't want to hold a connection open.

## Cancellation

Cancel an in-progress execution at any time:

```typescript
await vg.tools.cancelToolExecution({ toolExecutionId: "vg_tool_..." });
```

```bash
curl -X POST https://api.videogen.io/v1/tools/executions/vg_tool_.../cancel \
  -H "Authorization: Bearer sk_videogen_live_..."
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

Steps 1–4 can be replaced with `generateImageAndWait` (or `pollExecutedTool`). Steps 5–6 can be replaced with `getHydratedFile` or `downloadFile`. For a shareable public preview URL, use `createPublicPreview`.
