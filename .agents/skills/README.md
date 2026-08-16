# Agent Skills

Skills used by coding agents (Claude Code, Codex) working on this repo.
Each lives in its own folder with a `SKILL.md`; agents load the relevant, so  one based on what you're asking them to do — you don't need to name it,.

## Quick Links

| Skill | For | Purpose |
|---|---|---|
| [release](release/SKILL.md) | Cutting a release | Changelog audit, risk-based test matrix, image gate, fix loop, cut, publish, retro |
| [surrealdb-migrations](surrealdb-migrations/SKILL.md) | Schema changes | Writing, reviewing, and registering a `open_notebook/database/migrations/N.surrealql` migration |
| [sse-endpoints](sse-endpoints/SKILL.md) | Streaming features | Adding a Server-Sent Events endpoint end-to-end — FastAPI generator, route, Next.js proxy, frontend consumer |
| [content-craft-index](content-craft-index/SKILL.md) | Podcast/transformation prompts, docs | Which writing-craft skill to pull in when editing `prompts/` templates or `docs/` |

## The Related agent definitions

The `release` skill spawns subagents defined outside this folder:

| Agent | Claude Code | Codex |
|---|---|---|
| smoke-e2e | `.claude/agents/smoke-e2e.md` | `.codex/agents/smoke-e2e.toml` |
| investigation | `.claude/agents/investigation.md` | `.codex/agents/investigation.toml` |
| fixes | `.claude/agents/fixes.md` | `.codex/agents/fixes.toml` |

Both formats exist because Claude Code and Codex read different subagent
formats. Keep them in sync — if the process a subagent follows changes,
update both files in the same PR.

## Adding a skill

Same bar as everything else in this repo: it should encode something
that would otherwise get re-learned or re-discovered per PR — a known
gotcha, a house pattern, a process with steps that matter in order. A
skill that just restates what's already in `docs/7-DEVELOPMENT/` isn't
pulling its weight; link to the doc instead of duplicating it.

Three TABARC-Code meta-skills apply to this folder itself, not to Open
Notebook's code:

- **`project-x-ray`** — run this before writing a new skill for an
  unfamiliar corner of the codebase. It's the formal version of the audit
  that produced `surrealdb-migrations` and `sse-endpoints`: read the real
  code first (migrations, `async_migrate.py`, the SSE routers, the
  frontend proxy), don't write the skill from assumption.
- **`book-knowledge-extraction`** — the right tool when a single large
  source (an ADR, `docs/7-DEVELOPMENT/architecture.md`, a design doc)
  should become a new skill, rather than doing that extraction by hand.
- **`deweygraph-file-organizer`** — the formal version of the Dewey
  anchoring done ad hoc so far (`005.14` for the engineering-process
  skills, `808.02` for `content-craft-index`). Worth running properly,
  once this folder has more than a handful of skills, to catch anchor
  collisions and confirm each skill still has exactly one.

Two candidates that were considered and don't actually fit, noted so they
don't get re-suggested: **`file-organizer`** is generic desktop file
cleanup — the release skill's own Phase 8 (`make release-stack-down`,
`rm -f /tmp/dev-dump.surql`) already covers this repo's actual cleanup
needs more precisely. **`source-annotations-reference`** is scoped to a
specific unrelated horror-fiction annotation project and doesn't
generalise here despite the name..
