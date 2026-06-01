# Ray API — condensed contract

Base URL `https://ray-api.gege.mn`. Auth `Authorization: Bearer $RAY_API_KEY` on every
request. **Source of truth:** `GET /openapi.json` (fetch it for exact shapes).

## Endpoints (API-key auth)
| Method | Path | Purpose |
|---|---|---|
| POST | `/send` | Send a notification (single or fan-out) |
| GET | `/templates` | List templates (`?folder=`, `?includeArchived=true`) |
| POST | `/templates` | Create a template (raw only) |
| GET | `/templates/:id` | Get a template (published + draft) |
| PATCH | `/templates/:id` | Update a template's draft (`publish:true` to publish) |
| POST | `/templates/:id/publish` | Publish the current draft |
| GET | `/sends/:id` | Aggregate status of a fan-out send |
| GET | `/notifications` | Delivery rows (admin/debug) |

Public, no API key: `GET /healthz`, `GET /openapi.json`, `GET /docs`, `POST /webhooks/*`
(provider callbacks), `GET /metrics` (optional bearer).

## Channel kinds (templates)
`channelKind`: `email_html` · `fcm_basic` · `slack_text` · `markdown` · `text`.

`content` by kind:
- **email_html**: `{ source:"raw", subject (1–998), bodyHtml, bodyText }` — `bodyText` is the
  required plain-text fallback.
- **fcm_basic**: `{ title, body, imageUrl? }`.
- **slack_text**: `{ text }`.

(Fetch the OpenAPI for the precise per-kind schema.)

## Recipient shapes (in `/send`)
Depends on the channel config's kind:
- **ses_email**: `{ "email": "a@b.com", "name": "Ada" }` (name optional).
- **fcm_push**: `{ "deviceToken": "…" }` **or** `{ "topic": "…" }` (not both).
- **slack_webhook**: no per-message recipient (posts to the configured webhook) — pass
  `recipient: {}` / omit per the OpenAPI.

## `POST /send` body
```
{
  "channelConfigId": "<uuid>",     // which configured channel to send through
  "templateId": "<uuid>",          // a published template of a compatible kind
  "params": { "<var>": "<string>" },// fills {{vars}}; all values are strings
  "recipient": { ... },            // XOR targets
  "targets": [{ "recipient": {...}, "externalUserId": "u1" }],  // XOR recipient, fan-out
  "externalUserId": "user_123",    // optional
  "showInFeed": false,             // optional
  "notBefore": "2026-01-01T00:00:00Z"  // optional, schedule
}
```
Headers: `Idempotency-Key: <uuid>` (recommended). Returns a `sendId`.

## `POST /templates` body
```
{
  "name": "<unique within the workspace>",
  "folder": "",
  "channelKind": "email_html",
  "content": { "source":"raw", "subject":"…", "bodyHtml":"…", "bodyText":"…" },
  "logTitle": "…",            // required; the in-app feed entry title
  "logDescription": "…",      // required; feed entry description
  "paramOverrides": {},       // optional: mark a {{var}} optional / describe it
  "publish": true             // false = draft
}
```

## Conventions
- **Variables**: Mustache `{{var}}` anywhere in subject/body/logTitle/logDescription become
  required send params (auto-detected). `paramOverrides[name].optional = true` to relax one.
- **Names** are unique per workspace; re-creating errors → GET then PATCH for idempotent imports.
- **Errors**: JSON `{ "name": "...Error", "message": "..." }`; 401 missing/bad key, 403/422
  scope or validation, 404 not found, 409 idempotency replay returns the original `sendId`.
- **Rate limits**: per-workspace; responses carry `RateLimit-*` headers.
