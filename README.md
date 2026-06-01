# ray-skills

Reusable Claude Code **skills** for integrating projects with [Ray](https://ray-api.gege.mn),
our notification/email platform. Drop a skill into any project and Claude can send through Ray
or migrate existing templates into it — using the live API docs as the source of truth.

## Skills

| Skill | Use it for |
|---|---|
| `ray-integration` | Sending notifications/emails via Ray, and creating/migrating email templates into Ray. |

## Install

A Claude Code skill is just a directory containing `SKILL.md`. Pick one:

**Per project** — make it available in one repo:
```bash
ln -s ~/Projects/ray-skills/skills/ray-integration \
      /path/to/your-project/.claude/skills/ray-integration
# (or copy instead of symlink)
```

**All projects** — make it available everywhere:
```bash
ln -s ~/Projects/ray-skills/skills/ray-integration \
      ~/.claude/skills/ray-integration
```

Then, in a project, just ask Claude things like *"migrate our email templates to Ray"* or
*"send a welcome email through Ray"* — the skill activates from its `description`.

## Prerequisite

Set a Ray API key in the environment before using it (the skill never hardcodes it):
```bash
export RAY_API_KEY=…   # from Ray dashboard → workspace → API keys (write scope to manage templates)
```

## How it works

- `SKILL.md` is the lean entry point Claude loads; details live in `references/`.
- The authoritative API contract is always the live spec at
  `https://ray-api.gege.mn/openapi.json` — the skill tells Claude to fetch it rather than
  rely on anything baked in, so it can't go stale.

## Layout
```
skills/ray-integration/
  SKILL.md                      # entry point (name + description + overview)
  references/
    api.md                      # condensed endpoint/auth/contract reference
    migrate-templates.md         # step-by-step template migration playbook
```

Add more skills under `skills/<name>/SKILL.md` as Ray grows.
