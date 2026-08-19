---
name: Manage Agorapulse calendar notes
description: Search, create, update and delete publishing calendar notes in a workspace, with the permission and retry caveats that apply to the only full CRUD surface in the API.
api: openapi/agorapulse-calendar-notes-api-openapi.yml
operations: [getManagerOrganizations, getOrganizationWorkspaces, search, save, update, delete]
generated: '2026-08-13'
method: generated
source: openapi/_original/agorapulse-open-api-openapi.yml
---

# Manage Agorapulse calendar notes

Calendar notes are the only resource in the Agorapulse API with a complete CRUD
surface. Everything else is read-only or create-only.

## Before you start

- `X-API-KEY: <key>` on every request; base `https://api.agorapulse.com`.
- You need `organizationId` and `workspaceId` first — see
  `getManagerOrganizations` then `getOrganizationWorkspaces`.
- All four operations sit under
  `/v1.0/publishing/organizations/{organizationId}/workspaces/{workspaceId}/notes`.

## Steps

1. **Search existing notes** — `search`
   `GET .../notes`. Returns `SearchCalendarNoteOpenResponse` containing
   `CalendarNoteOpenItem` entries. `400` means your search parameters are invalid;
   `404` means the organization or workspace does not exist (or your key cannot
   see it).

2. **Create a note** — `save`
   `POST .../notes` with a `CreateCalendarNoteOpenRequest`. The `color` field takes
   a `PublishingCalendarNoteColor`. Returns `201` with
   `CreateCalendarNoteOpenResponse` — capture the `uid`, it is how you address the
   note afterwards.

3. **Update a note** — `update`
   `PUT .../notes/{uid}`. Returns `200`. A `403` here is a permissions failure, not
   a bad request: the API key inherits its creator's in-app role and that user
   cannot edit this note.

4. **Delete a note** — `delete`
   `DELETE .../notes/{uid}`. Returns `204` with no body. Same `403` semantics as
   update; `404` if the note, organization or workspace is not found.

## Retry rule — read this before you write

**There is no idempotency contract.** No `Idempotency-Key` header, no request-id,
no documented retry semantics on any write operation. If `save` times out and you
retry it, you will create a second calendar note. Before retrying a create, call
`search` and reconcile; treat a retry as a repair step, not an automatic one.
`update` and `delete` are naturally idempotent because they address a `uid`.

## Errors

`{ "code": …, "subCode": …, "message": … }` — not RFC 9457. `2` is unauthorized,
`3` is rate limit exceeded, `4` unprocessable input, `5` validation failed, `1013`
is the gateway saying no endpoint matched the path. `405`/`406`/`415` return no
body at all.

## Budget

500 requests per 30 minutes per key, shared with every other call that key makes.
