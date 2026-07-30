---
name: api
description: >-
  Build with the VideoGen API for AI video generation. Use when generating
  end-to-end videos from a script, voiceover, or slideshow, or generating
  standalone images, video clips, voiceovers, sound effects, and avatars, or
  managing projects, files, and webhooks with VideoGen.
license: MIT
compatibility: Requires an API key from app.videogen.io/api (VIDEOGEN_API_KEY).
metadata:
  { "openclaw": { "requires": { "env": ["VIDEOGEN_API_KEY"] }, "primaryEnv": "VIDEOGEN_API_KEY" } }
---

# VideoGen API

Turn a script, voiceover, or slideshow into a finished video with a single API call. End-to-end **video workflows** are the primary surface; the API also exposes standalone media tools (images, video clips, voiceovers, sound effects, avatars, and more). Everything is **asynchronous**: POST returns a run or execution id, poll until `succeeded`/`failed`/`cancelled`.

- **Docs**: <https://docs.videogen.io>
- **OpenAPI spec**: <https://docs.videogen.io/openapi.json>
- **Base URL**: `https://api.videogen.io`
- **Auth**: Bearer token (`Authorization: Bearer sk_videogen_live_...`)
- **SDKs**: `@videogen/sdk` (npm) · `videogen` (PyPI)

## Quick start

```typescript
import { VideoGen, createPublicPreview } from "@videogen/sdk";

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
// run.status → "succeeded"; use run.projectId for export/remix (projectUrl is optional: app editor link for manual review)

const execution = await vg.tools.generateImageAndWait({
  prompt: "A mountain at sunrise",
});
const fileId = execution.results?.[0]?.fileId;
const preview = await createPublicPreview({ client: vg, fileId });
```

```python
from videogen import VideoGen, create_public_preview

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
# run["status"] → "succeeded"; use run.get("project_id") for export/remix (project_url is optional: app editor link)

execution = vg.tools.generate_image_and_wait(prompt="A mountain at sunrise")
file_id = execution["results"][0]["file_id"]
preview = create_public_preview(vg, file_id)
```

Prefer `*AndWait` / `*_and_wait` when you can block. For start-then-poll yourself, use `pollWorkflowRun` / `poll_workflow_run` (or `pollExecutedTool` / `poll_executed_tool`). Python responses use snake_case keys.

## Core concepts

1. **Workflows are the primary surface.** `POST /v1/workflows/*` returns `{ workflowRunId, projectId, projectUrl }` with HTTP 202. Use `projectId` for export/remix; `projectUrl` is optional (opens the project in the app editor; team and collaborators only). Prefer `scriptToVideoAndWait` (or poll `GET /v1/workflows/runs/{id}` / `pollWorkflowRun`) until a terminal status. See [async-patterns.md](references/async-patterns.md).
2. **Remix actions polish a project.** After a workflow succeeds, call `POST /v1/projects/{projectId}/remix` to layer on music, logos, captions, transitions, or natural-language edits. Poll `GET /v1/projects/{projectId}/remix-actions` for status (or `pollRemixActions` / `remixAndWait`).
3. **Tool calls are async too.** `POST /v1/tools/...` returns `{ toolExecutionId }` with HTTP 202. Prefer `generateImageAndWait` (or poll `GET /v1/tools/executions/{id}` / `pollExecutedTool`).
4. **IDs are prefixed strings.** `vg_work_...`, `vg_tool_...`, `vg_file_...`, `vg_rmix_...`, `vg_voic_...`, `vg_pres_...`, `vg_enti_...`. Store as-is.
5. **Files require hydration for download URLs.** Generated files have signed URLs that expire. Use `getHydratedFile` or `POST /v1/files/{id}/hydrate` to refresh them. See [files-and-resources.md](references/files-and-resources.md).
6. **Webhooks follow the Standard Webhooks spec.** Register an endpoint, verify signatures with `verifyWebhookSignature`. See [webhooks.md](references/webhooks.md).

## Workflows (primary surface)

| Method               | Endpoint                                                              | Description                                              |
| -------------------- | -------------------------------------------------------------------- | ------------------------------------------------------- |
| `scriptToVideo`      | `POST /v1/workflows/script-to-video`                                 | Turn a script (used verbatim) into a narrated video       |
| `voiceoverToVideo`   | `POST /v1/workflows/voiceover-to-video`                              | Build a video from an uploaded voiceover                |
| `slideshowToVideo`   | `POST /v1/workflows/slideshow-to-video`                              | Build a narrated video from a PDF or slideshow          |
| `promptToVideoClip`  | `POST /v1/workflows/prompt-to-video-clip`                            | Generate a short AI video clip from a text prompt       |
| `listWorkflowRuns`   | `GET /v1/workflows/runs`                                             | List workflow runs, most recent first (`selfOnly=true` for key owner only) |
| `getWorkflowRun`     | `GET /v1/workflows/runs/{workflowRunId}`                             | Poll run status; includes `projectId` and optional `projectUrl` |
| `cancelWorkflowRun`  | `POST /v1/workflows/runs/{workflowRunId}/cancel`                     | Request cancellation of an in-progress workflow run     |

