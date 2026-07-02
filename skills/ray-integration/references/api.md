# Ray API — condensed contract

Base URL `https://ray-api.gege.mn`. Auth `Authorization: Bearer $RAY_API_KEY` on every
request. **Source of truth:** `GET /openapi.json` (fetch it for exact shapes).

## Endpoints (API-key auth)
| Method | Path | Purpose |
|---|---|---|
| GET | `/me` | Resolve key context → `{ tenantId, apiKeyId, scopes[] }` (preflight / scope check) |
| GET | `/channels` | List configured channels → `id` (=`channelConfigId`), `kind`, `templateKind`, `recipientSchema` |
| POST | `/send` | Send a notification (single, fan-out, multi-channel, or feed-only) |
| GET | `/templates` | List templates (`?folder=`, `?includeArchived=true`) |
| POST | `/templates` | Create a template (raw only) → `{ id, requiredParams[] }` |
| GET | `/templates/:id` | Get a template (`published` + `draft` versions) |
| PATCH | `/templates/:id` | Update the draft (`publish:true` to also publish) |
| POST | `/templates/:id/publish` | Publish the current draft |
| POST | `/templates/:id/test-send` | Render + send a **test** (`is_test`, never in customer feeds) |
| POST | `/templates/:id/archive` | Archive a template |
| GET | `/sends/:id` | Per-send status detail |
| GET | `/notifications` | Customer feed for one end user (`externalUserId` **required**) |
| GET | `/clicks` | Email link click stats (`?sendId=` or `?campaignId=`, ≥1 required) → `{ totalClicks, byUrl[] }` |
| GET·POST | `/tenant-webhooks` | List / create delivery-status webhooks (create returns the signing secret **once**) |
| GET·PATCH·DELETE | `/tenant-webhooks/:id` | Get / update (`rotateSecret` to roll) / archive a webhook |

Public, no API key: `GET /healthz`, `GET /openapi.json`, `GET /docs`,
`POST /webhooks/{channelType}/{channelConfigId}` (provider callbacks, signature-verified),
`GET /c/{token}` (signed click-tracking redirect — recipients follow wrapped email links),
`GET /metrics` (optional `METRICS_AUTH_TOKEN` bearer).

## Discover channels — `GET /channels`
Returns the channels configured in the workspace so you don't have to hardcode UUIDs or guess
recipient shapes. Safe metadata only — **never** the encrypted provider credentials.
```
{ "channels": [
  { "id": "<uuid>",          // use as channelConfigId in /send and /templates/:id/test-send
    "name": "…",
    "kind": "ses_email",     // config kind: ses_email · smtp_email · fcm_push · slack_webhook · discord_webhook · telegram_bot · generic_webhook
    "templateKind": "email_html",  // the template channelKind this channel accepts
    "recipientSchema": { … }       // inline JSON Schema for the `recipient` /send expects
  } ] }
```
No params. `recipientSchema` is the authoritative per-channel `recipient` shape (varies by kind —
e.g. `{ email }` for email, `{ deviceToken | topic }` for push); the static shapes below are a
quick reference. Match a `templateId` whose kind equals the channel's `templateKind`.

## Channel kinds (templates)
`channelKind`: `email_html` · `fcm_basic` · `slack_text` · `discord_text` · `telegram_text` ·
`webhook_json` · `markdown` · `text`.

`content` by kind:
- **email_html**: `{ source:"raw", subject (1–998), bodyHtml, bodyText }` — `bodyText` is the
  required plain-text fallback. Shared by both SES and SMTP email channels.
- **fcm_basic**: `{ title, body, imageUrl?, data? }` — `data` is a string→string map delivered
  verbatim to your client SDK.
- **slack_text**: `{ text }`.
- **discord_text**: `{ content }` (≤2000 chars; Discord markdown; `@`-mentions disabled at send).
- **telegram_text**: `{ text, disableLinkPreview? }` (≤4096 chars; Telegram-HTML subset only:
  `<b> <i> <u> <s> <a href> <code> <pre>` — variable values are HTML-escaped).
- **webhook_json**: `{ title, body, data? }` — delivered as a `{ id, title, body, data, sentAt }`
  JSON envelope, HMAC-SHA256 signed (`X-Ray-Signature: sha256=<hex>`).

(Fetch the OpenAPI for the precise per-kind schema.)

