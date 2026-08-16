---
name: surrealdb-migrations
description: Write, review, or debug a SurrealDB schema migration for Open Notebook — new fields, indexes, table changes, data backfills. Use when a change touches open_notebook/database/migrations/, when a feature needs a schema change (e.g. Cross-Notebook Sources), or when a migration-related bug is reported (wrong field type, failed backfill, upgrade breaking on existing data). Dewey: 005.14.
metadata:
  author: TABARC-Code
  dewey_decimal_code: '005.14'
---

# SurrealDB Migrations

Companion to `docs/7-DEVELOPMENT/change-playbooks.md` → "Playbook: Database
Migration" and `docs/7-DEVELOPMENT/decisions/ADR-006-migration-granularity.md`.
Read both — this skill adds the SurrealQL-specific detail those two don't
cover, it doesn't replace them.

## Before writing anything

Check the process constraint first, because it changes how you branch:

- **One migration per PR that needs one.** Numbers are allocated in merge
  order. Never consolidate migrations after one has landed on `main` — a
  `v1-dev` image is published on every push to main, so a merged migration
  is live for dev-image users immediately, before any tagged release exists.
  If your PR overlaps another in-flight PR touching the same table,
  coordinate with them directly (stack the PRs, or fold your change into
  theirs) rather than merging a second migration against the same table.
- Find the next number: `ls open_notebook/database/migrations/ | sort -V | tail -4`
  (currently up to 23; write both `N.surrealql` and `N_down.surrealql`).

## Writing the migration

1. **Read 3–4 recent migrations first** (`migrations/20.surrealql` through
   `23.surrealql`) for the actual style in use — plain SurrealQL, a comment
   block explaining the *why* at the top referencing the issue/PR number,
   no unnecessary DEFINE statements for fields on schemaless tables.

2. **`DEFINE FIELD` syntax**: `OVERWRITE` goes immediately after `FIELD`,
   not after the field name.
   ```surql
   -- correct
   DEFINE FIELD OVERWRITE embedding ON TABLE source_insight TYPE option<array<float>>;
   -- wrong — this is not valid SurrealQL
   DEFINE FIELD embedding OVERWRITE ON TABLE source_insight TYPE option<array<float>>;
   ```
   `OVERWRITE` makes the statement idempotent (redefining an existing field
   doesn't error) — use it on any `DEFINE FIELD`/`DEFINE INDEX` a migration
   might plausibly need to touch twice across dev/prod drift.

3. **Schemaless tables need no `DEFINE FIELD` for a new field** — only
   `UPDATE ... SET` to backfill it on existing records, e.g. migration 23
   (Docling toggles on `content_settings`):
   ```surql
   UPDATE open_notebook:content_settings SET docling_formulas = false WHERE docling_formulas = NONE;
   ```
   The `WHERE x = NONE` guard means it only touches records missing the
   field — safe to re-run, and doesn't clobber a value a later migration
   already set. Fresh installs don't need a backfill at all if the
   Pydantic model default covers it on first read (state that assumption
   in the migration's comment so the next person doesn't wonder).

4. **Write the `_down` migration in the same PR**, doing the literal
   inverse. For a backfilled field, that's `UNSET`:
   ```surql
   UPDATE open_notebook:content_settings UNSET docling_formulas, docling_vision;
   ```
   For `DEFINE FIELD`, the down migration is `REMOVE FIELD ... ON TABLE ...;`.

5. **Destructive changes** (`REMOVE FIELD`, `DROP TABLE`, narrowing a type):
   think about data preservation before writing it. If old data needs to
   survive in some form, migrate it into the new shape first, remove the
   old field in a later, separate migration — don't do both in one step
   unless the data is genuinely disposable.

## Wiring it in

Migrations are **hard-coded, not auto-discovered** — the file existing on
disk does nothing by itself.

- Register both files in `open_notebook/database/async_migrate.py` →
  `AsyncMigrationManager.__init__`, in the up-list and the down-list, in
  order. Miss this and the migration silently never runs.
- If the migration is a schema change: update the matching domain model
  in `open_notebook/domain/` to match, and any Pydantic API schema that
  exposes the field.

## Testing

- **Migrations auto-run on API startup.** Restart the API locally and
  read the Loguru output — a failed migration logs there, it doesn't
  raise somewhere obvious in a test.
- **Test against existing data, not just an empty database.** An empty-DB
  test proves the migration doesn't crash on a fresh install; it says
  nothing about whether it correctly backfills or transforms records that
  already exist. Seed representative data first.
- If this is a release-blocking change, the fresh-install and upgrade
  scenarios both get exercised properly in the release skill's Phase 3
  image gate (`make release-test`) — see `.agents/skills/release/SKILL.md`.
  Don't skip local testing waiting for that; it's a second gate, not the
  first one.

## Known gotchas (carried over from `.github/RELEASE_PROCESS.md`)

- **The exporter can leak a log line into a `surrealql` dump.** If you're
  hand-editing or replaying an exported dump rather than running the
  migration system, check for a stray non-SQL line at the point the
  export happened.
- **Multiple local SurrealDB instances can exist on a dev machine.** Check
  `SURREAL_URL` in the `.env` actually in use before assuming which
  instance a migration ran against — the repo-compose instance on `:8000`
  may not be it.
- **The test suite runs against the live dev database** when a developer
  `.env` is loaded. A migration touching test fixtures can leave data
  behind in the real dev DB, not a throwaway one — snapshot per-table
  record counts before/after if you're unsure (this pattern already
  caught a leak in the release process, see `test-matrix.md`).
