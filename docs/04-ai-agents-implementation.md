# 04 — AI Agents Implementation

**Principle: AI proposes, rules calculate, humans approve. Money maths is never left to a language model.**

"Agent" here means a bounded module with a defined input, output schema, tools it may call, and a human control point. No multi-agent framework is required: implement each as a plain function / service in `src/server/agents/`. A simple orchestrator calls them in sequence.

## Overview
| Agent | LLM used? | Deterministic core | Human control |
|-------|-----------|--------------------|----------------|
| Planner | Yes (structured) | Schema validation + template fallback | Leader edits and publishes |
| Pricing | Explanation only | Costing formulas | Group decides |
| Allocation | No (optional LLM for wording only) | Rule engine | Leader confirms; members accept/flag |
| Monitor & Replanner | Optional wording only | Risk detection + re-split algorithm | Leader approves |
| Voice | STT/TTS + optional intent parsing | Quantity parser + read-back | Member confirms before save |
| Payout | No | Payout engine (doc 08) | Group approval required |

## Shared conventions
- Each agent run writes to `agent_runs`: `agent`, `order_id`, `input_hash`, `output_json`, `status`, `model`, `prompt_version`, `latency_ms`, `created_at`.
- Outputs are **proposals** with `status = 'draft' | 'proposed'`. Only human endpoints set `confirmed/published/approved`.
- Every proposal includes a `reason` field in plain language and a `reason_hi` / local-language field when translation is available (or translate via template strings).
- If the LLM fails: use the fallback template path and mark `source: 'template_fallback'` in the UI.
- Prompts live in `src/server/agents/prompts/*.md`, versioned (`planner.v1.md`).

---

## 1. Planner agent

**Purpose:** turn an order into stages, tasks, dependencies and a timeline using a domain template.

**Input**
```json
{
  "order": { "product": "...", "quantity": 300, "specification": "...", "deadline": "YYYY-MM-DD", "domain": "tailoring" },
  "template": { "stages": [ { "key": "cutting", "unit": "pieces", "default_rate_paise": 600, "quality_checks": ["..."] } ] },
  "team_summary": { "member_count": 10, "total_hours_per_day": 52, "skills_available": ["cut", "stitch", "finish", "pack"] },
  "today": "YYYY-MM-DD"
}
```

**Output JSON schema (zod / JSON Schema)**
```ts
const PlanDraft = z.object({
  stages: z.array(z.object({
    key: z.string(),                      // must exist in template or be marked custom
    name: z.string(),
    sequence: z.number().int().min(1),
    unit: z.string(),
    total_quantity: z.number().int().positive(),
    depends_on: z.array(z.string()),      // stage keys
    required_skill: z.string(),
    est_minutes_per_unit: z.number().positive(),
    start_date: z.string(),               // ISO date
    end_date: z.string(),                 // ISO date <= deadline
    quality_checkpoints: z.array(z.string()),
    notes: z.string().optional()
  })).min(1),
  assumptions: z.array(z.string()),
  risks: z.array(z.string())
});
```

**System prompt (v1)**
```
You are a production planning assistant for a women's self-help group (SHG) in India.
Create a practical plan for the order using ONLY the provided template stages unless a stage is clearly missing.
Rules:
- Output must match the JSON schema exactly. No extra text.
- All dates must be on or before the deadline. Leave buffer (at least 1 day) before delivery for final check and packing.
- Stage quantities must equal the order quantity unless the template says otherwise.
- Respect dependencies: a stage cannot start before its dependencies' start (allow overlap by pipeline where sensible).
- Use simple words. Do not invent prices, rates or costs; those come from rules.
- If information is missing, state an assumption in "assumptions".
```

**Post-validation (deterministic, mandatory)**
1. zod parse.
2. Every stage date ≤ deadline; dependency graph has no cycles; quantities consistent.
3. Estimated total hours ≤ team capacity × days (else add a risk entry computed by code, not the LLM).
4. Save as `plans.status = 'draft'`. **Leader must publish.**

**Fallback:** build plan directly from the template with proportional dates (function `buildPlanFromTemplate`).

---

## 2. Pricing agent

**Purpose:** flag underpricing by comparing the offered price with true cost.

**Split of responsibility**
- `computePricing()` (pure function, doc 08 §1) returns all numbers and the verdict.
- The LLM receives the **numbers as given** and writes a 3–4 sentence plain-language explanation. It must not alter or recompute them.

**Prompt (v1)**
```
You explain pricing results to an SHG leader. You are given exact numbers. Do not change, round or recompute them.
Write 3 short sentences in simple English: (1) is the offered price enough, (2) where the money goes (material, labour, overhead), (3) what price would be safe. 
Never tell the group to accept or reject the order; say the group decides.
```