## Recipient shapes (in `/send`)
Authoritative per channel via `GET /channels` → `recipientSchema`. Depends on the channel
config's kind. **Ray stores no recipient registry** — you supply these on every send and keep
the user → contact-info mapping yourself (see "pass-through recipients" in `SKILL.md`).
- **ses_email** / **smtp_email**: `{ "email": "a@b.com", "name": "Ada" }` (name optional; SES
  also accepts `cc` / `bcc` / `attachments` — see the OpenAPI).
- **fcm_push**: `{ "deviceToken": "…" }` **or** `{ "topic": "…" }` (exactly one). No device
  registry on Ray's side — pass the token (from your client SDK) each send, or a `topic` for
  multi-device fan-out.
- **slack_webhook** / **discord_webhook** / **generic_webhook**: no per-message recipient (the
  configured URL targets the destination) — pass `recipient: {}` / omit per the OpenAPI.
- **telegram_bot**: `{ "chatId": "…" }` — numeric chat/user id as a string, or
  `@channelusername`. The bot must already be a member of the target chat/channel.

## `POST /send` body
```
{
  "channelConfigId": "<uuid>",     // legacy/single mode: which configured channel to send through
  "templateId": "<uuid>",          // template send — XOR `content`; a published template of a compatible kind
  "content": { ... },              // inline send — XOR `templateId`; channel-shaped, raw (see below)
  "params": { "<var>": <any JSON> },// fills {{vars}} in the template OR inline content; arbitrary JSON (strings, numbers, bools, null, arrays, nested objects)
  "logTitle": "…",                 // inline send only; feed entry title (≤500)
  "logDescription": "…",           // inline send only; feed entry description (≤2000)
  "recipient": { ... },            // legacy/single mode; XOR targets, XOR deliveries
  "targets": [{ "recipient": {...}, "externalUserId": "u1" }],  // fan-out mode, 1–1000; XOR recipient, XOR deliveries
  "deliveries": [                  // multi-channel mode, 1–10; XOR recipient/targets
    { "channelConfigId": "…", "templateId": "…" /* | content */, "recipient": {...}, "params": {...} }
  ],
  "externalUserId": "user_123",    // required for feed-only sends; single-recipient label otherwise
  "feed": { "title": "…", "description": "…" },  // optional; creates ≤1 feed entry regardless of channel fan-out
  "showInFeed": false,             // optional (default false); legacy alias for `feed`
  "notBefore": "2026-01-01T00:00:00Z", // optional, schedule (ISO-8601, must end in Z)
  "priority": "high",              // optional, "high" (default) | "low"; "low" for marketing/bulk
  "trackClicks": false,            // optional, email only (default false); rewrite body links for click tracking
  "campaignId": "spring-sale"      // optional; groups click stats across sends (read via GET /clicks)
}
```
**Content source — exactly one of `templateId` or `content`, per delivery.** Inline `content`
uses the same per-kind shape as a template's `content` (e.g. email_html
`{ subject, bodyHtml, bodyText }`) and is **raw only** — designed/Maily content is dashboard-only,
same as `POST /templates`. Its `{{vars}}` must all be covered by `params`. `logTitle` /
`logDescription` are accepted **only** with inline `content` (template sends carry these on the
template version); they default to empty strings. Inline sends are not linked to a template
(their feed/log rows have a null `templateId`).

**Delivery — exactly one of three modes:**
- **Single** — top-level `channelConfigId` + `recipient`.
- **Fan-out** — `targets` (1–1000 entries), each `{ recipient, externalUserId? }`. Renders the
  body **once** and reuses it for every target (only `recipient` + `externalUserId` vary per
  target) — so it can't personalize per recipient; for `Hello {{name}}` send one `/send` per
  recipient instead.
- **Multi-channel** — `deliveries` (1–10 entries), each with its own `channelConfigId`,
  content source (`templateId` XOR `content`), `recipient`, and optional per-delivery `params`
  (falls back to the top-level `params`). The same logical notification over several channels
  for **one** recipient, identified by the top-level `externalUserId`. All deliveries share one
  `sendId`; not cross-producted with `targets[]` (loop over recipients yourself for bulk
  multi-channel).

**Feed-only sends** omit `recipient`/`targets`/`deliveries` entirely — just `feed` (its `title`
becomes required) + `externalUserId`. Zero dispatches, one feed entry, `send.completed` fires
immediately.

Returns **202** `{ sendId }` (one `sendId` even for fan-out/multi-channel; poll `GET /sends/:id`
for status). Header `Idempotency-Key: <uuid>` (recommended) — result is cached **24h**; replaying
with the *same* body returns the original result, a *different* body → **409**.

