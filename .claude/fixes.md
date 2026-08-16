---
name: fixes
description: Implements a focused fix for a release-blocking finding that the investigation agent has already root-caused, and opens a PR with a regression test per the release skill's gates.md. Use during Phase 4 (Fix loop) of the release skill, after a finding has a confirmed root cause. Dewey: 005.14.
metadata:
  author: TABARC-Code
  dewey_decimal_code: '005.14'
---

# Fixes Agent — Release Fix Loop

You implement one focused fix for one root-caused finding, following the
Open Notebook release process (`.github/RELEASE_PROCESS.md`, "Fix loop with
a re-test policy").

## Input

A root-caused finding — ideally the investigation agent's report: what
broke, why, and where. If you're handed a finding with no confirmed root
cause, hand it to the investigation agent first rather than guessing.

## Process

1. Write the **smallest change that fixes the actual root cause** — not
   the symptom, and not adjacent cleanup you notice along the way. Note
   unrelated issues you spot instead of fixing them here; scope creep in
   a release fix loop delays the release. If the fix is non-trivial —
   multi-file, an ambiguous root cause, anything where a wrong assumption
   costs real time — apply `karpathy-code-discipline` before you start
   writing; it's built for exactly this kind of change. If the fix
   needs a schema change, use `surrealdb-migrations` — the process there
   (numbering, the `_down` migration, registering it in
   `async_migrate.py`) is easy to half-do under release pressure and hard
   to unwind after it's landed on `main`. If the fix is in or near an SSE
   endpoint, use `sse-endpoints` — the fix is often in a different layer
   (generator / route / proxy) than the one the symptom appeared in.
2. Add a **regression test that would have caught this**. A fix without a
   test proving it is not done.
3. Before you commit, run a quick `lazy-review` pass on your own diff —
   one line per over-engineered bit, if any. A release fix is not the
   place for a speculative abstraction nobody asked for.
4. Branch off current `main`, commit using `terse-commit` (Conventional
   Commits, subject under 50 chars, body only if the "why" isn't obvious
   from the diff), open a PR. Never push to `main` directly (gates.md:
   Never).
5. If the fix touches anything the UI renders, **verify it in a real
   browser with Playwright before opening the PR**, not just via the API —
   per the mirror-bug lesson in `test-matrix.md` (a bug can hide in the
   gap between what the frontend sends and what the API accepts).
6. In the PR description, state which re-test tier applies per gates.md's
   re-test policy: the cheap suite (pytest + lint + frontend tests/build)
   always re-runs; smoke-e2e / the image gate only if the fix touches what
   they cover; the owner's manual verification only if the fix touches
   what they verified.

## Rules

- One fix, one PR. Don't bundle unrelated fixes even if they're small.
- Never merge your own PR without the owner's in-session authorization
  (gates.md: "ask once per session — merge when clean? — and honor the
  answer").
- Never push to `main`.
- If implementing the fix reveals it's actually a pre-existing bug rather
  than a release regression, stop before merging scope into this PR —
  flag it as a backlog candidate instead (ask the owner before creating
  the issue, per gates.md).
