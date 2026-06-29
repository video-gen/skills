---
name: api
description: >-
  Build with the VideoGen API for AI video generation. Use when generating
  end-to-end videos from a script, voiceover, or slideshow, or generating
  standalone images, video clips, voiceovers, sound effects, and avatars, or
  managing projects, files, and webhooks with VideoGen.
license: MIT
compatibility: Requires an API key from app.videogen.io/developers (VIDEOGEN_API_KEY).
metadata:
  { "openclaw": { "requires": { "env": ["VIDEOGEN_API_KEY"] }, "primaryEnv": "VIDEOGEN_API_KEY" } }
---

# VideoGen API

Turn a script, voiceover, or slideshow into a finished video with a single API call. End-to-end **video workflows** are the primary surface; the API also exposes standalone media tools (images, video clips, voiceovers, sound effects, avatars, and more). Everything is **asynchronous** — POST returns a run or execution id, poll until `succeeded`/`failed`/`cancelled`.

- **Docs**: <https://docs.videogen.io>
- **OpenAPI spec**: <https://docs.videogen.io/openapi.json>
- **Base URL**: `https://api.videogen.io`
- **Auth**: Bearer token (`Authorization: Bearer $VIDEOGEN_API_KEY`)
- **SDKs**: `@videogen/sdk` (npm) · `videogen` (PyPI)

## Quick start

```typescript
import { VideoGenClient, pollWorkflowRun } from "@videogen/sdk";

const client = new VideoGenClient({ token: process.env.VIDEOGEN_API_KEY });

const { workflowRunId, projectUrl } = await client.workflows.scriptToVideo({
  script:
    "Staying hydrated keeps your body and mind running at their best. Drinking enough water boosts your energy, focus, and mood. Keep a water bottle nearby and sip throughout the day.",
  remixActions: [
    { type: "ENABLE_CAPTIONS" },
    { type: "SET_BACKGROUND_MUSIC", fileId: "vg_file_...", volume: 0.25 },
  ],
});

const run = await pollWorkflowRun(client, workflowRunId);
// run.status → "succeeded"; projectUrl → open or export the video
```

```python
import os
from videogen import VideoGenApi, poll_workflow_run

client = VideoGenApi(token=os.environ["VIDEOGEN_API_KEY"])

response = client.workflows.add_visuals_narrations_and_captions_to_script(
    script="Staying hydrated keeps your body and mind running at their best. Drinking enough water boosts your energy, focus, and mood. Keep a water bottle nearby and sip throughout the day.",
    remix_actions=[
        {"type": "ENABLE_CAPTIONS"},
        {"type": "SET_BACKGROUND_MUSIC", "file_id": "vg_file_...", "volume": 0.25},
    ],
)
run = poll_workflow_run(client, response.workflow_run_id)
# run.status → "succeeded"; response.project_url → open or export the video
```

## Core concepts

1. **Workflows are the primary surface.** `POST /v1/workflows/*` returns `{ workflowRunId, projectId, projectUrl }` with HTTP 202. Poll `GET /v1/workflows/runs/{id}` or use `pollWorkflowRun` until a terminal status. See [async-patterns.md](references/async-patterns.md).
2. **Tool calls are async too.** `POST /v1/tools/...` returns `{ toolExecutionId }` with HTTP 202. Poll `GET /v1/tools/executions/{id}` or use `pollExecutedTool`.
3. **IDs are prefixed strings.** `vg_work_...`, `vg_tool_...`, `vg_file_...`, `vg_voic_...`, `vg_pres_...`. Store as-is.
4. **Files require hydration for download URLs.** Generated files have signed URLs that expire. Use `getHydratedFile` or `POST /v1/files/{id}/hydrate` to refresh them. See [files-and-resources.md](references/files-and-resources.md).
5. **Webhooks follow the Standard Webhooks spec.** Register an endpoint, verify signatures with `verifyWebhookSignature`. See [webhooks.md](references/webhooks.md).

## Workflows (primary surface)

| Method                                        | Endpoint                                                              | Description                                              |
| --------------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------- |
| `scriptToVideo`     | `POST /v1/workflows/script-to-video`    | Turn a script (used verbatim) into a narrated video     |
| `voiceoverToVideo`            | `POST /v1/workflows/voiceover-to-video`           | Build a video from an uploaded voiceover                 |
| `slideshowToVideo` | `POST /v1/workflows/slideshow-to-video` | Build a narrated video from a PDF or slideshow      |
| `storyboardToVideo`                | `POST /v1/workflows/storyboard-to-video`                | Build a video from a list of scenes (image or video clip each) |
| `getWorkflowRun`                              | `GET /v1/workflows/runs/{workflowRunId}`                            | Poll run status and `projectUrl`                         |

## Projects

| Method            | Endpoint                                             | Description                          |
| ----------------- | --------------------------------------------------- | ------------------------------------ |
| `listProjects`    | `GET /v1/projects`                                  | List projects (API-created by default; pass `includeUiProjects=true` for dashboard projects too) |
| `getProject`      | `GET /v1/projects/{projectId}`                      | Project metadata and `projectUrl`    |
| `exportProject`   | `POST /v1/projects/{projectId}/export`              | Start an MP4 export → `{ exportId }` |
| `getProjectExport`| `GET /v1/projects/{projectId}/exports/{exportId}`   | Poll export; `downloadUrl` when done |

## Tools (standalone media generation)

