# 10 — Seed Data and Demo Script

All data is **synthetic**. Names are fictional. Replace with real figures after field interviews, but keep this seed for reliable demos.

## 1. Demo scenario
A tailoring SHG with **10 members** receives an order for **300 cloth bags due in 12 days**.

## 2. SHG and members (seed)
SHG: "Demo Mahila SHG" (language `hi-IN`).

| # | Name (fictional) | Role | Skills (level) | Hours/day | Notes |
|---|------------------|------|----------------|-----------|-------|
| 1 | Leela | leader | cut 3, stitch 2, finish 2 | 4 | Leader and approver |
| 2 | Meena | approver | stitch 3, finish 2 | 5 | |
| 3 | Kavya | member | stitch 3 | 6 | **Will lag in the demo** |
| 4 | Radha | member | stitch 2, finish 3 | 4 | |
| 5 | Savitri | member | cut 3 | 5 | |
| 6 | Parvati | member | stitch 2, finish 2 | 4 | Spare capacity for re-split |
| 7 | Geeta | member | finish 3, pack 3 | 4 | |
| 8 | Lakshmi | member | stitch 3, cut 2 | 5 | Assisted mode (low literacy) |
| 9 | Anita | member | stitch 2 | 3 | |
| 10 | Sunita | member | cut 2, finish 2 | 4 | |

Add `auth` users for Leela (leader), Meena (approver), Kavya and Parvati (member) for login demos; others can be assisted/passive.

## 3. Order and cost seed
- Product: Cloth bag, cotton, 12×14 in, logo print, 20 in handle.
- Quantity 300, deadline = today + 12 days, buyer "Demo Buyer Pvt Ltd".
- **Offered price ₹65 (6500 paise)** → triggers the loss warning.
- Estimated per unit: material ₹38, labour ₹30 (cut 6 + stitch 18 + finish 6), overhead ₹4 → ₹72.
- After negotiation in the demo: agreed ₹83 (8300 paise).
- Actual costs to enter later: material ₹11,700 total, overhead ₹1,200 total.

## 4. Scripted progress data (for "Demo mode" button)
Provide admin shortcuts to insert progress entries so the demo does not wait:
- Cutting completes by day 3 (Savitri 150, Leela 100, Sunita 50).
- Stitching: Kavya assigned 70 units, reports only 15 by the check date (→ `AT_RISK`). Others on pace.
- Monitor proposal: move ~35 units from Kavya to Parvati/Meena (derived by the engine, not hard-coded).
- Quality: Radha's finishing batch: 10 units rework → later accepted 6 of them.
- Final state should yield numbers consistent with doc 08 formulas; assert them in tests rather than hard-coding display values.

## 5. 3-minute demo script

| Time | Step | What to show | Say |
|------|------|--------------|-----|
| 0:00 | Problem | Photo of the leader's notebook (from field visit); then speak the order | "Today this is a notebook and memory." |
| 0:25 | **Pricing agent** | Warning: ₹65 loses ₹7 per bag; ₹2,100 on the order; breakdown; recommended ₹83 | "The agent advises; the group decides." |
| 0:55 | Plan + allocation | Draft plan, publish; allocation with reason chips; leader confirms | "AI drafts, the leader approves." |
| 1:25 | **Monitor agent** | Kavya is behind; risk badge; proposed re-split diff; leader approves | "It never changes work silently." |
| 1:55 | Quality | 10 pieces to rework; accepted quantities and costs update | "Pay follows accepted work." |
| 2:15 | **Voice agent** | Member's phone speaks her earnings and the basis in her language | "She can hear exactly how her pay was worked out." |
| 2:40 | Approval | Group approves payout sheet; audit trail; SIMULATED label | "Rules calculate; humans approve." |
| 2:55 | Close | Federation licence + pilot plan (5 SHGs, 1 district, 3 months) | One-line impact statement |

## 6. Backup plan
- Record a full screen capture of the run (with audio) as a fallback.
- `scripts/reset-demo.ts` restores the seeded start state in under 30 seconds.
- Keep a local build that works with `LLM_PROVIDER=mock` returning a stored valid plan.
- Keep a mobile phone with the member login ready and volume tested.

## 7. Likely judge questions (prepare answers)
1. How do you ensure fairness? → configurable group-approved rules; visible basis; audit trail.
2. Why not let the AI compute pay? → reproducibility and trust; AI only proposes and explains.
3. What about low connectivity? → local save and later sync; voice and icons.
4. Who pays? → federations / livelihood programmes; show price per SHG and break-even number of SHGs.
5. How is quality judged? → checklists, human decision, AI only flags.
6. What if members dispute? → show calculation basis, correction run, audit history.
7. Is there a competitor? → re-check before answering; position as the production layer that complements savings, bookkeeping and marketplace tools.
