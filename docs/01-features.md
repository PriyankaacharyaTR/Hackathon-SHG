# 01 — Features

Priority key: **P0** must have, **P1** strong add, **P2** roadmap only (do not build).

## Scope in one sentence
One SHG, one bulk order, order creation → pricing check → plan → allocation → progress → monitoring/replan → quality → payout → group approval, with audit history.

## P0 — Must have

### F-01 SHG and member profiles
- SHG profile: name, village/district, language, leader, member count (10–20 expected).
- Member profile: name, phone (optional), preferred language, availability (hours/day, days), **multi-skill tags** with level (1–3), e.g. cut, stitch, finish, pack, cook, weave.
- Assisted mode flag: a member whose work is entered by another person; the record still names whose work it is and who entered it.

### F-02 Domain templates
- Three templates: **tailoring, food, crafts**. Each defines stages, units, default piece-rates, quality checklist items, typical dependencies.
- Leader can edit a copy of a template per order.

### F-03 Order creation
- Fields: product, quantity, specification, deadline, buyer, agreed/offered price per unit (or total), domain.
- Input by typing or voice (voice is P1 for capture, P0 for the typed form).

### F-04 Pricing check (Pricing agent output + deterministic maths)
- Cost per unit = material + labour + overhead; break-even price; recommended price with target margin.
- Verdict: `SURPLUS`, `BREAK_EVEN`, `LOSS`. Show breakdown and total impact on the order.
- Advises only. The group decides whether to accept the order.

### F-05 AI-drafted plan and timeline
- LLM drafts stages, tasks, dependencies, durations from the template and order. Validated against a JSON schema.
- Saved as `draft`. Leader edits and **publishes**. Nothing is active until published.

### F-06 Skill- and workload-based allocation
- Deterministic rule engine recommends member(s) per task with quantities and a plain-language reason.
- Members accept or flag; leader confirms. Reassignments are logged.

### F-07 Progress tracking
- Members report completed quantity by tap (P0) or voice (P1). Blockers can be raised with a reason.
- Order-level and task-level progress bars; per-member view.

### F-08 Costing
- Estimated cost and actual cost stored separately.
- Cost per unit vs buyer price; surplus / break-even / loss.

### F-09 Payout calculation
- Deterministic engine computes each member's earnings from **accepted** work plus configured surplus share.
- Each line carries an explanation breakdown. Sum of lines equals total payable exactly (no rounding drift).

### F-10 Group approval
- Payout sheet shown to group; approvers record approve / request correction.
- Quorum rule configurable (default: leader + majority of members present).
- Approved payout is locked; later changes need a new correction run.
- Payments are **SIMULATED** (label in UI and exports).

### F-11 Audit history
- Append-only log: who changed assignments, quantities, quality status, rules, payouts, with before/after and timestamp.

### F-12 Role-based access
- Roles: `leader`, `approver` (may equal leader or senior member), `member`.
- Members see only their own record; leader and approvers see group data. Enforced by database row-level security, not only the UI.

## P1 — Strong add

| ID | Feature | Notes |
|----|---------|-------|
| F-13 | Stage-wise quality checks and rework | Per stage: `accepted`, `rework`, `rejected`, with notes and optional photo. A human decides. AI may only flag. |
| F-14 | Delay alerts and replanning | Monitor agent detects at-risk tasks, proposes re-split. Never changes tasks silently. |
| F-15 | Voice progress update | Speak quantity → system repeats it back → member confirms → saved. |
| F-16 | Spoken earnings | Text-to-speech reads a member's assigned/accepted/rework quantities and earnings with the basis, in her language. |
| F-17 | Downloadable report | PDF or Markdown/HTML summary: order, costs, allocation, quality, payout, approvals. |
| F-18 | Configurable fairness rules | Group-approved allocation and payout rules visible to members (see doc 08). |
| F-19 | Offline-tolerant entry | Save locally, sync later (basic queue only). |

## P2 — Roadmap only (do NOT build; mention in pitch)
Supplier discovery, transport, loan referrals, buyer marketplace, payment gateway, advanced image-based quality checks.

## Visible agent moments (must be demo-ready)
1. Pricing agent warns the offered price loses money.
2. Monitor agent proposes a re-split after a member falls behind.
3. Voice agent speaks a member's earnings with the basis.

## Out of scope for the MVP
Real money movement, real KYC, real buyer integrations, vector databases, blockchain, multi-agent frameworks.
