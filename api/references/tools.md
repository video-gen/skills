# Tool endpoints

All tool endpoints return `202 Accepted` with `{ toolExecutionId }`. Poll `GET /v1/tools/executions/{toolExecutionId}` until the status is `succeeded`, `failed`, or `cancelled`, or use the `pollExecutedTool` SDK helper.

Common optional parameters on most tools:

| Param | Type | Default | Description |
|---|---|---|---|
| `numResults` | integer (1–100) | 1 | Number of output results to generate |
| `isOutputTemporary` | boolean | false | When true, generated files auto-delete after 24 hours |

---

## generateImage

Generate images from a text prompt, or transform an existing image.

**Endpoint:** `POST /v1/tools/generate-image`

| Param | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | yes | Text prompt |
| `aspectRatio` | `{ width, height }` | no | Aspect ratio (default 16:9) |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
const { toolExecutionId } = await client.tools.generateImage({
  prompt: "A futuristic cityscape at dusk",
  aspectRatio: { width: 1, height: 1 },
});
const result = await pollExecutedTool({ client, toolExecutionId });
```

```python
response = client.tools.generate_image(
    prompt="A futuristic cityscape at dusk",
    aspect_ratio={"width": 1, "height": 1},
)
result = poll_executed_tool(client, response.tool_execution_id)
```

---

## generateVideoClip

Generate a video clip from text, an image, or a video, with optional audio.

**Endpoint:** `POST /v1/tools/generate-video-clip`

| Param | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | yes | Text prompt |
| `generateAudio` | boolean | no | When true, audio is guaranteed. When false, audio may still be present (default false) |
| `aspectRatio` | `{ width, height }` | no | Aspect ratio (default 16:9) |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
const { toolExecutionId } = await client.tools.generateVideoClip({
  prompt: "Ocean waves crashing on rocks, slow motion",
  generateAudio: true,
});
const result = await pollExecutedTool({ client, toolExecutionId });
```

---

## textToSpeech

Convert text to speech audio. Only voices with `supportsDirectToolExecution: true` can be used.

**Endpoint:** `POST /v1/tools/text-to-speech`

| Param | Type | Required | Description |
|---|---|---|---|
| `ttsText` | string | yes | Text to synthesize |
| `voiceId` | string | yes | Voice ID from `GET /v1/resources/tts-voices`. Only voices with `supportsDirectToolExecution: true` are accepted |
| `speechLanguageCode` | string | no | ISO-639-1 language hint (e.g. `en`, `es`, `zh`) |
| `voiceSpeed` | number | no | Speech rate multiplier |
| `pronunciationReplacements` | `[{ original, replacement }]` | no | Custom pronunciation overrides |
| `autoExpandPronunciationReplacements` | boolean | no | Auto-expand numbers/symbols to spoken forms |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
const { toolExecutionId } = await client.tools.textToSpeech({
  ttsText: "Welcome to VideoGen. Your video is ready.",
  voiceId: "vg_voic_abc123",
  speechLanguageCode: "en",
  voiceSpeed: 1.1,
});
const result = await pollExecutedTool({ client, toolExecutionId });
```

---

## generateSoundEffect

Generate a sound effect from a text description.

**Endpoint:** `POST /v1/tools/generate-sound-effect`

| Param | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | yes | Description of the sound effect |
| `durationSeconds` | number | no | Target duration in seconds |
| `promptInfluence` | number | no | How strongly the prompt guides generation |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
const { toolExecutionId } = await client.tools.generateSoundEffect({
  prompt: "Thunder crack followed by rain",
  durationSeconds: 5,
});
const result = await pollExecutedTool({ client, toolExecutionId });
```

---

## generateMusic

Generate an instrumental music track from a text description. Output tracks are approximately 30 seconds long.

**Endpoint:** `POST /v1/tools/generate-music`

| Param | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | yes | Description of the music (include genre, mood, instrumentation, tempo) |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
const { toolExecutionId } = await client.tools.generateMusic({
  prompt: "Uplifting cinematic orchestral score with rising strings",
});
const result = await pollExecutedTool({ client, toolExecutionId });
```

---

## generateMotionGraphic

Generate an animated motion graphic video from a text prompt. Experimental and fully agentic: VideoGen plans the animation, optionally generates or fetches supporting media, and renders a self-contained clip. Best for precise text animations (typing effects, kinetic typography, lower thirds, animated captions) that stock or generated footage can't express.

**Endpoint:** `POST /v1/tools/generate-motion-graphic`

| Param | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | yes | Description of the animated motion graphic to generate |
| `fileIds` | string[] | no | Reference media file IDs (`vg_file_...`, up to 8) the motion graphic may display or animate |
| `durationSeconds` | integer | no | Length in seconds, 1–300 (default 5) |
| `aspectRatio` | object | no | `{ width, height }` (default 16:9) |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
const { toolExecutionId } = await client.tools.generateMotionGraphic({
  prompt:
    "A dark terminal window that types out the command `npm run build` character by character, then shows a green success checkmark",
  durationSeconds: 6,
});
const result = await pollExecutedTool({ client, toolExecutionId });
```

---

## generateAvatar

Generate a talking-head avatar video by pairing a presenter with an audio file.

**Endpoint:** `POST /v1/tools/generate-avatar`