`priority` (`"high"` default, `"low"`) sets dispatch order on the shared queue — `low` marketing
bulk yields to `high` transactional mail so a campaign can't delay a magic-link email. It changes
fetch order, not rate; for rate isolation too, send marketing through a separate channel config.

`trackClicks` (email only, default `false`) rewrites every absolute `<a href="http(s)…">` in the
HTML body to a signed Ray redirect (`/c/{token}`) that records the click and 302s to the original
URL — works on SES and SMTP alike. Only the HTML body is rewritten (not plain text); `mailto:`/`tel:`/
anchors are skipped, and an `<a data-ray-no-track>` opts a link out (e.g. unsubscribe). Pass a
`campaignId` to group clicks across sends, then read counts via `GET /clicks?campaignId=…` (or
`?sendId=…`) → `{ totalClicks, byUrl: [{ url, clicks }] }`, same API key as `/send`. Counts include
bot/scanner clicks and are total (not unique) — a trend signal, not exact.

## `POST /templates/:id/test-send` body
```
{ "channelConfigId": "<uuid>", "recipient": { ... }, "params": { "<var>": <any JSON> } }
```
Returns **202** `{ sendId }`. The safe way to preview how a template renders: these rows are
`is_test` and never surface in `GET /notifications` or any customer feed. `channelConfigId` and
`recipient` are required.

## `POST /templates` body
```
{
  "name": "<unique within the workspace>",
  "folder": "",
  "channelKind": "email_html",
  "content": { "source":"raw", "subject":"…", "bodyHtml":"…", "bodyText":"…" },
  "logTitle": "…",            // required; the in-app feed entry title
  "logDescription": "…",      // required; feed entry description
  "paramOverrides": {},       // optional: { "<var>": { optional?: bool, description?: str } }
  "publish": true             // false = draft (default false)
}
```
Lengths: `name` ≤100, `folder` ≤200, `logTitle` ≤500, `logDescription` ≤2000. `content` is
not type-checked by the spec — it's validated server-side against `channelKind` (use the
per-kind shapes above). POST returns **201** `{ id, requiredParams }`; the response's
`requiredParams` is the authoritative list of detected `{{vars}}`.

## Conventions
- **Variables (Mustache subset)**: raw templates render with a Mustache-subset engine and `params`
  takes **arbitrary JSON** (strings, numbers, bools, null, arrays, nested objects) — plain strings
  still work unchanged. Scalar interpolation `{{var}}` / dotted `{{a.b}}` / current-item `{{.}}` is
  **escaped per channel** (HTML/Slack/Discord), XSS-safe. Sections `{{#x}}…{{/x}}` iterate an array
  / render once for a truthy value / nothing when falsy (`false`/`null`/`""`/`[]`/absent); inverted
  `{{^x}}…{{/x}}` renders only when falsy/empty. `requiredParams` is **section-aware** — only
  top-level interpolations are required; section names and section-interior vars are not listed.
  `paramOverrides[name].optional = true` relaxes a top-level one. **Not supported:** unescaped
  `{{{x}}}` / `{{& x}}` (treated as literal text — no raw HTML from senders), partials, helper
  functions; an unbalanced section is rejected with 400 at create/publish; visual/"designed"
  (Maily) templates stay flat-vars only.
- **Names** are unique per workspace; re-creating errors → GET then PATCH for idempotent imports.
- **`GET /notifications`** is the end-user feed, not an admin log: `externalUserId` is required.
  It reads dedicated feed entries (one per `(send, externalUserId)`, created via `feed`/
  `showInFeed`), **decoupled from delivery** — an entry stays visible even if every channel in
  the send failed or was suppressed; test sends never create one. `status` in the response is
  always `"delivered"` and `providerMessageId`/`dispatchedAt` are always `null` (kept for
  compatibility with the older delivery-backed feed) — use `GET /sends/:id` for real delivery
  diagnostics. Cursor-paginated (`cursor`, `limit` ≤200, `after`/`before`); filter by `status`
  (kept for compatibility — any non-`delivered` value returns an empty page), `templateId`,
  `channelConfigId` (a multi-channel entry carries its *first* delivery's ids; both `null` for
  feed-only sends).
- **Errors**: JSON `{ "error": "<code>", "message": "<detail>" }` (`error` is always present).
  Status codes: **400** validation / bad body, **401** missing or invalid key, **404** not found,
  **409** idempotency conflict (same `Idempotency-Key`, *different* body), **429** rate limited.
  There is **no 403 or 422** — validation failures are 400.
- **Rate limits**: per-workspace. **429** returns `{ error, message, retry_after_seconds }` —
  wait `retry_after_seconds` before retrying.
