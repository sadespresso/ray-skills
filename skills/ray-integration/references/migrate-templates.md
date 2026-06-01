# Migrating existing templates into Ray

Goal: import this project's existing HTML email templates into Ray via `POST /templates`.

## Before you start
- Need `RAY_API_KEY` (a **write**-scoped Ray key) in the environment. If unset, stop and ask.
- All requests: `Authorization: Bearer $RAY_API_KEY` against `https://ray-api.gege.mn`.

## Build the request safely
HTML has quotes/newlines, so assemble JSON with `jq`, not string interpolation:
```bash
jq -n --arg html "$(cat path/to/email.html)" --arg subject "Welcome" --arg text "Welcome!" \
  '{ name:"welcome", folder:"", channelKind:"email_html",
     content:{ source:"raw", subject:$subject, bodyHtml:$html, bodyText:$text },
     logTitle:"Welcome email", logDescription:"Sent to new users", publish:true }' \
| curl -sS -X POST https://ray-api.gege.mn/templates \
    -H "Authorization: Bearer $RAY_API_KEY" -H "Content-Type: application/json" -d @- | jq .
```

## Rules
1. **Variables** — Ray uses Mustache `{{var}}`. Any `{{placeholder}}` in subject / bodyHtml /
   bodyText / logTitle / logDescription becomes a **required send-time param**. Keep `{{ }}`
   that are genuine variables. If the source uses `{{ }}` for something that is NOT a send
   variable (a CSS/JS framework, etc.), it will become a spurious required param — **flag it
   and ask** before migrating that one.
2. **`subject`, `bodyText`, `logTitle`, `logDescription` are all required — never empty.**
   Derive sensible values (subject from `<title>` or filename; `bodyText` by stripping tags,
   keeping link URLs) and report your guesses.
3. **`channelKind` is always `email_html`, `content.source` is `"raw"`.** The API is raw-HTML
   only. If a template is **MJML** or uses includes/partials, compile + inline it to final HTML
   first, then send that. Do not attempt to migrate visual/"designed" templates here.
4. **Names are unique per workspace.** Be idempotent: `GET /templates` first; if one with the
   same `name` exists, `PATCH /templates/{id}` (same body, omit `name`) instead of `POST`.
5. **Do not send any emails or invent recipients** — this task only creates/updates templates.

## Process
1. Find all email templates in this repo (look in `emails/`, `templates/`, `mail/`, `*.html`,
   compiled MJML output, etc.).
2. Show the mapping `file → { name, subject, detected {{vars}}, bodyText preview }` and
   **wait for approval** before creating anything.
3. On approval, create/update each (POST, or PATCH if it already exists).
4. Report each template's `id`, detected required params, and publish state; note anything
   guessed or skipped.
5. Verify: `GET https://ray-api.gege.mn/templates` and confirm all expected templates are
   present with `channelKind: "email_html"` and a `publishedVersionId`.