| Method                  | Endpoint                                 | Description                                               |
| ----------------------- | ---------------------------------------- | --------------------------------------------------------- |
| `generateImage`         | `POST /v1/tools/generate-image`          | Generate images from text or image |
| `generateVideoClip`     | `POST /v1/tools/generate-video-clip`     | Generate video from text, image, or video                 |
| `textToSpeech`          | `POST /v1/tools/text-to-speech`          | Text to speech voiceover                                  |
| `generateSoundEffect`   | `POST /v1/tools/generate-sound-effect`   | Generate sound effects                                    |
| `generateAvatar`        | `POST /v1/tools/generate-avatar`         | Avatar presenter video                                    |
| `vectorizeImage`        | `POST /v1/tools/vectorize-image`         | Raster to SVG                                             |
| `removeImageBackground` | `POST /v1/tools/remove-image-background` | Remove image background                                   |
| `removeVideoBackground` | `POST /v1/tools/remove-video-background` | Remove video background                                   |
| `upscaleImage`          | `POST /v1/tools/upscale-image`           | Upscale image resolution                                  |
| `upscaleVideo`          | `POST /v1/tools/upscale-video`           | Upscale video resolution                                  |

Full parameter schemas and examples: [tools.md](references/tools.md)

## Text generation

| Method         | Endpoint                | Description                                                   |
| -------------- | ----------------------- | ------------------------------------------------------------ |
| `generateText` | `POST /v1/text/generate`| Generate text from a prompt with a fast language model (sync)|

Unlike workflows and tools, text generation is **synchronous** — the response includes the generated `text` directly, no polling. Pass `prompt` (required), optional `system` instructions, a `model` quality tier (`LOW`, `STANDARD`, or `HIGH`, default `STANDARD`), `temperature`, and `maxOutputTokens` (default 512, max 2000). Useful for drafting scripts, titles, and descriptions before generating a video.

```typescript
const { text } = await client.text.generateText({
  prompt: "Write a punchy 30-second video script about staying hydrated.",
});
```

```python
response = client.text.generate_text(
    prompt="Write a punchy 30-second video script about staying hydrated.",
)
# response.text
```

## Entities

Reusable actors and visual styles shared across your team. Attach their reference images to workflows for a consistent character (`ACTOR`) or look (`VISUAL_STYLE`). Use an entity in a workflow by passing its `vg_enti_...` id: `visualStyle: { type: "ENTITY", entityId }` (script + voiceover), or per-scene `actorEntityId` / `visualStyleEntityId` on `storyboardToVideo`.

| Method                  | Endpoint                                       | Description                                                          |
| ----------------------- | ---------------------------------------------- | ------------------------------------------------------------------- |
| `listEntities`          | `GET /v1/entities`                             | List entities (filter with `?entityType=ACTOR` or `VISUAL_STYLE`)   |
| `createEntity`          | `POST /v1/entities`                            | Create an entity (`entityType`, `name`, optional `description`)     |
| `getEntity`             | `GET /v1/entities/{entityId}`                  | Get one entity with its reference images                            |
| `updateEntity`          | `POST /v1/entities/{entityId}/update`          | Update `name` or `description`                                      |
| `archiveEntity`         | `POST /v1/entities/{entityId}/archive`         | Archive an entity                                                   |
| `addEntityReference`    | `POST /v1/entities/{entityId}/references`      | Attach an uploaded image as a reference (`fileId`, `description`, `isDefault`) |
| `removeEntityReference` | `POST /v1/entities/{entityId}/references/remove` | Detach a reference image (`fileId`)                               |

Reference images are regular files: upload an image via `POST /v1/files/upload`, then attach its `fileId` to the entity.

## SDK helpers (not auto-generated)

| Helper                                             | Purpose                                                    |
| -------------------------------------------------- | ---------------------------------------------------------- |
| `pollWorkflowRun(client, workflowRunId, opts?)`    | Poll a workflow run until terminal status                  |
| `pollProjectExport(client, projectId, exportId)`   | Poll a project export until terminal status                |
| `pollExecutedTool(client, executionId, opts?)`     | Poll a tool execution until terminal status               |
| `uploadFile(client, file, opts)`                   | Create presigned upload → PUT bytes → poll until processed |
| `downloadFile(client, fileId, opts?)`              | Hydrate URL → download (stream to disk or return Response) |
| `getHydratedFile(client, fileId)`                  | Get file, re-hydrate if URLs expired                       |
| `verifyWebhookSignature(rawBody, headers, secret)` | Standard Webhooks signature verification                   |

## Reference files

- **[tools.md](references/tools.md)** — All 13 tool endpoints with parameters and examples
- **[files-and-resources.md](references/files-and-resources.md)** — File upload, download, hydration, avatar presenters, TTS voices
- **[webhooks.md](references/webhooks.md)** — Webhook CRUD and signature verification
- **[async-patterns.md](references/async-patterns.md)** — Polling, webhook delivery, cancellation

## Need more info?

If the reference files above don't cover what you need, consult the published API documentation:

- **Full docs (human-readable)**: <https://docs.videogen.io>
- **LLM-optimized overview**: <https://docs.videogen.io/llms.txt>
- **Complete docs with SDK examples**: <https://docs.videogen.io/llms-full.txt>
- **OpenAPI spec (JSON)**: <https://docs.videogen.io/openapi.json>
- **MCP server** (for AI clients): <https://docs.videogen.io/_mcp/server>

Use `llms.txt` for a quick scan of all endpoints and guide pages. Use `llms-full.txt` when you need complete page content including SDK code examples. Fetch the OpenAPI spec for exact request/response schemas.
