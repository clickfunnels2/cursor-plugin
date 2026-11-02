---
name: clickfunnels-media
description: >
  Manage ClickFunnels images, import videos, and find downloadable files. Use when
  asked to upload hosted course videos, poll video processing, attach lesson media,
  upload images, or discover downloads for products.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: https://accounts.myclickfunnels.com/llms.txt
    description: Integration and authentication overview.
  - path: https://accounts.myclickfunnels.com/.well-known/courses/skill.md
    description: Create courses and attach ready videos to lessons.
---

# ClickFunnels Media

Upload images by URL, import hosted videos in the background, and find existing
downloadable files. Use the returned IDs to attach media to your content.

## When to use this skill

- Move a hosted course library into ClickFunnels and attach its videos to lessons.
- Import an image and use its ID as a video poster or lesson image.
- Find a download's `asset_id` to grant file access through a product variant.

## Authentication

Use a workspace API token or OAuth token as `Authorization: Bearer <token>` with
read access for GET and write access for mutations. Tokens restricted by category
need `media:read` or `media:write`, and lesson updates also need `courses:write`.
Use the API base URL for the workspace's cluster. See the authentication links
in [/llms.txt](https://accounts.myclickfunnels.com/llms.txt). Never send an MCP token directly to the REST API.

For MCP, call `describe_category` with `media`. The `image` resource has CRUD,
`video` has asynchronous create plus list/fetch/update/destroy, and `download`
has list/fetch. Pass `workspace_id` on every data call. Use the operations and
attributes returned by discovery.

## Endpoint reference

| Action | Method | Path |
|---|---|---|
| List images | GET | `/api/v2/workspaces/:workspace_id/images` |
| Upload image by URL | POST | `/api/v2/workspaces/:workspace_id/images` |
| Fetch/update/delete image | GET / PATCH / DELETE | `/api/v2/images/:id` |
| List videos | GET | `/api/v2/workspaces/:workspace_id/videos` |
| Import video by URL | POST | `/api/v2/workspaces/:workspace_id/videos` |
| Poll/update/delete video | GET / PATCH / DELETE | `/api/v2/videos/:id` |
| List downloads | GET | `/api/v2/workspaces/:workspace_id/downloads` |
| Fetch download | GET | `/api/v2/downloads/:id` |

Lists return bare arrays. Follow `Pagination-Next` as the next request's `after`
cursor. Video lists accept `filter[status]`, including comma-separated states.
IDs in paths and media attachment fields accept numeric IDs or public IDs.

## Import and poll a video

```http
POST /api/v2/workspaces/123/videos
Content-Type: application/json
Authorization: Bearer <token>
```

```json
{
  "video": {
    "name": "Module 1: Welcome",
    "upload_source_url": "https://media.example.com/welcome.mp4",
    "poster_image_id": null
  }
}
```

The response is **201 Created**, with `status: "queued"`. The request does not
wait for the download. `name` is optional and defaults to the downloaded filename
without its extension. `poster_image_id` is an optional image in this workspace.
Identical URLs create separate videos.

```json
{
  "id": 789,
  "public_id": "AbCdEfGh",
  "workspace_id": 123,
  "name": "Module 1: Welcome",
  "status": "queued",
  "error_code": null,
  "metadata": null,
  "poster_image_id": null,
  "url": null,
  "thumbnail_url": null,
  "created_at": "2026-10-20T17:00:00.000Z",
  "updated_at": "2026-10-20T17:00:00.000Z"
}
```

Poll `GET /api/v2/videos/789` until `status` is `ready` or `failed`. An import
stays `queued` while it downloads and prepares the file; `processing` appears only
for videos uploaded in the dashboard. Once the video has a processed file it returns
`metadata` (filename, detected MIME type, duration in seconds, width, height), a
signed MP4 `url`, and a signed generated-frame `thumbnail_url`. The `url` expires after one hour and the
`thumbnail_url` after one day; fetch the video again to renew them. No HLS URL is
exposed.

Import accepts MP4, AVI, and MPEG video bodies up to 3 GB. MIME is detected from
the downloaded bytes. MOV is not supported. Redirects are followed with checks at
each hop. Use a direct download URL rather than a watch-page URL.

At most five imports per workspace may be queued or processing. The next import
returns **429** with `Retry-After: 60` and
`{"error":"Too many video imports in progress"}`. Wait before trying again.

## Attach a ready video to a lesson

```http
PATCH /api/v2/courses/lessons/42
```

```json
{"courses_lesson":{"video_id":789,"image_id":null}}
```

The response includes `video` and `image` objects (`id`, `public_id`, `name`,
`url`), or null when detached.
These fields work on lesson create and update; null detaches. The video must be
ready and belong to the lesson's workspace. This does not change lesson page
content. See the [Courses skill](https://accounts.myclickfunnels.com/.well-known/courses/skill.md).

## Update, retry, and remove videos

Only ready or failed videos accept PATCH. Send `name`, `poster_image_id`, or a
new `upload_source_url`. A new URL re-imports into the same record, resets status
to queued and clears `error_code`; existing lessons keep referencing its ID and
keep playing the previous file until the new one is ready. A failed re-import
keeps the previous file. Deleting a video nulls those lesson references.

Subscribe to V2 `video.processed` and `video.failed` webhooks to learn when an
import finishes. Each terminal status transition emits one event whose `data`
has the same shape as GET. There are no video CRUD webhook topics.

## Images and downloads

Create an image with `{"image":{"name":"Cover","upload_source_url":"https://media.example.com/cover.png"}}`.
Image update also accepts `alt_text`. The `url` in the create or update response is
the image's permanent URL. The file is processed in the background, so for a few
seconds that URL can answer 404 (new image) or the previous file (replaced image). Use an image's `id` or `public_id` in
`poster_image_id` or a lesson's `image_id`.

Downloads are read-only here. List or fetch an existing download and use its
`asset_id` in a product variant's `asset_ids` to grant download access.

## Worked example

1. Import the course's videos, keeping no more than five pending in this workspace.
2. Poll each returned ID, or wait for its processed/failed webhook.
3. For each ready video, PATCH the matching lesson with `video_id`.
4. Read the lesson back and verify `video_id`. Renew signed URLs by fetching the
   video whenever playback URLs expire.

## Common errors

| Error | Cause | Fix |
|---|---|---|
| 401 | Missing or invalid token | Send the appropriate REST or MCP credential. |
| 404 | Unknown resource or inaccessible workspace | Check the ID and workspace grant. |
| 422 `video_id must reference a ready video` | Pending, failed, missing, or foreign video | Poll until ready and use a video in this workspace. |
| 422 on video update | Video is queued or processing | Wait for ready or failed. |
| 429 | Five pending imports | Respect `Retry-After: 60`. |
| `download_failed` | Source could not be downloaded | Supply an accessible direct download URL. Details are intentionally not exposed. |
| `file_too_large` | Download exceeds the streaming limit | Supply a video under 3 GB. |
| `invalid_file_type` | Body is not an MP4, AVI, or MPEG video (MOV included) | Supply MP4, AVI, or MPEG. |
| `processing_failed` | Metadata or thumbnail preparation failed, or the import stalled for 3 hours | Check the video file and re-import. |
