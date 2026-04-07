---
name: dopost-api
description: Use the Dopost REST API to publish, schedule, and manage social media posts programmatically. Use this skill when the user wants to publish to social media, schedule posts, manage media uploads, check post status, list connected accounts, or interact with any Dopost API endpoint. Also activate when the user mentions dopost, the Dopost API, social media automation, or publishing via API key.
license: MIT
metadata:
  version: 1.0.0
  author: dopost-co
---

# Dopost API

Dopost is a social media management platform. Its REST API lets you publish and schedule posts, manage media, and inspect connected social accounts.

**Base URL:** `https://dopost.co/api/v1`  
**Auth:** All requests require the header `x-api-key: <your-key>`  
**Full docs:** https://dopost.co/docs/api

---

## Setup

Always check for an `.env` or `.env.local` file with `DOPOST_API_KEY`. If not present, ask the user for their key before proceeding.

```bash
export DOPOST_API_KEY="dpk_live_..."
```

---

## Endpoints

### Social Accounts

#### List connected accounts
```
GET /api/v1/social/accounts
Scope: social:read
```
```bash
curl -H "x-api-key: $DOPOST_API_KEY" https://dopost.co/api/v1/social/accounts
```
Returns `{ accounts: [{ id, platform, platformUsername, platformUserId }] }`.  
The `id` field is the `accountId` needed for publishing.

#### Get platform limits
```
GET /api/v1/social/limits/:platform
Scope: social:read
```
Platforms: `instagram`, `tiktok`, `youtube`, `facebook`, `linkedin`, `x`, `threads`, `bluesky`, `mastodon`, `pinterest`

```bash
curl -H "x-api-key: $DOPOST_API_KEY" https://dopost.co/api/v1/social/limits/instagram
```

---

### Posts

#### Publish or schedule a post
```
POST /api/v1/post/publish
Scope: post:write
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
- At least `text` or `media` is required
- Omit `publishAt` (or set to `null`) to publish immediately
- Returns `202` with `{ jobId, postId, statusUrl }`
- Publishing is **asynchronous** — poll the status endpoint

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

#### Check post status
```
GET /api/v1/post/status/:jobId
Scope: post:read
```
Poll this after publishing. Status values: `processing`, `published`, `failed`, `scheduled`.

#### List posts
```
GET /api/v1/post?status=PENDING&limit=20&cursor=<cursor>
Scope: post:read
```
Status filter values: `DRAFT`, `PENDING`, `PUBLISHED`, `FAILED`

#### Get a post
```
GET /api/v1/post/:postId
Scope: post:read
```

#### Reschedule a post
```
PATCH /api/v1/post/:postId
Scope: post:write
Body: { "publishAt": "2025-12-31T09:00:00Z" }
```
Only works on posts with status `PENDING`. The new date must be in the future.

#### Delete a post
```
DELETE /api/v1/post/delete/:postId
Scope: post:write
```
Cancels scheduling automatically if the post is `PENDING`.

---

### Media

#### Upload media (presigned URL flow)
```
POST /api/v1/media
Scope: media:write
Body: { "fileName": "photo.jpg", "contentType": "image/jpeg", "sizeInBytes": 204800 }
```
Returns `{ mediaId, uploadUrl, publicUrl }`.  
Then upload the file with a PUT request to `uploadUrl`:

```bash
curl -X PUT -H "Content-Type: image/jpeg" --data-binary @photo.jpg "$UPLOAD_URL"
```

Use `publicUrl` as the `media` URL when publishing.

#### List media
```
GET /api/v1/media?limit=20&cursor=<cursor>
Scope: media:read
```

#### Delete media
```
DELETE /api/v1/media/:mediaId
Scope: media:write
```

---

## Common workflows

### Publish a post now
1. `GET /api/v1/social/accounts` → pick `accountId`
2. `POST /api/v1/post/publish` with `accountId` + `text`
3. `GET /api/v1/post/status/:jobId` → poll until `published` or `failed`

### Publish a post with an image
1. `POST /api/v1/media` → get `uploadUrl` + `publicUrl`
2. PUT the file to `uploadUrl`
3. `POST /api/v1/post/publish` with `media: [publicUrl]`
4. Poll status

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
| `429` | Rate limit or monthly quota exceeded. Check `X-RateLimit-*` headers |

On `429`, respect the `Retry-After` header before retrying.

---

## MCP server (optional)

If the user has the Dopost MCP server configured, prefer using MCP tools (`publish_post`, `list_accounts`, etc.) over raw HTTP calls — they handle auth and path safety automatically.

MCP setup: https://dopost.co/docs/api
