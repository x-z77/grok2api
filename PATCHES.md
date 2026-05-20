# PATCHES.md

> Fork patches over upstream `TQZHR/grok2api` to fix Grok image generation
> broken by Grok's WS protocol change (2026-05).

## What broke

Grok migrated image generation to a new WebSocket protocol around 2026-05:
- New endpoint behavior at `wss://grok.com/ws/imagine/listen`
- New payload format (`input_text` not `input_scroll`, extra properties)
- Images delivered inline as base64 `blob` field (not URL)
- Final high-quality image arrives AFTER preview, signaled by
  `current_status: "completed"` for the matching `job_id`

Upstream `chenyme/grok2api` (and this fork) last updated 2026-04, before the
change. Symptom: `/v1/images/generations` returns `HTTP 200` with
`{"url":"error"}` for the legacy method and "Imagine websocket closed before
completed images" for the experimental method.

Chat (`/v1/chat/completions`) and video (model `grok-imagine-1.0-video`) use
the legacy conversation API and remained functional.

## Patches

### 1. `src/grok/imagineExperimental.ts` — payload format

`buildImagineWsPayload` updated to match the current grok.com web UI payload:
- `type: "input_text"` (was `"input_scroll"`)
- Added `enable_side_by_side: true`
- Added `enable_pro: false`

### 2. `src/grok/imagineExperimental.ts` — extract `blob`

`extractUrl` now prefers `msg.blob` (base64 JPEG) over `msg.url` (CDN URL
field, which often points to a 404 placeholder). Returns a
`data:image/jpeg;base64,<blob>` URI.

### 3. `src/grok/imagineExperimental.ts` — wait for `completed`

The message handler no longer treats `type:"image"` frames as completed.
Per the new protocol Grok emits multiple `image` frames per job
(progressively higher quality) and then a `json` frame with
`current_status: "completed"`. We:
- Track the latest blob per `job_id` in `latestByJob`
- Only finalize when the matching completed signal arrives
- Use the latest blob as the final image

### 4. `src/routes/openai.ts` — handle `data:` URIs

`fetchImageAsBase64` short-circuits for `data:` URIs (extracts the base64
inline without attempting an HTTP fetch). Needed because patch #2 returns
data URIs.

## How to use (post-patch)

Image generation requires `response_format: "b64_json"`:

```bash
curl https://<your-worker>.workers.dev/v1/images/generations \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"model":"grok-imagine-1.0","prompt":"...","n":1,"response_format":"b64_json"}'
```

`response_format: "url"` still routes through the `/images/<encoded>` proxy
route, which does not yet handle data URIs. b64_json is the default for the
experimental method anyway, so omitting `response_format` also works.

## Deployment

This fork is connected to Cloudflare Workers Builds:
- Push to `main` → Cloudflare auto-builds → auto-deploys
- Worker: `grok2api.<account>.workers.dev`
- D1 (`grok2api`) and KV (`grok2api-cache`) bindings preserved across deploys

## Syncing upstream

Forks do NOT auto-sync. To pull upstream changes:

1. Visit `https://github.com/x-z77/grok2api` → click "Sync fork"
2. Resolve any merge conflicts (likely in
   `src/grok/imagineExperimental.ts` and `src/routes/openai.ts`)
3. Verify all four patches are still present:
   - Payload `input_text` + `enable_side_by_side` + `enable_pro`
   - `extractUrl` checks `blob` first
   - `isCompleted` does NOT auto-complete on `type:"image"`
   - Message handler uses `latestByJob` keyed on `job_id`
   - `fetchImageAsBase64` handles `data:` URI

If upstream has merged equivalent fixes, drop these patches and use theirs.

## Verifying after a sync

```bash
# Reset token state in D1 console (Cloudflare dashboard):
UPDATE tokens SET failed_count = 0, cooldown_until = NULL, last_failure_reason = NULL, status = 'active';

# Then test:
curl -X POST https://<worker>/v1/images/generations \
  -H "Authorization: Bearer <KEY>" -H "Content-Type: application/json" \
  -d '{"model":"grok-imagine-1.0","prompt":"red strawberry","n":1,"response_format":"b64_json"}' \
  | jq '.data[0].b64_json' | head -c 50
```

A response > 100KB with a base64 string starting with `/9j/4` indicates the
patches work.

## Known limitations

- `response_format: "url"` not yet supported for blob-based image generation.
- Image edit endpoint works without patches.
- Video generation works without patches.

## Patch commit refs

- `f46945c` — prefer blob over url in extractUrl
- `1fc86df` — fetchImageAsBase64 handles data: URI
- `703f69c` — wait for completed status + latestByJob tracking
- (Earlier `d95719c` superseded by above; first attempt at protocol fix)
