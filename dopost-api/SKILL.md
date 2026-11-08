---
name: dopost-api
description: Use the Dopost REST API to save drafts, publish or schedule social posts individually or in groups, upload media, manage connected accounts, and retrieve social analytics. Use when the task requires Dopost API requests from an agent with HTTP or terminal access; prefer connected MCP tools when available.
metadata:
  openclaw:
    version: 1.4.0
    homepage: https://dopost.co/docs/api
    requires:
      env:
        - DOPOST_API_KEY
    primaryEnv: DOPOST_API_KEY
---

# Dopost API

Dopost is a social media management platform. Its REST API lets you publish and schedule posts, manage media, and inspect connected social accounts.

**Base URL:** `https://dopost.co/api/v1`
**Auth:** All requests require the header `x-api-key: <your-key>`
**Full docs:** https://dopost.co/docs/api

---

## Setup

Use `DOPOST_API_KEY` from the environment or an existing `.env` / `.env.local` file without displaying it. If missing, ask the user to configure it locally. Create the key in dopost under **Settings → API keys** (https://dopost.co/dashboard/settings/api-keys), select the workspace and required permissions, and copy it when revealed. API and MCP access requires a Creator, Pro or Agency workspace.

```bash
export DOPOST_API_KEY="dpk_live_..."
```

---

## Endpoints

### Social Accounts

#### List connected accounts
```
GET /api/v1/social/accounts
Scope: social:accounts
```
```bash
curl -H "x-api-key: $DOPOST_API_KEY" https://dopost.co/api/v1/social/accounts
```
Returns `{ accounts: [{ id, platform, platformUsername, platformUserId }] }`.
The `id` field is the `accountId` needed for publishing.

#### Get platform limits
```
GET /api/v1/social/limits/:platform
Scope: social:limits
```
Platforms: `x`, `instagram`, `instagram_direct`, `facebook`, `linkedin`, `linkedin_organization`, `tiktok`, `youtube`, `threads`, `bluesky`, `mastodon`, `pinterest`, `google-business`

> `instagram` = connected via Meta Business (Facebook Page linked). `instagram_direct` = connected directly via Instagram OAuth.
> `linkedin` = personal profile. `linkedin_organization` = company page.

```bash
curl -H "x-api-key: $DOPOST_API_KEY" https://dopost.co/api/v1/social/limits/instagram
```

---

### Posts

#### Publish, schedule, or save a draft
```
POST /api/v1/post/publish
Scope: posts:create
```
```json
{
  "accountId": "<accountId>",
  "text": "Post content",
  "media": ["https://..."],
  "publishAt": "2025-12-25T12:00:00Z",
  "platformOptions": {}
}
```
- Use `accountId` for one account or `posts` for a group; never send both
- At least `text` or `media` is required, except for drafts and Google Business events and offers with the required event fields
- `media` can be an array of URL strings or objects `{ url: "...", mediaId: "..." }`; use a returned ready media ID only with its matching URL
- Maximum 10 media items per post
- Omit `publishAt` (or set to `null`) to publish immediately unless `draft: true`
- Publishing/scheduling returns `202` with `{ success, jobId, postId, status, warnings }`
- Publishing is **asynchronous** — poll `GET /api/v1/post/:postId` to check status

Before publishing or scheduling, call `GET /api/v1/social/limits/:platform` for each distinct destination platform. Check text, media count and the chosen post type's requirements, including per-account overrides and thread entries. Use only verified file metadata; never infer dimensions, duration or codecs from a filename or URL. Explain known violations with the affected field, actual value and allowed limit. The server inspects received files and runs the same platform validation as the composer before creating posts; limits alone do not inspect an attachment or guarantee publication.

##### Save a draft

```json
{
  "accountId": "<accountId>",
  "draft": true,
  "text": "Work in progress"
}
```

Set `draft: true` and omit `publishAt`. This returns `201` with `status: "draft"`, `postId` and no `jobId`; Get Post returns `DRAFT`. Drafts create no publishing schedule or monthly publication charge and can be edited or published in the dopost dashboard. They may have incomplete text/media or exceed platform publishing limits, but attached files must still pass upload size, format and integrity checks. A non-null `publishAt` with `draft: true` returns `400`.

##### Create a post group

```json
{
  "text": "Shared caption",
  "media": ["https://example.com/photo.jpg"],
  "draft": true,
  "posts": [
    { "accountId": "<instagramAccountId>" },
    { "accountId": "<facebookAccountId>", "text": "Facebook caption" }
  ]
}
```

`posts` accepts 1–20 distinct active accounts in one authorized workspace. Each item requires `accountId` and inherits top-level `text`, `media`, `platformOptions`, `replyTo` and `thread`. An item field replaces the entire default field: objects and arrays are not merged; `media: []` clears shared media. Put `draft`, `publishAt` and `timezone` only at the top level, where they apply to the whole group.

Shared media URLs are imported and inspected once per request, then referenced by each account. Different media URLs are prepared separately. Every account is authorized and its publishing content validated before any post is created, with the draft exceptions above. A group validation failure rejects the whole request and identifies `accountId`, zero-based `postIndex` and the affected field.

The response contains `groupId` and `posts[]` with each `accountId` and `postId`. Draft groups return `201`, `status: "draft"` and no `jobId`; publishing/scheduled groups return `202`. Poll each `postId`, not `groupId`: after acceptance, delivery outcomes are independent per social network. Publishing counts one publication per account. Deleting a member deletes the whole group; rescheduling by `postId` changes only that member.

##### Platform-specific options (`platformOptions`)

**Instagram**
| Field | Values | Default |
|-------|--------|---------|
| `postType` | `"feed"` \| `"story"` \| `"reel"` | `"feed"` (auto `"carousel"` with multiple media) |

**TikTok**
| Field | Values | Default |
|-------|--------|---------|
| `privacyLevel` | `"PUBLIC_TO_EVERYONE"` \| `"MUTUAL_FOLLOW_FRIENDS"` \| `"FOLLOWER_OF_CREATOR"` \| `"SELF_ONLY"` | `"PUBLIC_TO_EVERYONE"` |
| `disableDuet` | `boolean` | `false` |
| `disableStitch` | `boolean` | `false` |
| `disableComment` | `boolean` | `false` |

**YouTube**
| Field | Values | Default |
|-------|--------|---------|
| `videoType` | `"video"` \| `"short"` | `"video"` |
| `title` | `string` | First 100 chars of `text` |
| `privacyStatus` | `"public"` \| `"private"` \| `"unlisted"` | `"public"` |

**Pinterest**
| Field | Description |
|-------|-------------|
| `board` | Board ID (required for Pinterest) |

#### Get a post
```
GET /api/v1/post/:postId
Scope: posts:read
```
Returns the full post object including publish status:
```json
{
  "id": "cm3abc123def456",
  "status": "PUBLISHED",
  "platform": "INSTAGRAM",
  "text": "Post content",
  "media": [],
  "postUrl": "https://instagram.com/p/...",
  "account": {
    "id": "cm3xyz789ghi012",
    "platform": "INSTAGRAM",
    "platformUsername": "myaccount"
  },
  "schedule": {
    "scheduledFor": "2026-04-13T09:00:00.000Z",
    "status": "PUBLISHED",
    "publishedAt": "2026-04-13T09:00:05.000Z"
  },
  "jobId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "source": "api",
  "createdAt": "2026-04-07T10:00:00.000Z",
  "updatedAt": "2026-04-07T10:32:18.000Z"
}
```
`schedule` is `null` for immediate posts. `postUrl` is `null` until published.
Use this endpoint to poll status after publishing.

#### List posts
```
GET /api/v1/post?status=PENDING&limit=20&cursor=<cursor>
Scope: posts:read
```
Status filter values: `DRAFT`, `PENDING`, `PUBLISHED`, `FAILED`

#### Reschedule a post
```
PATCH /api/v1/post/:postId
Scope: posts:reschedule
Body: { "publishAt": "2025-12-31T09:00:00Z" }
```
Only works on posts with status `PENDING`. The new date must be in the future.
Returns `{ id, scheduledFor }`.

#### Delete a post
```
DELETE /api/v1/post/delete/:postId
Scope: posts:delete
```
Cancels scheduling automatically if the post is `PENDING`.
If the post belongs to a group, deletion affects the entire group. A published or processing member blocks deletion. Confirm the group scope before deleting.
Returns `{ success: true, deletedPostId: "..." }`.

---

### Media

#### Upload media (presigned URL flow)
```
POST /api/v1/media
Scope: media:upload
Body: { "fileName": "photo.jpg", "contentType": "image/jpeg", "sizeInBytes": 204800, "purpose": "post" }
```
Returns `201` with:
```json
{
  "id": "cm3media001xyz",
  "uploadUrl": "https://storage.example.com/...?X-Amz-Signature=...",
  "publicUrl": "https://cdn.dopost.co/media/.../photo.jpg",
  "fileName": "photo.jpg",
  "contentType": "image/jpeg",
  "expiresIn": 3600
}
```

- `uploadUrl` is a temporary presigned URL — upload the file with a PUT request within `expiresIn` seconds
- `publicUrl` can be used in posts only after successful finalization
- `purpose: "library"` is the default: up to 1 GB per file, subject to available Media storage; the ready file appears in the library
- `purpose: "post"` prepares an attachment outside the Media library and its storage quota: 500 MB on paid plans, 50 MB on Free; platform limits can be smaller
- Accepted file types: JPEG, PNG, GIF, MP4 and MOV; the declared MIME must match the actual bytes
- After the PUT, call `/finalize` (see below) and wait for `READY` before attaching the file

```bash
curl -X PUT -H "Content-Type: image/jpeg" --data-binary @photo.jpg "$UPLOAD_URL"
```

#### Finalize media upload
```
POST /api/v1/media/{id}/finalize
Scope: media:upload
```
Must be called after the PUT. Checks actual file size, format and integrity, decodes videos completely, extracts metadata and a thumbnail, and marks the record as `READY`. Only `purpose: "library"` files appear in the library. Repeating finalization on a ready file is safe. Successful upload/finalization confirms file validity, not compatibility with every publishing destination; platform rules run when publishing or scheduling.

Invalid or oversized direct uploads return `415` or `413`, are marked `FAILED`, and their stored objects are removed. Inspection-service outages/timeouts return `503` with `retryable: true`; the direct upload stays pending so retry `/finalize` with the same media ID instead of uploading the file again. See Error handling below for codes and actions.

```bash
curl -X POST -H "x-api-key: $DOPOST_API_KEY" "https://dopost.co/api/v1/media/$MEDIA_ID/finalize"
```

#### Attach an external URL to a post

Pass a public HTTPS URL directly in `media` when the file is already accessible online. Dopost imports it into its storage as a post-only attachment before accepting the post, including draft and thread attachments. The source must remain accessible until import completes; delivery uses the stored copy afterward. A ready dopost file in the destination workspace is reused. Failed imports reject the request before a post is created and clean up the new staged copy. Do not delete an existing library file because it fails a destination's publishing rules.

#### List media
```
GET /api/v1/media?limit=20&cursor=<cursor>
Scope: media:list
```

#### Delete media
```
DELETE /api/v1/media/:mediaId
Scope: media:delete
```
Returns `{ message: "Media deleted", mediaId: "..." }`.

---

## Common workflows

### Publish a post now
1. `GET /api/v1/social/accounts` — pick `accountId`
2. `GET /api/v1/social/limits/:platform` — check text and the selected format
3. `POST /api/v1/post/publish` with `accountId` + `text`
4. `GET /api/v1/post/:postId` — poll until `status` is `PUBLISHED` or `FAILED`

### Publish a post with an image
1. Check destination limits, then `POST /api/v1/media` with `purpose: "post"` — get `id`, `uploadUrl` + `publicUrl`
2. PUT the file to `uploadUrl`
3. `POST /api/v1/media/{id}/finalize` — confirm upload and get metadata
4. `POST /api/v1/post/publish` with `media: [publicUrl]`
5. `GET /api/v1/post/:postId` — poll until published

### Schedule and reschedule
1. Publish with a future `publishAt`
2. To change the date: `PATCH /api/v1/post/:postId` with new `publishAt`
3. To cancel: `DELETE /api/v1/post/delete/:postId`

---

## Error handling

| Status | Meaning |
|--------|---------|
| `400` | Bad request — check the request body |
| `401` | Invalid or missing `x-api-key` |
| `403` | API key lacks the required scope |
| `404` | Resource not found or inactive account |
| `409` | Media is unavailable, not ready, or changed; check `MEDIA_NOT_READY` / `MEDIA_CHANGED` and the indicated recovery action |
| `413` | Upload or platform file-size limit exceeded; reduce the file size to the returned limit |
| `415` | File is unreadable, corrupt, unsupported, or has a mismatched declared MIME |
| `422` | Platform validation or permanent source error; inspect `errorCode`, `issues` and the affected field |
| `429` | Rate limit or monthly quota exceeded. Check `X-RateLimit-*` headers |
| `503` | Temporary source/storage/inspection failure; inspect `retryable` and retry the failed operation when appropriate |
| `504` | Media download timed out; retry or use a faster source URL |

On `429`, respect the `Retry-After` header before retrying.

Report the API's readable `error` and `errorCode`, including `fileName`, `field`, `stage`, `retryable`, `actual` and `expected` when supplied. For `POST_VALIDATION_FAILED`, explain each blocking `issues[]` entry using its received value and allowed limit; size violations use `413`, other platform violations use `422`. `warnings` are recommendations and do not block acceptance. Group errors identify the account and `postIndex`; thread/file indexes are zero-based.

For `MEDIA_FILE_CORRUPT` / `MEDIA_UNREADABLE`, ask for a newly exported readable file. For `MEDIA_FORMAT_UNSUPPORTED`, request a supported format; for `MEDIA_TYPE_MISMATCH`, correct the declared type or file. `MEDIA_INSPECTION_TIMEOUT` / `MEDIA_INSPECTION_UNAVAILABLE` are service failures, not proof that the file is bad. Retry direct finalization on the same ID when `retryable: true`. A successful finalization still does not bypass platform validation.

Source failures expose `MEDIA_SOURCE_DNS_ERROR`, `MEDIA_SOURCE_HTTP_ERROR`, `MEDIA_SOURCE_TLS_ERROR`, `MEDIA_SOURCE_UNAVAILABLE` or `MEDIA_DOWNLOAD_TIMEOUT`, with `reasonCode` / `upstreamStatus` when known. Storage failures use `MEDIA_STORAGE_UNAVAILABLE`. Follow the returned `retryable` value; `null` means unknown. Never invent a cause or retry unchanged input marked `retryable: false`.

After acceptance, poll the returned post ID rather than resubmitting it. If a post-creation response is lost, check recent posts before retrying to avoid creating a duplicate. A failed status read does not mean that publication failed.

---

## Examples

### 1. List connected accounts

**Request**
```bash
curl -H "x-api-key: $DOPOST_API_KEY" \
  https://dopost.co/api/v1/social/accounts
```

**Response `200 OK`**
```json
{
  "accounts": [
    {
      "id": "cm3xyz789ghi012",
      "platform": "INSTAGRAM",
      "platformUsername": "myaccount",
      "platformUserId": "17841400000000001"
    },
    {
      "id": "cm3abc456def789",
      "platform": "LINKEDIN",
      "platformUsername": "johndoe",
      "platformUserId": "urn:li:person:a1B2c3D4e5"
    }
  ]
}
```

---

### 2. Publish a text post immediately

**Request**
```bash
curl -X POST \
  -H "x-api-key: $DOPOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "accountId": "cm3xyz789ghi012",
    "text": "Hello from the Dopost API!"
  }' \
  https://dopost.co/api/v1/post/publish
```

**Response `202 Accepted`**
```json
{
  "success": true,
  "jobId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "postId": "cm3abc123def456",
  "status": "processing"
}
```

---

### 3. Check post status

**Request**
```bash
curl -H "x-api-key: $DOPOST_API_KEY" \
  https://dopost.co/api/v1/post/cm3abc123def456
```

**Response — published**
```json
{
  "id": "cm3abc123def456",
  "status": "PUBLISHED",
  "platform": "INSTAGRAM",
  "text": "Hello from the Dopost API!",
  "media": [],
  "postUrl": "https://www.instagram.com/p/ABC123/",
  "account": {
    "id": "cm3xyz789ghi012",
    "platform": "INSTAGRAM",
    "platformUsername": "myaccount"
  },
  "schedule": null,
  "jobId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "source": "api",
  "createdAt": "2026-04-07T10:00:00.000Z",
  "updatedAt": "2026-04-07T10:32:18.000Z"
}
```

**Response — failed**
```json
{
  "id": "cm3abc123def456",
  "status": "FAILED",
  "platform": "INSTAGRAM",
  "text": "Hello from the Dopost API!",
  "media": [],
  "postUrl": null,
  "account": {
    "id": "cm3xyz789ghi012",
    "platform": "INSTAGRAM",
    "platformUsername": "myaccount"
  },
  "schedule": null,
  "jobId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "source": "api",
  "createdAt": "2026-04-07T10:00:00.000Z",
  "updatedAt": "2026-04-07T10:05:00.000Z"
}
```

---

### 4. Publish an Instagram Reel

**Request**
```bash
curl -X POST \
  -H "x-api-key: $DOPOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "accountId": "cm3xyz789ghi012",
    "text": "New reel! #content",
    "media": ["https://cdn.example.com/video.mp4"],
    "platformOptions": {
      "postType": "reel"
    }
  }' \
  https://dopost.co/api/v1/post/publish
```

**Response `202 Accepted`**
```json
{
  "success": true,
  "jobId": "a1b2c3d4-0000-4abc-8def-111122223333",
  "postId": "cm3reel001xyz",
  "status": "processing"
}
```

---

### 5. Schedule a post

**Request**
```bash
curl -X POST \
  -H "x-api-key: $DOPOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "accountId": "cm3abc456def789",
    "text": "Scheduled post for next Monday!",
    "publishAt": "2026-04-13T09:00:00Z"
  }' \
  https://dopost.co/api/v1/post/publish
```

**Response `202 Accepted`**
```json
{
  "success": true,
  "jobId": "b2c3d4e5-1111-4bcd-9ef0-222233334444",
  "postId": "cm3sched001abc",
  "status": "scheduled"
}
```

---

### 6. Upload media and publish with it

**Step 1 — Request presigned URL**
```bash
curl -X POST \
  -H "x-api-key: $DOPOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "fileName": "photo.jpg",
    "contentType": "image/jpeg",
    "sizeInBytes": 204800,
    "purpose": "post"
  }' \
  https://dopost.co/api/v1/media
```

**Response `201 Created`**
```json
{
  "id": "cm3media001xyz",
  "uploadUrl": "https://storage.example.com/uploads/photo.jpg?X-Amz-Signature=...",
  "publicUrl": "https://cdn.dopost.co/media/cm3media001xyz/photo.jpg",
  "fileName": "photo.jpg",
  "contentType": "image/jpeg",
  "expiresIn": 3600
}
```

**Step 2 — Upload the file**
```bash
curl -X PUT \
  -H "Content-Type: image/jpeg" \
  --data-binary @photo.jpg \
  "https://storage.example.com/uploads/photo.jpg?X-Amz-Signature=..."
```
Returns `200` with empty body on success.

**Step 3 — Finalize and wait for READY**
```bash
curl -X POST \
  -H "x-api-key: $DOPOST_API_KEY" \
  https://dopost.co/api/v1/media/cm3media001xyz/finalize
```
Continue only after successful finalization returns `status: "READY"`.

**Step 4 — Publish using the public URL**
```bash
curl -X POST \
  -H "x-api-key: $DOPOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "accountId": "cm3xyz789ghi012",
    "text": "Check out this photo!",
    "media": ["https://cdn.dopost.co/media/cm3media001xyz/photo.jpg"]
  }' \
  https://dopost.co/api/v1/post/publish
```

---

### 7. Reschedule a pending post

**Request**
```bash
curl -X PATCH \
  -H "x-api-key: $DOPOST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "publishAt": "2026-04-20T14:00:00Z" }' \
  https://dopost.co/api/v1/post/cm3sched001abc
```

**Response `200 OK`**
```json
{
  "id": "cm3sched001abc",
  "scheduledFor": "2026-04-20T14:00:00.000Z"
}
```

---

### 8. List posts with filter

**Request**
```bash
curl -H "x-api-key: $DOPOST_API_KEY" \
  "https://dopost.co/api/v1/post?status=PUBLISHED&limit=5"
```

**Response `200 OK`**
```json
{
  "posts": [
    {
      "id": "cm3abc123def456",
      "status": "PUBLISHED",
      "text": "Hello from the Dopost API!",
      "publishedAt": "2026-04-07T10:32:18.000Z",
      "account": {
        "id": "cm3xyz789ghi012",
        "platform": "INSTAGRAM",
        "platformUsername": "myaccount"
      }
    }
  ],
  "nextCursor": null,
  "hasMore": false
}
```

---

### 9. Delete a post

**Request**
```bash
curl -X DELETE \
  -H "x-api-key: $DOPOST_API_KEY" \
  https://dopost.co/api/v1/post/delete/cm3sched001abc
```

**Response `200 OK`**
```json
{
  "success": true,
  "deletedPostId": "cm3sched001abc"
}
```

---

## Analytics

All five endpoints below use `GET` and require `analytics:read` on the API key plus a Creator, Pro or Agency workspace. Existing keys do not gain new permissions automatically; create a key with the required scope when needed.

| Endpoint | Use |
|----------|-----|
| `/api/v1/analytics/overview` | Account KPIs, account/platform rankings, comparisons, and data coverage |
| `/api/v1/analytics/posts` | Rank posts published through dopost by performance |
| `/api/v1/analytics/posts/:postId` | Latest post metrics, collection progress, and checkpoint snapshots |
| `/api/v1/analytics/insights` | Best Time, Best Day, Format Opportunity, and Platform Pulse |
| `/api/v1/analytics/audience` | Available audience breakdowns for Facebook and Instagram |

### Filters and pagination

For overview, post lists, insights, and audience, use `days` (default 30) or both `startDate` and `endDate`. Ranges must fit the workspace's plan history. Choose at most one of `accountId`, comma-separated `accountIds`, or `groupId`; `platform` can also restrict the results. Audience requires a single `accountId`.

Overview supports `comparison=previous|none`, comma-separated `metrics`, and `includeSeries=true|false` (REST default: true). Use `includeSeries=false` when only summary KPIs are needed.

Post lists support `sortBy=overall|likes|engagements|comments|reach|views|shares|saves`, `limit` (1–100, default 20), and an opaque `cursor`. Follow `nextCursor` while `hasMore` is true when the user needs additional results. Do not imply the first page contains every post.

Audience supports `metric` (default `followerDemographics`), `dimension=age|city|country|gender|locale`, and `timeframe=day|this_week|this_month`. TikTok Analytics is unavailable until the integration is enabled and the account has the required permissions.

### Example requests

```bash
# Compare account performance without daily series.
curl -H "x-api-key: $DOPOST_API_KEY" \
  'https://dopost.co/api/v1/analytics/overview?days=30&comparison=previous&includeSeries=false'

# Find the strongest posts by engagement.
curl -H "x-api-key: $DOPOST_API_KEY" \
  'https://dopost.co/api/v1/analytics/posts?days=30&sortBy=engagements&limit=20'

# Inspect one post: replace the placeholder with an actual dopost post ID.
curl -H "x-api-key: $DOPOST_API_KEY" \
  'https://dopost.co/api/v1/analytics/posts/POST_ID'

# Read timing and format insights.
curl -H "x-api-key: $DOPOST_API_KEY" \
  'https://dopost.co/api/v1/analytics/insights?days=30'

# Read audience countries: replace the placeholder with an actual account ID.
curl -H "x-api-key: $DOPOST_API_KEY" \
  'https://dopost.co/api/v1/analytics/audience?accountId=ACCOUNT_ID&metric=followerDemographics&dimension=country'
```

Include `dataThrough`, `lastUpdatedAt`, or `asOf` when available. Treat omitted or `null` values as unavailable, not zero; a returned `0` is a measured zero. Explain partial coverage and insufficient insight readiness instead of inventing metrics or recommending a best time without data. Post lists omit posts without available analytics; a post detail `404` can mean no analytics exists or the post is outside the workspace.

For errors: `400` means invalid filters or a range outside plan history; `401` means invalid/missing authentication; `403` means missing scope or plan access; `429` means rate limiting. Do not retry authorization errors indefinitely.

Full endpoint reference: https://dopost.co/docs/api/endpoints/analytics/overview (and the sibling posts, post-detail, insights, and audience pages).

## MCP server (optional)

If the user has the Dopost MCP server configured, prefer using MCP tools (`publish_post`, `list_accounts`, etc.) over raw HTTP calls — they handle auth and path safety automatically.

For analytics, use `get_analytics_overview`, `list_post_performance`, `get_post_performance`, `get_content_insights`, and `get_audience_analytics`. MCP series are omitted by default and post lists return at most 20 items per call; follow the tool schema rather than applying REST limits. Reconnect OAuth to approve `analytics:read` if it was not previously granted.

MCP setup: https://dopost.co/docs/api/mcp/setup

## Workspace dates and Google Business Profile

Unzoned publication/reschedule dates use the workspace IANA timezone automatically.
Keep an explicit Z or offset if the user supplied one; do not invent a timezone.
An optional `timezone` override is accepted by publish/reschedule. Recurring
creation defaults to the workspace zone; updates retain the existing series zone.

Google Business accounts use `GOOGLE_BUSINESS`; list accounts with
`platform=google-business`. Connect and select verified locations in the dopost
web flow before using the returned dopost accountId. Google login alone does not
connect a location. Use the existing action scopes, never Google access tokens.

Google `platformOptions` supports `postType` (standard/event/offer/media),
`ctaType` (book/order/shop/learn_more/sign_up/call), `ctaUrl`, `eventTitle`,
`eventStart`, `eventEnd`, `couponCode`, `redeemOnlineUrl`, `termsConditions`,
and `mediaCategory`. Event/offer title and dates are required; text/media are
optional. Gallery media has no text/CTA. PROFILE and COVER require photos; LOGO
aliases PROFILE. Videos are MP4 gallery uploads only. Recurring supports standard
updates only. Query `/api/v1/social/limits/google-business` for conditional rules.

Google post readback includes platformOptions, providerStatus,
providerResourceName and structured error information. Submission acceptance
is not a guarantee of public visibility while Google processes/moderates it.

Analytics Overview accepts GOOGLE_BUSINESS and Search/Maps/device breakdowns,
websiteClicks, callClicks, directionRequests, messagingClicks, bookings and
menuClicks. Views aliases profile impressions; never add both. These are location
metrics, not post metrics. `/api/v1/analytics/search-terms` returns monthly terms
with per-location values or privacy thresholds, using the existing analytics:read
scope. Preserve null vs zero and thresholds; use nextCursor for pagination.
See https://dopost.co/docs/api/google-business for examples and the full contract.