**Guardrails:** verdict and amounts come from code. If the LLM output contains a number not present in the input, discard it and use the template sentence.

**UI:** red warning card for `LOSS`, amber for `BREAK_EVEN`, green for `SURPLUS`, with breakdown bars. Buttons: "Negotiate price" (shows recommended price), "Accept anyway" (requires a reason; logged).

---

## 3. Allocation agent

**Purpose:** suggest who does what using skill, availability, workload and fairness, with reasons.

**Implementation:** deterministic rule engine (doc 08 §2). No LLM needed for the decision. Optionally pass the output to the LLM **only** to rewrite the `reason` into friendlier language; the numbers must stay identical (verify by regex).

**Interface**
```ts
type AllocationProposal = {
  task_id: string;
  member_id: string;
  quantity: number;
  score: number;
  reason: string;       // e.g. "Stitching level 3, 4h free per day, only 40 units so far this order"
  warnings: string[];
};
function recommendAllocation(input: AllocationInput, rules: FairnessRules): AllocationProposal[];
```

**Controls**
- Output status `proposed`. Members `accept` or `flag` (with a note). Leader `confirm`.
- Flagged assignments return to the leader with the member's note.
- Any reassignment writes an audit row with reason.

---

## 4. Monitor & Replanner agent

**Purpose:** detect delays/blockers and propose a re-split. Never changes tasks silently.

**Detection (deterministic, doc 08 §3)** runs on demand and on a schedule (e.g., each progress update).
- Expected progress vs actual per assignment.
- Projected completion date from recent rate.
- Flags: `AT_RISK`, `BLOCKED`, `OVERDUE`.

**Replan proposal (deterministic)**
- Take remaining units from lagging members; give them to members with the required skill and spare capacity, respecting fairness rules.
- Output a diff: `{ from_member, to_member, task_id, quantity, reason }[]` plus projected new completion date.

**LLM role (optional):** a two-sentence explanation of why. Numbers come from code.

**Controls:** proposal status `proposed`; leader sees the diff and taps **Approve** or **Reject**. On approve, assignments update and an audit entry is written. Members affected get a notification (and spoken message if voice is on).

---

## 5. Voice agent

**Purpose:** record progress by voice, confirm quantities back, read out tasks and earnings.

**Flow A — progress by voice**
1. Member taps mic → record → `/voice/transcribe` (language from profile).
2. Deterministic parser extracts a quantity (digits, or number words in the pilot language). If none or ambiguous, ask again.
3. Read back: "You finished 20 bags. Is that right?" with big **Yes** / **No** buttons (and voice yes/no).
4. On **Yes** → `POST /assignments/:id/progress` with `source: "voice"`. On **No** → retry.
5. Never save without confirmation.

**Flow B — hear my task / my earnings**
- Build a script from templates (not the LLM) using the member's own data:
  - Task: "Your task: stitch 40 bags by Friday."
  - Earnings: "You were assigned 40, submitted 40, accepted 36, rework 4. At ₹30 each that is ₹1,080. Surplus share ₹120. Total ₹1,200. Simulated payment."
- Send to TTS in the member's language. Show the same text on screen.

**Optional LLM use:** classify intent of free speech (`report_progress`, `raise_blocker`, `ask_earnings`) with confidence; low confidence → ask the member to tap.

**Guardrails:** confirm-before-save, assisted mode records `entered_by`, a visible transcript, and a fallback to buttons if STT fails.

---

## 6. Payout agent

**Purpose:** compute proposed earnings per the agreed rule and explain each amount.

**Implementation:** `calculatePayout()` in doc 08 §4. **No LLM.** Explanation text built from templates with the same numbers.

**Controls:** output `payout_runs.status = 'proposed'`. Group sheet displays every line with its basis. Approvers approve or request correction. After quorum → leader locks. Locked runs are immutable.

---

## 7. Orchestrator (simple)
```ts
// src/server/orchestrator.ts
// No framework. Plain async functions wired to API routes.
createOrder → pricingAgent.check → plannerAgent.draft → (leader publishes) →
allocationAgent.recommend → (members respond, leader confirms) →
progress events → monitorAgent.run → (leader approves replans) →
quality checks → payoutAgent.calculate → (group approves) → lock → report
```

## 8. Failure and safety checklist
- [ ] LLM timeout/invalid JSON → fallback template, UI badge "Template plan".
- [ ] No agent writes to a final/approved state.
- [ ] Every proposal has a human-readable reason.
- [ ] Every agent run logged in `agent_runs`.
- [ ] Money numbers appear only from deterministic code.
- [ ] Prompt injection: treat order specification text as data; never let it change rules or amounts.