| Param | Type | Required | Description |
|---|---|---|---|
| `avatarPresenterId` | string | yes | Presenter ID from `GET /v1/resources/avatar-presenters` |
| `audioStorageFileId` | string | yes | File ID of an AUDIO file (typically from text-to-speech) |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
// Step 1: Generate speech
const { toolExecutionId: ttsExecId } = await client.tools.textToSpeech({
  ttsText: "Hello, welcome to our product demo.",
});
const ttsResponse = await pollExecutedTool({ client, toolExecutionId: ttsExecId });
const audioFileId = ttsResponse.results[0].fileId;

// Step 2: List presenters and pick one
const { avatarPresenters } = await client.resources.listAvatarPresenters();
const presenter = avatarPresenters[0];

// Step 3: Generate avatar video
const { toolExecutionId: avatarExecId } = await client.tools.generateAvatar({
  avatarPresenterId: presenter.avatarPresenterId,
  audioStorageFileId: audioFileId,
});
const avatarResult = await pollExecutedTool({ client, toolExecutionId: avatarExecId });
```

---

## vectorizeImage

Convert a raster image to SVG.

**Endpoint:** `POST /v1/tools/vectorize-image`

| Param | Type | Required | Description |
|---|---|---|---|
| `imageStorageFileId` | string | yes | File ID of the source image |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
const { toolExecutionId } = await client.tools.vectorizeImage({
  imageStorageFileId: "vg_file_abc123",
});
const result = await pollExecutedTool({ client, toolExecutionId });
```

---

## removeImageBackground

Remove the background from an image, returning a transparent-background PNG.

**Endpoint:** `POST /v1/tools/remove-image-background`

| Param | Type | Required | Description |
|---|---|---|---|
| `imageStorageFileId` | string | yes | File ID of the source image |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
const { toolExecutionId } = await client.tools.removeImageBackground({
  imageStorageFileId: "vg_file_abc123",
});
const result = await pollExecutedTool({ client, toolExecutionId });
```

---

## removeVideoBackground

Remove the background from a video, producing a transparent-background video.

**Endpoint:** `POST /v1/tools/remove-video-background`

| Param | Type | Required | Description |
|---|---|---|---|
| `videoStorageFileId` | string | yes | File ID of the source video |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
const { toolExecutionId } = await client.tools.removeVideoBackground({
  videoStorageFileId: "vg_file_vid123",
});
const result = await pollExecutedTool({ client, toolExecutionId });
```

---

## upscaleImage

Increase image resolution while preserving detail.

**Endpoint:** `POST /v1/tools/upscale-image`

| Param | Type | Required | Description |
|---|---|---|---|
| `imageStorageFileId` | string | yes | File ID of the source image |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
const { toolExecutionId } = await client.tools.upscaleImage({
  imageStorageFileId: "vg_file_abc123",
});
const result = await pollExecutedTool({ client, toolExecutionId });
```

---

## upscaleVideo

Increase video resolution while preserving detail.

**Endpoint:** `POST /v1/tools/upscale-video`

| Param | Type | Required | Description |
|---|---|---|---|
| `videoStorageFileId` | string | yes | File ID of the source video |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
const { toolExecutionId } = await client.tools.upscaleVideo({
  videoStorageFileId: "vg_file_vid123",
});
const result = await pollExecutedTool({ client, toolExecutionId });
```

---

## image3dEffect

Turn a still image into a short video clip with a 3D parallax motion effect, simulating camera movement through the scene.

**Endpoint:** `POST /v1/tools/image-3d-effect`

| Param | Type | Required | Description |
|---|---|---|---|
| `imageStorageFileId` | string | yes | File ID of the source image |
| `numResults` | integer | no | Number of results (default 1) |
| `isOutputTemporary` | boolean | no | Auto-delete after 24h (default false) |

```typescript
const { toolExecutionId } = await client.tools.image3dEffect({
  imageStorageFileId: "vg_file_abc123",
});
const result = await pollExecutedTool({ client, toolExecutionId });
```

---

## getToolExecutionInfo (poll)

Retrieve the current status and result of a tool execution.

**Endpoint:** `GET /v1/tools/executions/{toolExecutionId}`

Response shape (`ExecutedTool`):

| Field | Type | Description |
|---|---|---|
| `toolExecutionId` | string | Execution ID |
| `status` | `pending` \| `running` \| `succeeded` \| `failed` \| `cancelled` | Current status |
| `toolType` | string | Tool name (e.g. `GENERATE_IMAGE`) |
| `progressPercentage` | number | Completion progress for the current attempt (0–100). Always `100` when `status` is `succeeded`. |
| `attemptIndex` | integer | Zero-based index of the current or most recent execution attempt |
| `results` | `ToolSuccessResult[]` | Present when `succeeded` — one entry per candidate |
| `error` | `{ message, code? }` | Present when `failed` |

Each `ToolSuccessResult`:

| Field | Type | Description |
|---|---|---|
| `fileId` | string | File ID for the generated asset |
| `type` | `IMAGE` \| `VIDEO` \| `AUDIO` | File type |
| `file` | `StorageFile \| null` | Hydrated file metadata (populated from webhooks or after hydration) |

---

## cancelToolExecution

Cancel an in-progress tool execution.

**Endpoint:** `POST /v1/tools/executions/{toolExecutionId}/cancel`

Returns `202 Accepted`. The execution transitions to `cancelled` if it has not already completed.

```typescript
await client.tools.cancelToolExecution({ toolExecutionId: "vg_tool_abc123" });
```