## Projects

| Method                   | Endpoint                                             | Description                          |
| ------------------------ | ---------------------------------------------------- | ------------------------------------ |
| `listProjects`           | `GET /v1/projects`                                   | List projects (API-created by default; pass `includeUiProjects=true` for dashboard projects; `selfOnly=true` for key owner only) |
| `getProject`             | `GET /v1/projects/{projectId}`                       | Project metadata; optional `projectUrl` for the app editor |
| `exportProject`          | `POST /v1/projects/{projectId}/export`               | Start an MP4 export → `{ exportId }` |
| `listProjectExports`     | `GET /v1/projects/{projectId}/exports`               | List a project's export ids (newest first) |
| `getProjectExport`       | `GET /v1/projects/{projectId}/exports/{exportId}`    | Poll export; `downloadUrl`/`thumbnailUrl` are 7-day signed URLs (auto re-signed near expiry); `exportFileId` to hydrate fresh URLs |
| `remixProject`           | `POST /v1/projects/{projectId}/remix`                | Apply remix actions → `{ remixActionIds }` |
| `listProjectRemixActions`| `GET /v1/projects/{projectId}/remix-actions`         | Poll remix action status and progress |

## Tools (standalone media generation)

| Method                  | Endpoint                                 | Description                                               |
| ----------------------- | ---------------------------------------- | --------------------------------------------------------- |
| `generateImage`         | `POST /v1/tools/generate-image`          | Generate images from text or image                        |
| `generateVideoClip`     | `POST /v1/tools/generate-video-clip`     | Generate video from text, image, or video                 |
| `textToSpeech`          | `POST /v1/tools/text-to-speech`          | Text to speech voiceover                                  |
| `generateSoundEffect`   | `POST /v1/tools/generate-sound-effect`   | Generate sound effects                                    |
| `generateMusic`         | `POST /v1/tools/generate-music`          | Generate instrumental music from a prompt                 |
| `generateMotionGraphic` | `POST /v1/tools/generate-motion-graphic` | Generate an animated motion graphic (experimental)        |
| `generateAvatar`        | `POST /v1/tools/generate-avatar`         | Avatar presenter video                                    |
| `vectorizeImage`        | `POST /v1/tools/vectorize-image`         | Raster to SVG                                             |
| `removeImageBackground` | `POST /v1/tools/remove-image-background` | Remove image background                                   |
| `removeVideoBackground` | `POST /v1/tools/remove-video-background` | Remove video background                                   |
| `upscaleImage`          | `POST /v1/tools/upscale-image`           | Upscale image resolution                                  |
| `upscaleVideo`          | `POST /v1/tools/upscale-video`           | Upscale video resolution                                  |
| `image3dEffect`         | `POST /v1/tools/image-3d-effect`         | Turn a still image into a short 3D parallax video clip      |
| `listToolExecutions`    | `GET /v1/tools/executions`               | List tool executions, most recent first (`selfOnly=true` for key owner only) |
| `getToolExecutionInfo`  | `GET /v1/tools/executions/{toolExecutionId}` | Poll tool execution status and results              |
| `cancelToolExecution`   | `POST /v1/tools/executions/{toolExecutionId}/cancel` | Cancel an in-progress tool execution          |

Full parameter schemas and examples: [tools.md](references/tools.md)

## Text generation

| Method         | Endpoint                | Description                                                   |
| -------------- | ----------------------- | ------------------------------------------------------------ |
| `generateText` | `POST /v1/text/generate`| Generate text from a prompt with a fast language model (sync)|

Unlike workflows and tools, text generation is **synchronous**: the response includes the generated `text` directly, no polling. Pass `prompt` (required), optional `system` instructions, a `quality` tier (`LOW`, `STANDARD`, `HIGH`, or `MAX`, default `STANDARD`), `temperature`, and `maxOutputTokens` (default 512, max 2000). Useful for drafting scripts, titles, and descriptions before generating a video.

```typescript
const { text } = await vg.text.generateText({
  prompt: "Write a punchy 30-second video script about staying hydrated.",
});
```

```python
response = vg.text.generate_text(
    prompt="Write a punchy 30-second video script about staying hydrated.",
)
# response["text"]
```

## Account

| Method  | Endpoint     | Description                                                              |
| ------- | ------------ | ----------------------------------------------------------------------- |
| `getMe` | `GET /v1/me` | Return the account and team behind the API key (`apiKeyId`, `apiKeyNickname`, `email`, `displayName`, `teamId`). No parameters |

Use `GET /v1/me` as a connection test to confirm a key is valid.

```typescript
const me = await vg.account.getMe();
// me.apiKeyId, me.apiKeyNickname, me.email, me.displayName, me.teamId
```

```python
me = vg.account.get_me()
# me["api_key_id"], me["api_key_nickname"], me["email"], me["display_name"], me["team_id"]
```

## Entities (internal)

