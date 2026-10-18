---
name: dopost-api
description: Use the Dopost REST API to publish, schedule, manage social media posts, and retrieve account, post, content, and audience analytics. Use this skill when the user wants to publish to social media, schedule posts, manage media uploads, check post status, list connected accounts, analyze performance, or interact with any Dopost API endpoint. Also activate when the user mentions dopost, the Dopost API, social media automation, or publishing via API key.
metadata:
  openclaw:
    version: 1.3.0
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

#### Publish or schedule a post
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
- At least `text` or `media` is required, except for Google Business events and offers with the required event fields
- `media` can be an array of URL strings or objects `{ url: "..." }`
- Maximum 10 media items per post
- Omit `publishAt` (or set to `null`) to publish immediately
- Returns `202` with `{ success, jobId, postId, status }`
- Publishing is **asynchronous** — poll `GET /api/v1/post/:postId` to check status

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
Returns `{ success: true, deletedPostId: "..." }`.

---

### Media

#### Upload media (presigned URL flow)
```
POST /api/v1/media
Scope: media:upload
Body: { "fileName": "photo.jpg", "contentType": "image/jpeg", "sizeInBytes": 204800 }
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
- `publicUrl` is the permanent URL — use this as the `media` URL when publishing
- Maximum file size: 1 GB
- After the PUT, call `/finalize` (see below) to confirm the upload

```bash
curl -X PUT -H "Content-Type: image/jpeg" --data-binary @photo.jpg "$UPLOAD_URL"
```

#### Finalize media upload
```
POST /api/v1/media/{id}/finalize
Scope: media:upload
```
Must be called after the PUT to confirm the upload. Verifies the file exists in storage, extracts metadata (dimensions, duration, thumbnail), and marks the record as `READY`. Media **only appears in the library** after finalization.

```bash
curl -X POST -H "x-api-key: $DOPOST_API_KEY" "https://dopost.co/api/v1/media/$MEDIA_ID/finalize"
```

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
2. `POST /api/v1/post/publish` with `accountId` + `text`
3. `GET /api/v1/post/:postId` — poll until `status` is `PUBLISHED` or `FAILED`

### Publish a post with an image
1. `POST /api/v1/media` — get `id`, `uploadUrl` + `publicUrl`
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
| `429` | Rate limit or monthly quota exceeded. Check `X-RateLimit-*` headers |

On `429`, respect the `Retry-After` header before retrying.

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
    "sizeInBytes": 204800
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

**Step 3 — Publish using the public URL**
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
