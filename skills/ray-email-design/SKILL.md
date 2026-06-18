---
name: ray-email-design
description: Design consistent, on-brand email templates and save them to Ray. Use when creating new email templates, restyling existing emails, or standardizing the look of a project's emails for Ray. Derives a brand profile from the repo's existing emails/assets (or asks), builds ONE frozen email-safe HTML shell, then generates raw-HTML templates that all reuse it — with local preview, test-send, and publish via POST /templates.
---

# Ray email design

Produce **visually consistent** transactional emails for Ray and save them via `POST /templates`.
The trick to consistency is a **single frozen HTML shell** (header, footer, wrapper) that every
template's body is injected into — so the chrome is byte-identical across the whole set, not just
"the same colours."

**Hard Ray constraints (don't fight them):**
- The API is **raw HTML only** (`channelKind:"email_html"`, `content.source:"raw"`, with
  `subject` / `bodyHtml` / `bodyText`). The visual/Maily editor is dashboard-only — this skill
  emits its own email-safe HTML.
- Variables are Mustache. The raw renderer now supports the Mustache subset (sections
  `{{#x}}…{{/x}}`, inverted `{{^x}}…{{/x}}`, dotted `{{a.b}}`, current-item `{{.}}`; `params`
  takes arbitrary JSON) — but this skill deliberately keeps templates **flat, simple `{{name}}`
  vars** matching `^[a-zA-Z_][a-zA-Z0-9_]*$` for consistent, predictable transactional emails. No
  fallbacks (`{{x,fallback=y}}`), no helpers/partials, and unescaped `{{{x}}}` is not supported. Reach
  for sections/loops only via `ray-integration` when a template genuinely needs them (e.g. a digest).
- **Every top-level `{{var}}` in the final `bodyHtml`/`bodyText`/`subject` becomes a _required_ send
  parameter.** Keep the variable surface small and deliberate (see "Constants vs variables").

**Auth, channels, and the full API:** this skill restates only the template endpoints it needs
(see `references/save-to-ray.md`). For `GET /me`, `GET /channels`, sending, and everything else,
use the **`ray-integration`** skill, or the live spec at `https://ray-api.gege.mn/openapi.json`.
Read `$RAY_API_KEY` from the environment (never hardcode); managing templates needs a **write**
scope.

## Project artifacts — `.ray/email/` (committed)

The brand lives in the consuming repo so it's version-controlled and team-shared:

```
.ray/email/
  layout.html   # the FROZEN shell — bulletproof email HTML with a `<!-- CONTENT -->` slot.
                # Brand constants (logo, colours, footer legal) are baked in as LITERALS.
                # Per-recipient values stay as {{vars}} (default: {{unsubscribe_url}} only).
  brand.md      # tokens (logo URL, palette, font stack, footer text, width), tone/CTA rules,
                # and the explicit constant-vs-variable split decided at setup.
```

If `.ray/email/layout.html` is missing, run **Brand setup** first. If it exists, reuse it verbatim.

## 1 — Brand setup (first run only)

Goal: agree a brand profile, then build the shell **once**.

1. **Derive, don't interrogate.** Scan the repo for signal before asking:
   - existing emails: `emails/`, `templates/`, `mail/`, `**/*.html`, MJML, `react-email`/`@react-email`;
   - brand config: Tailwind theme / CSS custom properties / design tokens, a logo asset.
   Propose a profile: **logo URL, primary + accent colours, font stack, header layout, footer
   text + legal/address, content width** (default 600px).
2. **Logo must be a hosted `https://` URL.** Email can't reliably inline local assets (no CID via
   the API). If only a local file exists, **ask the user for its hosted URL** — don't guess.
3. **Confirm the constant-vs-variable split.** Propose: bake brand constants as literals; keep
   only genuinely per-recipient values as `{{vars}}` — by default just `{{unsubscribe_url}}`.
   Show the user the exact list and let them promote/demote items. This _is_ their required-param
   surface, so make it explicit.
4. **Build the frozen shell** per `references/email-html-rules.md` (bulletproof: tables, inline
   CSS, MSO/VML buttons, dark-mode, preheader, alt text). Write `.ray/email/layout.html` and
   `.ray/email/brand.md`. Tell the user to commit them.

## 2 — Generate (one template, or a batch)

For each template, two modes:
- **new** — from a brief ("password reset email") → write the body copy + structure.
- **restyle** — read an existing email → keep the subject + meaningful copy, **strip its old
  chrome**, and re-flow the content into the brand shell. (This differs from `ray-integration`'s
  `migrate-templates`, which imports existing HTML _as-is_; here we impose the consistent look.)

Inject the body into the frozen shell's `<!-- CONTENT -->` slot to form `bodyHtml`. Never
re-derive the shell — reuse `.ray/email/layout.html` exactly so the set stays consistent.

## 3 — Required outputs per template

Produce all of these (see `references/save-to-ray.md` for the exact `POST /templates` body):
- **`subject`** — the email subject (1–998 chars; may contain `{{vars}}`).
- **`bodyHtml`** — shell + body, fully inlined.
- **`bodyText`** — a **genuine** plain-text alternative that mirrors the HTML and uses the **same
  `{{vars}}`** (not a tag-strip dump). It's a required field and the fallback real clients show.
- **`logTitle` / `logDescription`** — the in-app **feed** entry (≤500 / ≤2000). This is _not_ the
  subject — write it for an in-app notification list (e.g. title `"Password reset requested"`).
- Keep variable names valid (`^[a-zA-Z_][a-zA-Z0-9_]*$`) and minimal.

## 4 — Verify before saving

1. **Local preview.** Fill every `{{var}}` with a plausible **sample** value (infer from the name:
   `first_name`→"Alex", `reset_url`→a fake `https://…`, `unsubscribe_url`→`#`). Write the rendered
   HTML to `/tmp/ray-preview/<name>.html` and open it in the browser. Iterate on layout here —
   it's instant and needs no inbox.
2. **Test-send (real inbox).** Once it looks right, offer a Ray **test-send**
   (`POST /templates/:id/test-send` to an address the user controls). These rows are `is_test` and
   never appear in any customer feed — it's the true Gmail/Outlook rendering check.

## 5 — Save & publish

- **Idempotent upsert by name** (names are unique per workspace): list templates, and if one with
  this name exists, **PATCH its draft**; otherwise **POST a new draft** (`publish:false`). Re-running
  the skill updates the draft instead of erroring on a duplicate name.
- **Print a param manifest** per template: its `required_params` (content vars + any per-recipient
  shell vars like `unsubscribe_url`) and a ready-to-run sample `POST /send` payload, so the
  integrator knows the exact contract.
- **Publish only on explicit confirmation** — ideally after a green test-send. Publishing makes the
  version live and sendable. Use `POST /templates/:id/publish` (or `PATCH … {publish:true}`).

## References
- `references/email-html-rules.md` — the bulletproof email-HTML standard + the frozen-shell anatomy.
- `references/save-to-ray.md` — the slim template-write contract (create/upsert, test-send, publish).
- Anything else (auth, channels, sending) → the `ray-integration` skill or
  `https://ray-api.gege.mn/openapi.json`.
