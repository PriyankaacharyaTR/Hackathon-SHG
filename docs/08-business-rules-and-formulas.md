# 08 — Business Rules and Formulas

All logic here is **deterministic code** in `src/domain/`. No LLM involvement. All money is integer paise; never use floats for money. Use `Math.round`/integer division only at defined points and always reconcile with largest-remainder allocation.

---

## 1. Pricing (`computePricing`)

**Inputs:** quantity `Q`, offered price per unit `P`, per-unit estimated cost: `material m`, `labour l`, `overhead o`, target margin `g` (default 15%).

```
cost_per_unit      = m + l + o
total_cost         = Q × cost_per_unit
revenue_offered    = Q × P
total_impact       = revenue_offered − total_cost
break_even_price   = cost_per_unit
recommended_price  = ceil_to_rupee( cost_per_unit × (100 + g) / 100 )   // 100 paise multiple, round UP
verdict:
   P <  cost_per_unit  → LOSS
   P == cost_per_unit  → BREAK_EVEN
   P >  cost_per_unit  → SURPLUS   (also report margin_pct = (P − cost)/cost)
   Optional: SURPLUS but margin_pct < g → "LOW_MARGIN" flag
```
Labour per unit `l` = sum over stages of `rate_paise_per_unit` (piece rates).

**Golden example (cloth bags):** m=3800, l=3000, o=400 → cost/unit 7200. Q=300, P=6500 → revenue 1,950,000; total cost 2,160,000; impact **−210,000 paise (−₹2,100)**; verdict `LOSS`; recommended = ceil_to_rupee(7200×1.15 = 8280) = **8300 (₹83)**.

---

## 2. Allocation (`recommendAllocation`)

**Inputs:** stages (with `required_skill`, `total_quantity`, `est_minutes_per_unit`, dates), members (skills with level, `hours_per_day`, available days, active), existing load, fairness rules.

**Hard constraints (must pass or member is excluded)**
1. Member has `required_skill` (level ≥ 1).
2. Member active and available on at least one working day in the stage window.
3. Capacity > 0.

**Capacity (units)**
```
working_days      = count of available days between stage start and end (inclusive)
hours_available   = working_days × hours_per_day × utilisation   // utilisation default 0.8 (buffer)
already_load_min  = minutes already allocated in overlapping stages
capacity_units    = floor( (hours_available × 60 − already_load_min) / est_minutes_per_unit )
```

**Score (0–1), default weights** `w_skill=0.4, w_capacity=0.3, w_fair=0.3` (configurable):
```
skill_score    = level / 3
capacity_score = capacity_units / max(capacity_units over eligible members)
fair_score     = 1 − (member_allocated_value_share / equal_share)  clipped to [0,1]
                 // member_allocated_value_share = member's allocated piece-rate value this order ÷ total
                 // equal_share = 1 / eligible_member_count
score = w_skill×skill_score + w_capacity×capacity_score + w_fair×fair_score
```

**Quantity split per stage**
1. `weight_i = score_i × capacity_units_i` for eligible members.
2. Initial `qty_i = Q × weight_i / Σweight`, capped at `capacity_units_i`.
3. Redistribute capped overflow to remaining members by weight; repeat until placed or no capacity left.
4. Convert to integers via largest remainder so Σ `qty_i` = Q (or = total capacity if short).
5. If total capacity < Q: output a **shortfall warning** (units, stage) — never silently drop.
6. Optional rule: `max_share_per_member` (default 40% of any stage) to avoid one person dominating; `min_batch` (default 10 units) to avoid trivial assignments.

**Reason string** is template-built from the actual numbers, e.g.
`"Stitching level 3 · 4 h/day free · 40 units so far (fair share) → 60 units"`.

**Order of processing:** by stage `sequence`; update `already_load` after each stage.

---

## 3. Monitoring and re-split

**Per assignment on day `t`:**
```
window_days         = working days from stage start to end
elapsed_days        = working days elapsed (capped at window_days)
expected_qty        = assigned_qty × elapsed_days / window_days
actual_qty          = Σ progress_entries.quantity
progress_ratio      = actual_qty / expected_qty            (if expected_qty = 0 → 1)
recent_rate         = units/day over the last 2 working days (fallback: average so far)
projected_finish    = today + ceil( (assigned_qty − actual_qty) / max(recent_rate, ε) ) working days
```
**Flags**
- `BLOCKED`: open blocker exists.
- `OVERDUE`: today > end_date and remaining > 0.
- `AT_RISK`: projected_finish > end_date **or** (elapsed_days ≥ 25% of window and progress_ratio < 0.75).

**Re-split (`proposeResplit`)**
1. For each flagged assignment, `shortfall = remaining units − what she can finish by end_date at recent_rate`.
2. Candidates = members with the required skill and spare capacity (same capacity formula using remaining window), excluding flagged ones.
3. Distribute `shortfall` using the same weighting as allocation (§2), respecting fairness rules.
4. Output diff rows `{from_member, to_member, stage_id, quantity, reason}`, plus `projected_end_date`.
5. If candidates cannot absorb it: output `still_at_risk: true` with the remaining shortfall and suggest extending the deadline or reducing quantity (human decision).
6. **Never apply automatically.**

---

## 4. Payout (`calculatePayout`)

**Principle:** pay for **accepted** work. Units in `rework` or `rejected` earn nothing until they become accepted.

**Inputs:** accepted units per member per stage, stage `rate_paise_per_unit`, order revenue, actual costs, fairness rules.

