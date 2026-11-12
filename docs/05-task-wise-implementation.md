# 05 — Task-wise Implementation Plan

Work **one task at a time**. After each task: run its acceptance check, commit, then stop and report. Priorities: P0 first, then P1, never P2.

Team context: small team (about two people), hackathon timeline. Finale: 27 Dec 2026, Bengaluru. Build the payout engine first.

Legend: 🔴 P0 · 🟡 P1 · ⚪ stretch

---

## Phase A — Foundation

### Task 0 — Project bootstrap 🔴
- Init repo: Vite + React + TypeScript, Tailwind, ESLint, Prettier, Vitest.
- Decide backend: Supabase Edge Functions (default). Record the decision in `docs/DECISIONS.md`.
- Add `.env.example` (doc 03), config loader with zod validation.
- Folder structure per doc 06.
- **Accept:** `npm run dev` shows a blank app; `npm test` runs; config fails fast on missing env.

### Task 1 — Money and core domain library 🔴
- `src/domain/money.ts`: integer paise helpers, `formatINR`, `allocateByLargestRemainder`.
- Types for Order, Stage, Task, Assignment, ProgressEntry, QualityCheck, PayoutRun, PayoutLine.
- **Accept:** unit tests for rounding: distributing 10,000 paise across weights [1,1,1] sums to exactly 10,000.

### Task 2 — Database schema, RLS, seed 🔴
- Apply doc 07 migrations. Enable RLS everywhere. Create seed script (doc 10).
- **Accept:** logging in as a member returns only her own assignments and payout line; leader sees all. Proven by an automated test.

---

## Phase B — Deterministic engines (build before any UI polish)

### Task 3 — Payout engine 🔴
- Implement `calculatePayout()` (doc 08 §4) with explanation objects.
- **Accept:** golden test from doc 08 worked example passes exactly; total of lines = total payable; rework/rejected units earn nothing.

### Task 4 — Pricing engine 🔴
- `computePricing()` (doc 08 §1).
- **Accept:** 300 bags @ ₹65 → verdict `LOSS`, impact −₹2,100, recommended ₹83.

### Task 5 — Allocation engine 🔴
- `recommendAllocation()` (doc 08 §2) with configurable weights, reasons, shortfall warnings.
- **Accept:** property tests: no member exceeds capacity; sum of quantities per stage equals stage total (or shortfall flagged); deterministic with same inputs.

### Task 6 — Monitor and re-split engine 🟡
- `detectRisks()` and `proposeResplit()` (doc 08 §3).
- **Accept:** seeded lagging member triggers `AT_RISK`; proposal moves units only to members with the skill and capacity; totals preserved.

---

## Phase C — Core product flow (P0 UI)

### Task 7 — Auth and role shell 🔴
- Supabase auth, role-aware routing: leader dashboard vs member workspace.
- **Accept:** each role lands on the right home screen; unauthorised routes redirect.

### Task 8 — SHG and member profiles 🔴
- CRUD members with multi-skill tags (level 1–3), availability, language, assisted-mode flag.
- **Accept:** add a member with 3 skills; shown in allocation candidates.

### Task 9 — Templates and order creation 🔴
- Seed templates (tailoring, food, crafts); order form; order detail page.
- **Accept:** create the cloth-bag order from doc 10.

### Task 10 — Pricing check UI 🔴
- Cost inputs form, call pricing endpoint, warning card, "Negotiate price" and "Accept anyway (reason)".
- **Accept:** loss warning appears with breakdown; override is audited.

### Task 11 — LLM client and Planner agent 🔴
- Implement `LlmClient`, planner prompt v1, zod validation, fallback template, `agent_runs` logging.
- Draft plan screen: editable stages/tasks, "Publish".
- **Accept:** valid plan generated; with LLM disabled the template fallback works; nothing active until published.

### Task 12 — Allocation UI and acceptance loop 🔴
- Recommend → show proposals with reasons → members accept/flag → leader confirm.
- **Accept:** flagged assignment returns to leader with note; reassignment audited.

