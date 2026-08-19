---
name: Upload media and stage an Agorapulse draft
description: Create a media upload slot, poll it to ready, and stage a simple draft post — including which of the two media surfaces to use and why the draft operation is deprecated.
api: openapi/agorapulse-media-api-openapi.yml
operations: [getManagerOrganizations, getOrganizationWorkspaces, getWorkspaceProfiles, create, get, requestUpload, save_1, list]
generated: '2026-08-13'
method: generated
source: openapi/_original/agorapulse-open-api-openapi.yml
---

# Upload media and stage an Agorapulse draft

Agorapulse exposes **two different media surfaces**. Picking the wrong one is the
most common way to get stuck here.

| Surface | Operation | Path | Use it for |
|---|---|---|---|
| Publishing media | `create`, `get` | `/v1.0/publishing/.../media` | Media you intend to attach to a post |
| Studio media | `requestUpload` | `/v1.0/library/.../studio/media/upload` | Uploading into the content library (Studio) |

## Before you start

- `X-API-KEY: <key>`, base `https://api.agorapulse.com`.
- Resolve `organizationId` and `workspaceId` first.

## Steps — publishing media

1. **Create an upload slot** — `create`
   `POST /v1.0/publishing/organizations/{organizationId}/workspaces/{workspaceId}/media`
   with a `CreateMediaOpenRequest`. Returns `201` and a `MediaOpenResponse` carrying
   a `mediaUid` and a `PublishingApiMediaStatus`. A `400` here is "Invalid or
   unsupported file name" — fix the name, not the bytes.

2. **Poll the slot to ready** — `get`
   `GET .../media/{mediaUid}`. Returns `MediaStatusOpenResponse` with the status,
   a `PublishingApiMediaMetadata`, a `PublishingApiMediaError` when processing
   failed, and `NetworkCompatibility` telling you which networks will accept this
   asset. **Check compatibility before you attach it** — a file that is valid for
   Facebook may be rejected by Instagram or TikTok.

   `410 Media has expired` means the slot aged out. It cannot be recovered; go back
   to step 1 and upload again. Poll with backoff — this loop shares the
   500-requests-per-30-minutes budget with everything else.

## Steps — studio media

`requestUpload` — `POST /v1.0/library/organizations/{organizationId}/workspaces/{workspaceId}/studio/media/upload`
with a `StudioMediaUploadOpenRequest`. Returns `201` and a
`StudioMediaUploadOpenResponse` containing a **presigned upload URL**. PUT your
bytes to that URL directly; it is short-lived, so request it immediately before
uploading rather than caching it.

## Staging a draft — and its deprecation

`save_1` — `POST /v1.0/publishing/organizations/{organizationId}/workspaces/{workspaceId}/simple-drafts`
with a `CreateSimpleDraftOpenRequest` (`PostType`, `PostNetworks`, `PostMedia`,
and a per-profile `ProfileScheduling`). Returns `201` and a
`CreateSimpleDraftOpenResponse` wrapping a `GroupOfPostsSummary`.

**This operation is marked `deprecated: true` in the published contract.** No
`Sunset` header, no removal date and no named replacement are published, so treat
it as usable-but-unsafe: build against it only with a plan to re-check the spec at
`https://api.agorapulse.com/docs/open-api.yml` before each release.

## Pinterest

If the target profile is Pinterest you need a board. `list` —
`GET /v1.0/publishing/organizations/{organizationId}/workspaces/{workspaceId}/profiles/{profileUid}/pinterest/boards`
returns `PinterestBoardListOpenResponse`. A `400` here means the profile is not in
the workspace, is not a Pinterest profile, or has no usable Pinterest token —
three distinct causes behind one status, so check the `message`.

## Retry rule

None of these writes is idempotent and none accepts an idempotency key. A retried
`create` makes a second upload slot; a retried `save_1` makes a second draft.
Reconcile before retrying.

## Watching the result

Subscribe to the `PUBLISHING_POST` webhook (actions `PUBLISHED`, `FAILED`) rather
than polling for publication outcome. Verify with `X-Hook-Signature`.
