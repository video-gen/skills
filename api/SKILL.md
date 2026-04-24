---
name: api
description: >-
  Build with the VideoGen API for AI media generation. Use when generating
  images, videos, voiceovers, sound effects, avatars, or managing files and
  webhooks with VideoGen.
license: MIT
compatibility: Requires an API key from app.videogen.io/developers (VIDEOGEN_API_KEY).
metadata: {"openclaw": {"requires": {"env": ["VIDEOGEN_API_KEY"]}, "primaryEnv": "VIDEOGEN_API_KEY"}}
---

# VideoGen API

Generate images, videos, voiceovers, sound effects, and avatar clips programmatically. All tool endpoints are **asynchronous** — POST returns a `toolExecutionId`, poll until `succeeded`/`failed`/`cancelled`.

- **Docs**: <https://videogen.docs.buildwithfern.com>
- **OpenAPI spec**: <https://videogen.docs.buildwithfern.com/openapi.json>
- **Base URL**: `https://api.videogen.io`
- **Auth**: Bearer token (`Authorization: Bearer $VIDEOGEN_API_KEY`)
- **SDKs**: `@videogen/sdk` (npm) · `videogen` (PyPI)

## Quick start

```typescript
import { VideoGenClient, pollExecutedTool } from "@videogen/sdk";

const client = new VideoGenClient({ token: process.env.VIDEOGEN_API_KEY });

const { toolExecutionId } = await client.tools.promptToImage({
  prompt: "A mountain at sunrise, cinematic lighting",
});

const execution = await pollExecutedTool(client, toolExecutionId);
// execution.results[0].storageFileId → "vg_file_..."
```

```python
import os
from videogen import VideoGenApi, poll_executed_tool

client = VideoGenApi(token=os.environ["VIDEOGEN_API_KEY"])

response = client.tools.prompt_to_image(prompt="A mountain at sunrise, cinematic lighting")
execution = poll_executed_tool(client, response.tool_execution_id)
# execution.results[0].storage_file_id → "vg_file_..."
```

## Core concepts

1. **All tool calls are async.** POST returns `{ toolExecutionId }` with HTTP 202. Poll `GET /v1/tools/executions/{id}` or use `pollExecutedTool` until a terminal status. See [async-patterns.md](references/async-patterns.md).
2. **IDs are prefixed strings.** `vg_exec_...`, `vg_file_...`, `vg_voic_...`, `vg_pres_...`. Store as-is.
3. **Files require hydration for download URLs.** Generated files have signed URLs that expire. Use `getHydratedFile` or `POST /v1/files/{id}/hydrate` to refresh them. See [files-and-resources.md](references/files-and-resources.md).
4. **Webhooks follow the Standard Webhooks spec.** Register an endpoint, verify signatures with `verifyWebhookSignature`. See [webhooks.md](references/webhooks.md).

## Available tools

| Method | Endpoint | Description |
|---|---|---|
| `promptToImage` | `POST /v1/tools/prompt-to-image` | Generate images from text |
| `promptToVideoClip` | `POST /v1/tools/prompt-to-video-clip` | Generate video from text |
| `imageToVideoClip` | `POST /v1/tools/image-to-video-clip` | Animate image into video |
| `imageToImage` | `POST /v1/tools/image-to-image` | Transform image with prompt |
| `videoToVideoClip` | `POST /v1/tools/video-to-video-clip` | Restyle video with prompt |
| `textToSpeech` | `POST /v1/tools/text-to-speech` | Text to speech voiceover |
| `promptToSoundEffect` | `POST /v1/tools/prompt-to-sound-effect` | Generate sound effects |
| `audioToAvatarClip` | `POST /v1/tools/audio-to-avatar-clip` | Avatar presenter video |
| `vectorizeImage` | `POST /v1/tools/vectorize-image` | Raster to SVG |
| `removeImageBackground` | `POST /v1/tools/remove-image-background` | Remove image background |
| `removeVideoBackground` | `POST /v1/tools/remove-video-background` | Remove video background |
| `upscaleImage` | `POST /v1/tools/upscale-image` | Upscale image resolution |
| `upscaleVideo` | `POST /v1/tools/upscale-video` | Upscale video resolution |

Full parameter schemas and examples: [tools.md](references/tools.md)

## SDK helpers (not auto-generated)

| Helper | Purpose |
|---|---|
| `pollExecutedTool(client, executionId, opts?)` | Poll until terminal status |
| `uploadFile(client, file, opts)` | Create presigned upload → PUT bytes → poll until processed |
| `downloadFile(client, storageFileId, opts?)` | Hydrate URL → download (stream to disk or return Response) |
| `getHydratedFile(client, storageFileId)` | Get file, re-hydrate if URLs expired |
| `verifyWebhookSignature(rawBody, headers, secret)` | Standard Webhooks signature verification |

## Reference files

- **[tools.md](references/tools.md)** — All 13 tool endpoints with parameters and examples
- **[files-and-resources.md](references/files-and-resources.md)** — File upload, download, hydration, avatar presenters, TTS voices
- **[webhooks.md](references/webhooks.md)** — Webhook CRUD and signature verification
- **[async-patterns.md](references/async-patterns.md)** — Polling, webhook delivery, cancellation
