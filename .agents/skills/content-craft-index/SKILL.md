---
name: content-craft-index
description: Points to the right TABARC-Code writing/craft skill when editing Open Notebook's own AI-generation surface — podcast prompts, transformation prompts, or user-facing docs. Use when editing anything under prompts/, adding a new Transformation or Episode Profile, or writing/reviewing docs under docs/. Dewey: 808.02.
metadata:
  author: TABARC-Code
  dewey_decimal_code: '808.02'
---

# Content Craft Index

Scope note, because it's easy to overclaim this: these skills help you
*write* the `.jinja` templates, transformation presets, and docs well.
They are not loaded by Open Notebook at runtime — the deployed app sends
its prompts to whichever AI provider the end user configured (OpenAI,
Anthropic, Ollama, 18 others), not necessarily Claude. Use these when
you're in a Claude session editing the template text itself; the benefit
compounds into every user's output regardless of which model they run,
because it's baked into the prompt, not into a Claude-only capability.

## Podcast generation — `prompts/podcast/outline.jinja`, `transcript.jinja`

| Editing this... | Pull in |
|---|---|
| Segment structure, flow, short/medium/long sizing | `pacing-time-engine` |
| Speaker dialogue — avoiding the flat, info-dumpy back-and-forth generic podcast AI defaults to | `dialogue-subtext-engine` |
| Speaker persona design — the `backstory`/`personality` fields both templates read per speaker | `character-psychology-engine` |

The templates already ask the model to "match content to speaker
expertise" and vary segment size — that's the right instinct, it's just
thin. These three skills are where the actual craft detail for doing that
well lives; worth a pass next time either template gets touched, not just
when the roadmap's async/live-update work lands.

## Transformations — `prompts/transformation/execute.jinja`, user-authored presets

| Building this preset... | Pull in |
|---|---|
| Professional/expert-voice summary | `professional-voice-engine` |
| Magazine-style long-form summary | `magazine-writer` |

## Reviewing generated output

| Symptom | Pull in |
|---|---|
| A generated transcript, note, or summary reads like typical LLM output — "stands as a testament", rule-of-three, inflated significance | `humanizer` |
| Punctuation density in a generated transcript specifically — matters more here than in ordinary prose, because Episode/Speaker Profile audio is TTS reading the text aloud, and heavy em-dash use reads as an odd pause pattern rather than a visual cue | `em-dash-calibration` |

## Docs — `docs/0-START-HERE/` through `docs/7-DEVELOPMENT/`

| Task | Pull in |
|---|---|
| Terminology consistency across the 8 doc sections (e.g. "Transformation" vs "transformation", "Episode Profile" capitalisation) | `oxford-editorial-indexer` |
| `VISION.md`, an ADR under `decisions/`, or `docs/2-CORE-CONCEPTS/ai-context-rag.md` — anything arguing a technical position about RAG, model choice, or product posture with evidence rather than just asserting it | `llm-white-paper-strategist` |