### Task 13 — Progress tracking 🔴
- Member tap entry (big +/− and confirm), blocker button, progress bars (task and order).
- Assisted entry records `entered_by`.
- **Accept:** progress updates reflect in leader dashboard immediately.

### Task 14 — Costing (estimated vs actual) 🔴
- Actual cost entry; cost per unit vs price; surplus/break-even/loss.
- **Accept:** estimated and actual stored separately; variance shown.

### Task 15 — Payout sheet and group approval 🔴
- Calculate → sheet with explanation per line → approvers approve/request correction → quorum → lock.
- Label "SIMULATED PAYMENT".
- **Accept:** locked run is immutable; correction creates a new run linked to the old.

### Task 16 — Audit log view 🔴
- Append-only log, filter by entity/user/date.
- **Accept:** assignment change, quantity change, quality change and payout actions all appear with before/after.

---

## Phase D — Strong adds (P1)

### Task 17 — Stage-wise quality and rework 🟡
- Checklist per stage, accepted/rework/rejected counts, notes, optional photo to private storage.
- Rework loops back to the assignee; payout uses accepted only.
- **Accept:** rework quantity changes accepted total and the payout preview.

### Task 18 — Monitor UI and replanning 🟡
- Risk badges; proposal diff view; Approve/Reject.
- **Accept:** demo step 4 works end to end.

### Task 19 — Voice progress update 🟡
- Mic capture, STT call, quantity parser, read-back confirm, save with `source: voice`.
- **Accept:** spoken "20" in the pilot language is read back and saved only after confirmation; fallback to buttons if STT fails.

### Task 20 — Spoken earnings and tasks 🟡
- Template-built script, TTS in member language, on-screen text mirrors audio.
- **Accept:** demo step 6 works on a phone.

### Task 21 — Configurable fairness rules 🟡
- Rules screen (weights, surplus split mode, group fund %, rework policy); group approval of rules; visible to members.
- **Accept:** changing a rule changes allocation/payout output and is audited.

### Task 22 — Report download 🟡
- Generate PDF or HTML/MD report of the order, costs, allocation, quality, payout, approvals.
- **Accept:** downloaded report matches on-screen numbers.

### Task 23 — Offline-tolerant entry ⚪
- Queue progress entries locally; sync on reconnect; show "saved, will sync".

---

## Phase E — Polish and demo readiness

### Task 24 — i18n and low-literacy pass 🔴
- Externalise strings; pilot language; icons with labels; large buttons; contrast; TTS on key numbers.
- **Accept:** member workspace usable without reading English text.

### Task 25 — Demo hardening 🔴
- Seed script resets to demo start state; "Demo mode" shortcuts (advance time, simulate member lag); record backup video.
- **Accept:** the 3-minute script (doc 10) runs twice in a row without manual DB fixes.

### Task 26 — Testing and docs 🔴
- Complete doc 11 checklist; README with setup, screenshots, architecture.
- **Accept:** all tests green; definition of done met.

### Task 27 — Pitch support (non-code) 🔴
- Interview 1–2 SHG leaders; replace synthetic numbers where possible.
- Revenue slide: price per SHG, cost to serve, break-even SHG count, pilot plan (e.g. 5 SHGs, 1 district, 3 months).
- Re-check the "no competitor" claim.
- Prepare Q&A and rehearse.

---

## Suggested order if time is tight
Tasks 0→1→3→4→5 → 2 → 7→9→10→11→12→13→14→15→16 (**P0 complete**) → 18→17→19→20 (the three visible agent moments) → 24→25→26.

## Per-task prompt template for the AI tool
```
Implement Task <N> from /docs/05-task-wise-implementation.md.
Read the referenced sections of docs 02–10 first.
Write tests alongside the code. Do not touch unrelated files.
When done, list files changed, how to run the acceptance check, and any open questions.
```
