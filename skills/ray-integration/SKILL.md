---
name: ray-integration
description: Integrate a project with Ray, the notification/email platform at https://ray-api.gege.mn. Use when sending notifications or emails through Ray, creating or migrating email/notification templates into Ray, or wiring Ray's HTTP API into an app. Covers auth, GET /channels, POST /send, POST /templates, and the live OpenAPI spec.
---

# Ray integration

Ray is a multi-tenant notification platform (email via SES, push via FCM, Slack). This
project talks to it over HTTP.

- **Base URL:** `https://ray-api.gege.mn`
- **Authoritative, always-current contract:** `https://ray-api.gege.mn/openapi.json`
  (OpenAPI 3.1, generated from the server's own schemas — it can't drift). **Fetch it**
  whenever you need exact request/response shapes. Human UI: `https://ray-api.gege.mn/docs`.

When in doubt about any field, fetch the OpenAPI rather than guessing.

## Auth
Every request sends `Authorization: Bearer $RAY_API_KEY` (keys look like `ck_live_…`).
- `RAY_API_KEY` is a Ray API key (Ray dashboard → workspace → **API keys**). **Read it from
  the environment; never hardcode it.** Sending needs any key; creating/updating templates
  needs a **write**-scoped key.
- One key = one Ray **workspace** (tenant). Each product is normally its own workspace, with
  its own channel configs, templates, suppression list, and keys.
- **Preflight:** `GET /me` → `{ tenantId, apiKeyId, scopes }`. Cheap way to confirm the key is
  valid and (before template work) that `scopes` includes `write` — do this instead of guessing.

## Discover channels — `GET /channels`
Don't hardcode UUIDs or guess recipient shapes — list what the workspace actually has:
```
GET /channels → { channels: [{ id, name, kind, templateKind, recipientSchema }] }
```
Use a channel's `id` as `channelConfigId` below; `recipientSchema` is the exact `recipient`
object that channel expects; pick a `templateId` whose kind matches `templateKind`. Returns
safe metadata only — never provider credentials.

## Send a notification — `POST /send`
```
{
  "channelConfigId": "<a channel id from GET /channels>",
  "templateId": "<uuid>",                // template send — XOR `content`
  "params": { "name": "Ada" },           // fills {{vars}} (all strings)
  "recipient": { "email": "a@b.com" },   // shape depends on the channel (see references/api.md)
  "externalUserId": "user_123",          // optional; ties the delivery to a feed user
  "showInFeed": true                     // optional; surface it in the in-app feed
}
```
- **Content source — provide exactly one:**
  - `templateId` (+ `params`) — render a published template.
  - `content` — inline, channel-shaped content (e.g. email `{ subject, bodyHtml, bodyText }`),
    no template needed. `params` still fill its `{{vars}}`; add optional `logTitle` /
    `logDescription` for the in-app feed entry. **Raw only** — designed/Maily content stays
    dashboard-only, same as `POST /templates`.
- **Delivery — provide exactly one:** `recipient` (single) **or**
  `targets: [{ recipient, externalUserId? }, …]` (fan-out, one delivery/feed row each).
- Send an `Idempotency-Key: <uuid>` header so retries don't double-send.
- `notBefore: "<ISO-8601>"` schedules a future send.

## Create / migrate templates — `POST /templates`
For bulk import from an existing system, follow **`references/migrate-templates.md`**.
Quick shape: `channelKind: "email_html"`, `content: { source:"raw", subject, bodyHtml, bodyText }`,
plus `logTitle`, `logDescription`, and `publish`. The API is **raw HTML only** — visual /
"designed" (Maily) templates are dashboard-only. Template variables are Mustache `{{var}}`
and become **required params** at send time.
- **Verify a render without spamming anyone:** `POST /templates/:id/test-send` delivers to an
  address you control and is flagged `is_test` (never shows in customer feeds). Prefer it over
  a real `/send` when checking a migrated template.

## References
- `references/api.md` — condensed contract: endpoints, channel kinds, recipient & content
  shapes, errors, rate limits.
- `references/migrate-templates.md` — step-by-step migration of existing templates into Ray.
- Anything else → `https://ray-api.gege.mn/openapi.json`.