These endpoints exist in the OpenAPI spec but are not yet published in the public API reference. They are callable with a standard API key.

Reusable actors, products, and visual styles shared across your team. Attach their reference images to workflows for a consistent character (`ACTOR`) or look (`VISUAL_STYLE`). `PRODUCT` entities are a reusable library of product reference images. Use an entity in a workflow by passing its `vg_enti_...` id: `visualStyle: { type: "ENTITY", entityId }` (script + voiceover), or per-scene `actorEntityIds` / `productEntityIds` / `visualStyleEntityId` on `storyboardToVideo`.

| Method                  | Endpoint                                       | Description                                                          |
| ----------------------- | ---------------------------------------------- | ------------------------------------------------------------------- |
| `listEntities`          | `GET /v1/entities`                             | List entities (filter with `?entityType=ACTOR`, `PRODUCT`, or `VISUAL_STYLE`) |
| `createEntity`          | `POST /v1/entities`                            | Create an entity (`entityType`, `name`, optional `description`)     |
| `getEntity`             | `GET /v1/entities/{entityId}`                  | Get one entity with its reference images                            |
| `updateEntity`          | `POST /v1/entities/{entityId}/update`          | Update `name` or `description`                                      |
| `archiveEntity`         | `POST /v1/entities/{entityId}/archive`         | Archive an entity                                                   |
| `addEntityReference`    | `POST /v1/entities/{entityId}/references`      | Attach an uploaded image as a reference (`fileId`, `description`, `isDefault`) |
| `removeEntityReference` | `POST /v1/entities/{entityId}/references/remove` | Detach a reference image (`fileId`)                               |

Reference images are regular files: upload an image via `POST /v1/files/upload`, then attach its `fileId` to the entity.

## Internal workflows (not in public API reference)

| Method              | Endpoint                               | Description                                              |
| ------------------- | -------------------------------------- | -------------------------------------------------------- |
| `storyboardToVideo` | `POST /v1/workflows/storyboard-to-video` | Build a video from a list of scenes (image or video clip each) |

## SDK helpers

Named exports and matching methods on the client (`vg.pollWorkflowRun(...)` / `vg.poll_workflow_run(...)`). Tools, workflows, and projects also expose `*AndWait` / `*_and_wait` wrappers (start + poll).

| Helper                                             | Purpose                                                    |
| -------------------------------------------------- | ---------------------------------------------------------- |
| `pollWorkflowRun({ client, workflowRunId, ...opts })`    | Poll a workflow run until terminal status                  |
| `pollProjectExport({ client, projectId, exportId })`   | Poll a project export until terminal status                |
| `pollExecutedTool({ client, toolExecutionId, ...opts })`     | Poll a tool execution until terminal status               |
| `pollRemixActions({ client, projectId, ...opts })`       | Poll remix actions until all are terminal                  |
| `uploadFile({ client, data, type?, displayName?, ... })` | Create presigned upload → PUT bytes → poll until processed |
| `downloadFile({ client, fileId, ...opts })`              | Hydrate URL → download (stream to disk or return Response) |
| `getHydratedFile({ client, fileId })`                  | Get file, re-hydrate if URLs expired                       |
| `pollPublicPreview({ client, fileId, ...opts })`         | Poll until public preview / embed playback URLs are ready |
| `createPublicPreview({ client, fileId, ...opts })`       | Enable public preview + poll until ready                   |
| `verifyWebhookSignature({ rawBody, headers, secret })` | Standard Webhooks signature verification; returns parsed event |

## Reference files

- **[tools.md](references/tools.md)**: All tool endpoints with parameters and examples
- **[files-and-resources.md](references/files-and-resources.md)**: File upload, download, hydration, search, public preview, avatar presenters, TTS voices
- **[webhooks.md](references/webhooks.md)**: Webhook CRUD and signature verification
- **[async-patterns.md](references/async-patterns.md)**: Polling, webhook delivery, cancellation

## Need more info?

If the reference files above don't cover what you need, consult the published API documentation:

- **Full docs (human-readable)**: <https://docs.videogen.io>
- **LLM-optimized overview**: <https://docs.videogen.io/llms.txt>
- **Complete docs with SDK examples**: <https://docs.videogen.io/llms-full.txt>
- **OpenAPI spec (JSON)**: <https://docs.videogen.io/openapi.json>
- **Documentation MCP server** (read the API docs from an AI client): <https://docs.videogen.io/_mcp/server>
- **API MCP server** (execute the API from an AI client): hosted remote server (recommended) at `https://mcp.videogen.io/mcp` with an `Authorization: Bearer sk_videogen_live_...` header, or the local stdio server published as `@videogen/mcp` (`npx -y @videogen/mcp` with a `VIDEOGEN_API_KEY` env var). Full tool reference: <https://docs.videogen.io/libraries/mcp>

Use `llms.txt` for a quick scan of all endpoints and guide pages. Use `llms-full.txt` when you need complete page content including SDK code examples. Fetch the OpenAPI spec for exact request/response schemas.
