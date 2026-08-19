---
name: Pull Agorapulse profile reports
description: Walk the organization -> workspace -> profile chain and pull audience, content and community-management reports for a social profile over a date range.
api: openapi/agorapulse-reports-api-openapi.yml
operations: [getManagerOrganizations, getOrganizationWorkspaces, getWorkspaceProfiles, getAudience, getCommunityManagement, getContentReport, health]
generated: '2026-08-13'
method: generated
source: openapi/_original/agorapulse-open-api-openapi.yml
---

# Pull Agorapulse profile reports

Agorapulse's reporting endpoints are the reason most integrations exist: export
audience, content and community-management analytics for one social profile into
a BI tool. Nothing is addressable until you have three identifiers, so the walk
below is mandatory, not optional.

## Before you start

- Base URL `https://api.agorapulse.com`. Every path is prefixed `/v1.0/`.
- Send `X-API-KEY: <key>` on **every** request. There is no OAuth on the REST API.
- The key inherits the permissions of the user who created it. Use an org owner's
  or manager's key or you will silently see fewer profiles than exist.
- Reporting access is documented as a **Custom plan** feature. A key on a lower
  plan authenticates but has nothing to read.
- Budget: **500 requests per 30 minutes per key**. The walk below costs 3 requests
  before you fetch a single report, so cache the three identifiers.

## Steps

1. **Check the service is up** — `health`
   `GET /v1.0/report/health`. Cheap, unauthenticated-shaped probe to confirm
   connectivity before you burn requests on the walk.

2. **List organizations** — `getManagerOrganizations`
   `GET /v1.0/core/organizations`. Returns `OrganizationListResponse`. Take the
   `organizationId` (int64) you want.

3. **List workspaces** — `getOrganizationWorkspaces`
   `GET /v1.0/core/organizations/{organizationId}/workspaces`. Returns
   `WorkspaceListResponse`. Take the `workspaceId`.

4. **List profiles** — `getWorkspaceProfiles`
   `GET /v1.0/core/organizations/{organizationId}/workspaces/{workspaceId}/profiles`.
   Returns `ProfileListResponse`. Take the **`profileUid`** — a UID string, not the
   numeric profile id. Note the profile's network; it decides how you read the
   report body.

5. **Pull the reports** — one call per report type, all three take the same shape:
   - `getAudience` — `GET .../profiles/{profileUid}/insights/audience`
   - `getContentReport` — `GET .../profiles/{profileUid}/insights/content`
   - `getCommunityManagement` — `GET .../profiles/{profileUid}/insights/communitymanagement`

   All three live under `/v1.0/report/organizations/{organizationId}/workspaces/{workspaceId}/`.

## Date range — the trap

`since` and `until` are **Unix timestamps** on the REST API. If you are used to the
MCP tools, note they take ISO 8601 strings and convert internally; calling REST with
an ISO string will fail validation. Convert before you call.

## Reading the response

Report payloads are **specialised per network**, not generalised. `OpenAudienceInsight`
resolves to `FacebookAudienceInsight`, `InstagramAudienceInsight`,
`LinkedinAudienceInsight`, `TiktokAudienceInsight`, `TwitterAudienceInsight`,
`YoutubeAudienceInsight` or `ThreadsAudienceInsight` depending on the profile.
Branch on the profile's network before you read fields — do not assume a common
metric set exists across networks.

## Errors

Errors are **not** RFC 9457. The envelope is:

```json
{ "code": 1005, "subCode": 1104, "message": "…" }
```

`code` is either a global family — `1` internal, `2` unauthorized, `3` rate limit
exceeded, `4` unprocessable input, `5` validation failed — or the component that
produced the error. `1013` means the API gateway matched no endpoint at all, so
check your path before you debug the feature. `405`, `406` and `415` return a
status with **no body**. Never parse `message`.

## What is out of scope

Custom reports, Social Media ROI reports and Listening data are documented as
excluded from the API. Do not attempt to derive them from these three endpoints.