**Steps**
```
1. piece_earnings_i = Σ_stage ( accepted_qty_i,stage × rate_stage )
2. revenue          = Q_delivered_accepted × agreed_price_per_unit      // only units accepted by buyer; for demo = accepted final units
3. direct_costs     = actual material + actual overhead + other actual costs
4. total_piece      = Σ piece_earnings_i
5. surplus          = revenue − direct_costs − total_piece
6. if surplus < 0 → shortfall mode (see below)
7. group_fund       = round_half_up( surplus × group_fund_pct / 100 )          // default 10%
8. distributable    = surplus − group_fund
9. surplus_share_i  = distributable × (basis_i / Σ basis)                       // basis by mode below
10. Round all surplus_share_i with largest remainder so Σ = distributable exactly
11. total_i = piece_earnings_i + surplus_share_i
12. total_payable = Σ total_i ;  check: total_payable + group_fund + direct_costs = revenue
```
**Surplus share modes (group-selected)**
- `proportional` (default): basis_i = piece_earnings_i.
- `equal`: basis_i = 1 for every member with accepted work > 0.
- `hybrid`: `equal_pct`% equal + rest proportional.

**Rework / rejection policy (group-selected)**
- `unpaid_until_accepted` (default): no pay for rework/rejected units. Rejected material loss is a group cost (added to `direct_costs`), not deducted from the member.
- `member_bears_rejected_material` (optional, must be explicitly approved; show warning).

**Leader coordination allowance (optional):** fixed paise or % of revenue taken before surplus; must be visible in the sheet.

**Shortfall mode (surplus < 0):** piece earnings are paid in full if funds allow; the shortfall is shown as `group_loss` and flagged to approvers; do not silently reduce member pay. Group decides (e.g., pay pro-rata) via an explicit approved rule.

**Explanation object per line**
```json
{
  "lines": [
    { "stage": "stitching", "accepted": 150, "rate_paise": 1800, "amount_paise": 270000 },
    { "stage": "finishing", "accepted": 100, "rate_paise": 600,  "amount_paise": 60000 }
  ],
  "piece_total_paise": 330000,
  "surplus_basis": "proportional",
  "surplus_share_paise": 99000,
  "total_paise": 429000,
  "assigned": 250, "submitted": 250, "accepted": 250, "rework": 0, "rejected": 0
}
```
The spoken and on-screen explanation are generated **from this object** with templates.

### Golden payout example A (use as unit test)
Order: 300 bags at agreed ₹83 (8300 paise). Rates: cutting 600, stitching 1800, finishing 600. Group fund 10%. Mode `proportional`.

| Member | Accepted units | Piece earnings (paise) |
|--------|----------------|------------------------|
| A | cutting 300, finishing 100 | 180,000 + 60,000 = **240,000** |
| B | stitching 150, finishing 100 | 270,000 + 60,000 = **330,000** |
| C | stitching 150, finishing 100 | 270,000 + 60,000 = **330,000** |

- total_piece = 900,000
- revenue = 300 × 8300 = **2,490,000**
- direct costs = material 1,170,000 + overhead 120,000 = 1,290,000
- surplus = 2,490,000 − 1,290,000 − 900,000 = **300,000**
- group_fund = **30,000**; distributable = **270,000**
- surplus shares: A 270,000×240/900 = **72,000**; B **99,000**; C **99,000**
- totals: A **312,000**, B **429,000**, C **429,000**; total_payable = **1,170,000**
- reconciliation: 1,170,000 + 30,000 + 1,290,000 = 2,490,000 ✓

### Golden payout example B (rework + rounding)
Same as A but B's stitching accepted = **140** (10 units still in rework, unpaid).
- B piece = 252,000 + 60,000 = 312,000 → total_piece = 882,000
- surplus = 2,490,000 − 1,290,000 − 882,000 = **318,000**; fund = **31,800**; distributable = **286,200**
- exact shares: A 77,877.55 · B 101,240.82 · C 107,081.63
- floors sum to 286,198 → 2 leftover paise go to largest fractions (B, C)
- final surplus shares: A **77,877**, B **101,241**, C **107,082** (sum 286,200 ✓)

---

## 5. Default fairness config (`fairness_rules.config`)
```json
{
  "allocation": { "w_skill": 0.4, "w_capacity": 0.3, "w_fair": 0.3, "utilisation": 0.8, "max_share_per_member_pct": 40, "min_batch": 10 },
  "payout": {
    "surplus_mode": "proportional",
    "equal_pct_in_hybrid": 30,
    "group_fund_pct": 10,
    "rework_policy": "unpaid_until_accepted",
    "leader_allowance": { "type": "none", "value": 0 }
  },
  "approval": { "quorum_rule": "leader_plus_majority_present" }
}
```
Rules must be shown to members in plain language and approved by the group before use (status `approved`).

## 6. Approval quorum
- `leader_plus_majority_present`: leader approves **and** more than half of members marked present approve.
- Any `request_correction` blocks locking until a new run supersedes it.
- Locking requires quorum; locked runs are immutable.

## 7. Invariants to assert in code and tests
1. Σ stage assignment quantities = stage total (or shortfall flagged).
2. No member's allocated minutes exceed her capacity.
3. Σ payout lines = total_payable; total_payable + fund + direct costs = revenue (when surplus ≥ 0).
4. Rework/rejected units earn zero.
5. Same input → same output (no randomness, no clock reads inside engines; pass `today` in).
6. All money values are integers.
