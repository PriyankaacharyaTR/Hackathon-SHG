# SHG Fulfilment & Fair Pay Platform — AI Build Pack

Tech4Bharat 2026 | PS 8.1 | Agentic AI. Product name is a working title.

## What this pack is
A set of Markdown specs to hand to an AI coding tool (Claude Code, Cursor, Copilot, etc.) so it can build the hackathon MVP: **one SHG receives one bulk order and manages it from creation to approved payout.**

## How to use it with an AI tool
1. Create an empty repo. Copy this whole folder to `/docs` in the repo.
2. Copy `12-ai-tool-rules.md` to the repo root as `CLAUDE.md` (or `AGENTS.md` / `.cursorrules`).
3. Paste the **Kickoff Prompt** below as the first message.
4. Then work through `05-task-wise-implementation.md` one task at a time. After each task, run its acceptance check before moving on.

## Reading order for the AI
| # | File | Purpose |
|---|------|---------|
| 01 | features.md | What to build, with P0/P1/P2 priority |
| 02 | requirements.md | Functional, non-functional, constraints, stack |
| 03 | api-configurations.md | Env vars, LLM, voice, Supabase, REST API contract |
| 04 | ai-agents-implementation.md | The 5 agents: prompts, schemas, guardrails |
| 05 | task-wise-implementation.md | Ordered build tasks with acceptance criteria |
| 06 | architecture-and-project-structure.md | Folder layout, data flow, module boundaries |
| 07 | database-schema.md | SQL schema, RLS policies, seed notes |
| 08 | business-rules-and-formulas.md | Pricing, allocation, monitoring, payout maths with worked example |
| 09 | ui-ux-voice-spec.md | Screens, low-literacy design, voice flows, i18n |
| 10 | seed-data-and-demo-script.md | Synthetic data and the 3-minute demo |
| 11 | testing-and-acceptance.md | Test cases, golden numbers, definition of done |
| 12 | ai-tool-rules.md | Standing rules for the AI tool (becomes CLAUDE.md) |
| 13 | pitch-and-business-model.md | Non-code: pitch, business model, field validation |

## Kickoff Prompt (paste this first)
```
You are building the SHG Fulfilment & Fair Pay Platform hackathon MVP.
Read every file in /docs before writing code, in the order listed in /docs/00-README-START-HERE.md.
Follow /docs/12-ai-tool-rules.md strictly.
Non-negotiable principle: AI proposes, rules calculate, humans approve.
Money maths, allocation and monitoring are deterministic code, never LLM output.
Start with Task 0 and Task 1 in /docs/05-task-wise-implementation.md.
Stop after each task, show me what changed, and wait for my confirmation.
Use synthetic data only and label all payments "SIMULATED".
```

## Core principle (repeat everywhere)
**AI proposes. Rules calculate. Humans approve.**

## Things to confirm before the pitch
- Event rules and submission format (the event page could not be read when this pack was prepared; check it directly).
- Voice language support for the pilot language (verify, do not assume).
- "No competitor" claim for production-coordination tools (re-check before claiming it).
- Real numbers from 1–2 SHG leader interviews to replace synthetic figures.
