# Saving email templates to Ray

The slim write-path contract this skill needs. For auth context, `GET /me`, `GET /channels`, and
sending, use the **`ray-integration`** skill; the always-current source of truth is
`https://ray-api.gege.mn/openapi.json`.

- Base URL `https://ray-api.gege.mn`. Every request: `Authorization: Bearer $RAY_API_KEY`.
- Managing templates needs a **write**-scoped key (`GET /me` → `scopes` includes `write`).
- All email templates here are `channelKind:"email_html"`, `content.source:"raw"`.

## Endpoints used

| Method | Path | Purpose |
|---|---|---|
| GET | `/templates` | List → find an existing template by `name` (for upsert). `?folder=`, `?includeArchived=true` |
| POST | `/templates` | Create → `{ id, requiredParams[] }` |
| GET | `/templates/:id` | Fetch `published` + `draft` versions |
| PATCH | `/templates/:id` | Update the draft (`publish:true` to also publish) |
| POST | `/templates/:id/publish` | Publish the current draft |
| POST | `/templates/:id/test-send` | Render + send a **test** (`is_test`, never in customer feeds) |
| GET | `/channels` | Get a `channelConfigId` (email channel) for test-send — see `ray-integration` |

## `POST /templates` body

```json
{
  "name": "password-reset",
  "folder": "",
  "channelKind": "email_html",
  "content": {
    "source": "raw",
    "subject": "Reset your password",
    "bodyHtml": "<!DOCTYPE html>… shell with body injected …",
    "bodyText": "Reset your password\n\nClick: {{reset_url}}\n\nUnsubscribe: {{unsubscribe_url}}"
  },
  "logTitle": "Password reset requested",
  "logDescription": "A reset link was sent.",
  "paramOverrides": {},
  "publish": false
}
```

- Lengths: `name` ≤100, `folder` ≤200, `subject` 1–998, `logTitle` ≤500, `logDescription` ≤2000.
- `content` is validated **server-side** against `channelKind` (not by the spec schema) — match the
  `email_html` shape exactly.
- Response `requiredParams` is the **authoritative** list of `{{vars}}` Ray detected — use it for
  the manifest (below).
- `paramOverrides[name] = { optional?: bool, description?: str }` relaxes/annotates a var. Use
  sparingly — by default every var is required.

## Idempotent upsert by name

Template `name` is unique per workspace, so a second `POST` with the same name errors. Re-running
the skill must update, not break:

```
list = GET /templates?includeArchived=true
existing = list.templates.find(t => t.name === name)

if existing:
    PATCH /templates/{existing.id}  with { content, logTitle, logDescription, publish:false }
else:
    POST /templates                 with { name, channelKind, content, logTitle, logDescription, publish:false }
```

Always save as a **draft** first (`publish:false`). Don't publish until verified + confirmed.

## Verify, then publish

1. **Test-send** (real inbox, `is_test`):
   ```json
   POST /templates/:id/test-send
   { "channelConfigId": "<an email channel id from GET /channels>",
     "recipient": { "email": "you@yourco.com" },
     "params": { "reset_url": "https://app.example.com/r/abc", "unsubscribe_url": "https://app.example.com/u/abc" } }
   ```
   → `202 { sendId }`. Supply a value for **every** required param or it `400`s.
2. **Publish** only on explicit user confirmation:
   ```
   POST /templates/:id/publish        # or PATCH /templates/:id { …, "publish": true }
   ```

## Param manifest (print after saving each template)

Give the integrator the exact contract so their real sends don't `400`:

```
Template: password-reset  (draft saved, id=<uuid>)
Required params: reset_url, unsubscribe_url        # from POST/PATCH `requiredParams`
Sample send (via ray-integration / POST /send):
  POST /send
  { "channelConfigId": "<email channel id>",
    "templateId": "<uuid>",
    "recipient": { "email": "user@theirco.com" },
    "params": { "reset_url": "https://…", "unsubscribe_url": "https://…" } }
```

Note which params are **per-recipient** (e.g. `unsubscribe_url`) vs. content values, so the caller
knows what their backend must compute per send. Ray stores no recipient/token registry — callers
pass the recipient and params on every send (see `ray-integration`).
