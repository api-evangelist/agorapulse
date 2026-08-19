---
name: Triage and reply to the Agorapulse inbox
description: Find inbox items across a workspace, read a conversation's messages, and post a reply — the API's only outbound community-management write.
api: openapi/agorapulse-items-api-openapi.yml
operations: [getManagerOrganizations, getOrganizationWorkspaces, findItems, getMessages, create1]
generated: '2026-08-13'
method: generated
source: openapi/_original/agorapulse-open-api-openapi.yml
---

# Triage and reply to the Agorapulse inbox

The inbox surface is three operations: find items, read a conversation, reply.
The reply is the only place the API speaks outbound to a social network, so treat
it with more care than anything else in this contract.

## Before you start

- `X-API-KEY: <key>`, base `https://api.agorapulse.com`.
- Resolve `organizationId` and `workspaceId` via `getManagerOrganizations` and
  `getOrganizationWorkspaces`.
- Note the identifier switch: the inbox addresses **`accountUid`**, not
  `profileUid`. They are different identifiers for adjacent things.

## Steps

1. **Find items** — `findItems`
   `GET /v1.0/inbox/organizations/{organizationId}/workspaces/{workspaceId}/items`.
   Returns `ItemsSearchResponse` of `ItemDTO`. Each item carries an `AbstractItem`
   payload and an `ItemSentiment`. Filter with `ItemFilterType` and order with
   `Order`.

   There is **no pagination contract** — no cursor, no page parameter, no limit is
   documented on this operation. Expect the whole collection and size your handling
   accordingly.

2. **Read the conversation** — `getMessages`
   `GET /v1.0/inbox/organizations/{organizationId}/workspaces/{workspaceId}/accounts/{accountUid}/conversations/{conversationId}/messages`.
   Returns `MessagesSearchResponse`. Read the thread before composing anything.

3. **Reply** — `create1`
   `POST /v1.0/inbox/organizations/{organizationId}/workspaces/{workspaceId}/accounts/{accountUid}/reply`
   with a `CreateReplyRequest`. Returns `200`.

## Guardrails for the reply

- This operation publishes to a real social account under the customer's brand.
  It is not reversible through this API — there is no delete-reply operation.
- **No idempotency key exists.** A retried reply is a second public reply. If the
  call times out, call `getMessages` and check whether it landed before retrying.
- The reply inherits the API key owner's permissions. A `403`-class failure means
  the underlying user cannot act on that account, not that the request was malformed.

## Staying current without polling

Do not poll `findItems` on a timer — you will burn the 500-requests-per-30-minutes
budget. Agorapulse pushes an `INBOX_ITEM` webhook with actions `CREATED`,
`CLASSIFIED` and `LABELIZED`, carrying the full `InboxItem` payload. Verify each
delivery with the `X-Hook-Signature` header (SHA256 HMAC of the raw body using the
subscription secret), return `200` to acknowledge, and return `410` if the endpoint
is retiring — Agorapulse will disable the subscription for you. See
`asyncapi/agorapulse-webhooks.yml`.

## Errors

`{ "code": …, "subCode": …, "message": … }`. `3` is rate limit exceeded — back off,
there is no documented `Retry-After` or `X-RateLimit-*` header to read. `1013` means
the gateway matched no endpoint; check the path segments, this surface has five of them.
