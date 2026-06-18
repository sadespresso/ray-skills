---
name: ray-integration
description: Integrate a project with Ray, the notification/email platform at https://ray-api.gege.mn. Use when sending notifications or emails through Ray, creating or migrating email/notification templates into Ray, or wiring Ray's HTTP API into an app. Covers auth, GET /channels, POST /send, POST /templates, and the live OpenAPI spec.
---

# Ray integration

Ray is a multi-tenant notification platform — email (Amazon SES or raw SMTP), push (FCM),
Slack, Discord, Telegram, and generic signed webhooks, all behind one HTTP API. This
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
  "params": { "name": "Ada" },           // fills {{vars}}; arbitrary JSON (see Template params below)
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
- `priority: "high" | "low"` (default `"high"`). Mark marketing/bulk sends `"low"` so they
  yield to transactional mail — see **Bulk & marketing** below.
- `trackClicks: true` (email only, default `false`) rewrites HTML body links so clicks are
  counted; pass an optional `campaignId` to group them. Read counts via `GET /clicks` — see
  **Click tracking** below.

**Ray is a stateless relay — it keeps no recipient registry.** You pass the channel-shaped
recipient on every send (an email, an FCM `deviceToken`/`topic`, a Telegram `chatId`, …) and
own the user → contact-info mapping in *your* system. There is no "register device /
subscriber" step and no "send to user U" call — same as you'd pass an email address rather
than pre-registering it. `externalUserId` only labels the delivery for the in-app feed; it
never decides who actually receives the message. (Migrating from OneSignal/Firebase/Knock,
which hold the device registry for you? This is the inverted model — supply the token each
send, or use an FCM `topic` for multi-device fan-out.)

### Bulk & marketing
Ray has no audience/segment/subscriber store — your system owns the contact list, segmentation,
unsubscribe links, and consent. To send a campaign you build the recipient list yourself and call
`POST /send`. Two things to get right:
- **Personalization (`Hello {{name}}`):** a `targets[]` fan-out renders the body **once** and
  reuses it for everyone — the only per-recipient fields are `recipient` and `externalUserId`, so
  it can't personalize the body. For per-recipient content, send **one `/send` per recipient**
  (`recipient` + that person's `params`, or pre-rendered inline `content`). Reserve `targets[]`
  for identical-body blasts. Either way Ray queues the sends and paces them to the channel's
  provider rate limit — you don't throttle client-side.
- **Don't starve transactional mail:** mark every marketing/bulk send `"priority": "low"` so a
  magic-link or receipt sent mid-campaign jumps ahead instead of waiting behind the blast. For
  full isolation, send marketing through a **separate channel config** (its own provider
  credential → its own independent rate bucket, so the blast can't consume the transactional
  send rate either).

### Click tracking
Set `trackClicks: true` on an email send and Ray rewrites every absolute `<a href="http(s)…">` in
the HTML body to a signed redirect (`/c/<token>`) that records the click and 302s to the original
URL — transport-agnostic (SES + SMTP), since the rewrite happens at send time. Opt-in per send.
- Only the HTML body is rewritten (not plain text); `mailto:`/`tel:`/anchors are left alone, and an
  `<a data-ray-no-track>` opts a link out (e.g. unsubscribe/legal links).
- Pass a `campaignId` to aggregate clicks across many sends (e.g. one `/send` per recipient in a
  campaign), then read **`GET /clicks?campaignId=…`** (or `?sendId=…`) → `{ totalClicks, byUrl: [{ url, clicks }] }`
  with the same key you send with.
- Counts include bot/scanner pre-fetches and are total, not unique — a trend signal, not exact.

## Create / migrate templates — `POST /templates`
For bulk import from an existing system, follow **`references/migrate-templates.md`**.
Quick shape: `channelKind: "email_html"`, `content: { source:"raw", subject, bodyHtml, bodyText }`,
plus `logTitle`, `logDescription`, and `publish`. The API is **raw HTML only** — visual /
"designed" (Maily) templates are dashboard-only. Template variables use the Mustache subset
below; top-level `{{var}}` interpolations become **required params** at send time.
- **Verify a render without spamming anyone:** `POST /templates/:id/test-send` delivers to an
  address you control and is flagged `is_test` (never shows in customer feeds). Prefer it over
  a real `/send` when checking a migrated template.

### Template params & syntax (Mustache subset)
Raw-HTML templates render with a **Mustache-subset** engine, and `params` accepts **arbitrary
JSON** (strings, numbers, booleans, `null`, arrays, nested objects) — not just strings. **Plain
strings still work unchanged**, so existing flat `{{var}}` templates need no migration.
- `{{var}}` / `{{a.b}}` / `{{.}}` — escaped scalar interpolation (per-channel escaping, XSS-safe);
  dotted paths and the current item.
- `{{#x}}…{{/x}}` — **section**: iterates an array (once per element), renders once for a truthy
  object/scalar, or nothing when falsy (`false`/`null`/`""`/`[]`/absent).
- `{{^x}}…{{/x}}` — **inverted section**: renders only when the value is falsy/empty.
- `required_params` is **section-aware**: only top-level interpolations are required; section names
  and variables used inside a section are not listed.

Example — a digest with one card per league:
```
{{#boards}}<div class="card"><b>{{name}}</b> · {{count}}{{#questions}}<p>{{text}}</p>{{/questions}}</div>{{/boards}}{{^boards}}<p>Nothing pending 🎉</p>{{/boards}}
```
```json
{ "params": { "boards": [ { "name": "Asian League", "count": 1, "questions": [ { "text": "Who tops the group?" } ] } ] } }
```
Caveats: unescaped output `{{{x}}}` / `{{& x}}` is **not supported** (treated as literal text — no
raw HTML from senders); **partials and helper functions are unsupported**; an unbalanced section
(unclosed/mismatched `{{#x}}…{{/x}}`) is **rejected with 400 at create/publish**; and visual /
"designed" (Maily) templates stay **flat-vars only** — sections are a raw-HTML capability.

## References
- `references/api.md` — condensed contract: endpoints, channel kinds, recipient & content
  shapes, errors, rate limits.
- `references/migrate-templates.md` — step-by-step migration of existing templates into Ray.
- Anything else → `https://ray-api.gege.mn/openapi.json`.
