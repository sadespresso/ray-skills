# Ray API — condensed contract

Base URL `https://ray-api.gege.mn`. Auth `Authorization: Bearer $RAY_API_KEY` on every
request. **Source of truth:** `GET /openapi.json` (fetch it for exact shapes).

## Endpoints (API-key auth)
| Method | Path | Purpose |
|---|---|---|
| GET | `/me` | Resolve key context → `{ tenantId, apiKeyId, scopes[] }` (preflight / scope check) |
| POST | `/send` | Send a notification (single or fan-out) |
| GET | `/templates` | List templates (`?folder=`, `?includeArchived=true`) |
| POST | `/templates` | Create a template (raw only) → `{ id, requiredParams[] }` |
| GET | `/templates/:id` | Get a template (`published` + `draft` versions) |
| PATCH | `/templates/:id` | Update the draft (`publish:true` to also publish) |
| POST | `/templates/:id/publish` | Publish the current draft |
| POST | `/templates/:id/test-send` | Render + send a **test** (`is_test`, never in customer feeds) |
| POST | `/templates/:id/archive` | Archive a template |
| GET | `/sends/:id` | Per-send status detail |
| GET | `/notifications` | Customer feed for one end user (`externalUserId` **required**) |
| GET·POST | `/tenant-webhooks` | List / create delivery-status webhooks (create returns the signing secret **once**) |
| GET·PATCH·DELETE | `/tenant-webhooks/:id` | Get / update (`rotateSecret` to roll) / archive a webhook |

Public, no API key: `GET /healthz`, `GET /openapi.json`, `GET /docs`,
`POST /webhooks/{channelType}/{channelConfigId}` (provider callbacks, signature-verified),
`GET /metrics` (optional `METRICS_AUTH_TOKEN` bearer).

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
  "showInFeed": false,             // optional (default false)
  "notBefore": "2026-01-01T00:00:00Z"  // optional, schedule (ISO-8601, must end in Z)
}
```
`targets` holds 1–1000 entries. Returns **202** `{ sendId }` (one `sendId` even for fan-out;
poll `GET /sends/:id` for status). Header `Idempotency-Key: <uuid>` (recommended) — result is
cached **24h**; replaying with the *same* body returns the original result, a *different* body
→ **409**.

## `POST /templates/:id/test-send` body
```
{ "channelConfigId": "<uuid>", "recipient": { ... }, "params": { "<var>": "<string>" } }
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
- **Variables**: Mustache `{{var}}` anywhere in subject/body/logTitle/logDescription become
  required send params (auto-detected). `paramOverrides[name].optional = true` to relax one.
- **Names** are unique per workspace; re-creating errors → GET then PATCH for idempotent imports.
- **`GET /notifications`** is the end-user feed, not an admin log: `externalUserId` is required and
  only `show_in_feed=true`, non-test rows show. Cursor-paginated (`cursor`, `limit` ≤200,
  `after`/`before`); filter by `status` (default `delivered`), `templateId`, `channelConfigId`.
- **Errors**: JSON `{ "error": "<code>", "message": "<detail>" }` (`error` is always present).
  Status codes: **400** validation / bad body, **401** missing or invalid key, **404** not found,
  **409** idempotency conflict (same `Idempotency-Key`, *different* body), **429** rate limited.
  There is **no 403 or 422** — validation failures are 400.
- **Rate limits**: per-workspace. **429** returns `{ error, message, retry_after_seconds }` —
  wait `retry_after_seconds` before retrying.
