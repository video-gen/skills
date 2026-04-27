# Files and resources

## Files

Generated assets and user uploads are represented as `StorageFile` objects. Signed download URLs expire — call the hydrate endpoint or use the `getHydratedFile` helper to refresh them.

### StorageFile shape

| Field | Type | Description |
|---|---|---|
| `fileId` | string | File ID (`vg_file_...`) |
| `type` | `IMAGE` \| `VIDEO` \| `AUDIO` | File type |
| `scope` | `GLOBAL` \| `PROJECT` \| `EXPORT` \| `TEMPORARY` | File scope |
| `displayName` | string | Display name |
| `description` | string \| null | Description |
| `durationSeconds` | number \| null | Duration for video/audio, null for images |
| `transcript` | string \| null | Transcript for video/audio when available |
| `thumbnailSource` | FileSource \| null | Thumbnail image |
| `previewSource` | FileSource \| null | Preview rendition (720p for video, resized for images) |
| `downloadSource` | FileSource \| null | Highest-quality downloadable rendition |

Each `FileSource`:

| Field | Type | Description |
|---|---|---|
| `status` | `pending` \| `ready` \| `failed` \| `skipped` | Rendition status |
| `url` | string \| null | Signed URL (present when `ready` and hydrated) |
| `expiresAt` | number \| null | Unix timestamp when the URL expires |
| `width` | integer \| null | Width in pixels |
| `height` | integer \| null | Height in pixels |
| `fileBytes` | integer \| null | File size in bytes |

### File scopes

| Scope | Meaning |
|---|---|
| `GLOBAL` | User-uploaded or standalone generated files — persists indefinitely |
| `PROJECT` | Project-specific files (e.g. TTS clips in a generated project) |
| `EXPORT` | Project exports |
| `TEMPORARY` | Short-lived files, automatically deleted after 24 hours |

---

## List files

**Endpoint:** `GET /v1/files`

Returns `{ files: StorageFile[] }`.

```typescript
const { files } = await client.files.getFiles();
```

---

## Get file

**Endpoint:** `GET /v1/files/{fileId}`

Returns a single `StorageFile`.

```typescript
const file = await client.files.getFile({ fileId: "vg_file_abc123" });
```

---

## Hydrate file

**Endpoint:** `POST /v1/files/{fileId}/hydrate`

Generates fresh signed URLs for all renditions. Returns the full `StorageFile` with populated sources.

```typescript
const file = await client.files.hydrateFile({ fileId: "vg_file_abc123" });
console.log(file.downloadSource?.url); // fresh signed URL
```

### SDK helper: getHydratedFile

Fetches a file and automatically hydrates it if source URLs are missing or expired.

```typescript
import { getHydratedFile } from "@videogen/sdk";

const file = await getHydratedFile(client, "vg_file_abc123");
```

---

## Upload a file

### Raw API flow

1. `POST /v1/files/upload` — creates a pending file and returns `{ fileId, uploadUrl }`
2. `PUT` the raw file bytes to `uploadUrl`
3. Poll `GET /v1/files/{fileId}` until a source has `status: "ready"`

**Create file upload request:**

| Param | Type | Required | Description |
|---|---|---|---|
| `type` | `IMAGE` \| `VIDEO` \| `AUDIO` | yes | File type |
| `displayName` | string | yes | Display name |
| `isTemporary` | boolean | no | Guaranteed available for 24 hours, then may be archived. Not analyzed (no description, transcript, or embedding). Default false. |

### SDK helper: uploadFile

Wraps the full flow — creates the upload, PUTs the bytes, polls until processed, returns hydrated file.

```typescript
import { uploadFile } from "@videogen/sdk";
import { readFileSync } from "node:fs";

const buffer = readFileSync("./photo.jpg");

const file = await uploadFile(client, buffer, {
  type: "IMAGE",
  displayName: "photo.jpg",
});

// file.fileId → "vg_file_..."
// file.downloadSource.url → signed download URL
```

Options:

| Option | Type | Default | Description |
|---|---|---|---|
| `type` | `IMAGE` \| `VIDEO` \| `AUDIO` | — | Required file type |
| `displayName` | string | — | Required display name |
| `temporary` | boolean | false | Guaranteed available for 24 hours, then may be archived. Not analyzed. |
| `pollIntervalMs` | number | 2000 | Poll interval while waiting for processing |
| `timeoutMs` | number | 3,600,000 | Max wait time before throwing |
| `signal` | AbortSignal | — | Abort signal |

---

## Download a file

### SDK helper: downloadFile

Hydrates the file and downloads it. If `outputPath` is provided, streams to disk. Otherwise returns the raw `Response`.

```typescript
import { downloadFile } from "@videogen/sdk";

// Stream to disk
await downloadFile(client, "vg_file_abc123", { outputPath: "./output.mp4" });

// Or get the Response for custom handling
const response = await downloadFile(client, "vg_file_abc123");
const bytes = await response.arrayBuffer();
```

---

## Resources

### List avatar presenters

**Endpoint:** `GET /v1/resources/avatar-presenters`

Returns `{ avatarPresenters: AvatarPresenter[] }`.

| Field | Type | Description |
|---|---|---|
| `avatarPresenterId` | string | Presenter ID (`vg_pres_...`) — pass to `audioToAvatarClip` |
| `displayableGender` | `MALE` \| `FEMALE` \| `NEUTRAL` | Gender |
| `imageUrl` | string | Still image URL |
| `thumbnailUrl` | string | Thumbnail URL |
| `previewVideoUrl` | string | Short preview clip URL |

```typescript
const { avatarPresenters } = await client.resources.listAvatarPresenters();
```

### List TTS voices

**Endpoint:** `GET /v1/resources/tts-voices`

Returns `{ ttsVoices: TtsVoice[] }`. Optional query param `includeDeprecatedVoices` (default false).

| Field | Type | Description |
|---|---|---|
| `voiceId` | string | Voice ID (`vg_voic_...`) — pass to `textToSpeech` |
| `languageCode` | string | Locale tag (e.g. `en-US`, `es-ES`) |
| `displayName` | string | Voice name |
| `displayGender` | `MALE` \| `FEMALE` \| `NEUTRAL` | Gender |
| `accent` | string \| null | Accent (e.g. `american`, `british`) |
| `description` | string \| null | Voice description |
| `supportsDirectToolExecution` | boolean | Must be `true` for `textToSpeech` endpoint |
| `supportsAllLanguages` | boolean | When true, can synthesize any language |
| `isDeprecated` | boolean | Prefer non-deprecated voices for new integrations |

```typescript
const { ttsVoices } = await client.resources.listTtsVoices();
const usableVoices = ttsVoices.filter((v) => v.supportsDirectToolExecution);
```
