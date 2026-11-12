# 07 — Database Schema (Postgres / Supabase)

Money = integer **paise**. IDs = `uuid` default `gen_random_uuid()`. All tables have `created_at timestamptz default now()`.

## 1. Tables (DDL sketch — implement as ordered migrations)

```sql
-- ===== Identity & group =====
create table shgs (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  village text, district text,
  default_language text default 'hi-IN',
  created_at timestamptz default now()
);

create table members (
  id uuid primary key default gen_random_uuid(),
  shg_id uuid not null references shgs(id),
  auth_user_id uuid unique,                 -- null for assisted-mode members
  name text not null,
  phone text,                               -- synthetic only
  role text not null default 'member' check (role in ('leader','approver','member')),
  language text default 'hi-IN',
  hours_per_day numeric(4,1) default 4,
  available_days smallint[] default '{1,2,3,4,5,6}',   -- 1=Mon..7=Sun
  assisted_mode boolean default false,
  active boolean default true,
  created_at timestamptz default now()
);

create table member_skills (
  member_id uuid references members(id) on delete cascade,
  skill text not null,                      -- 'cut','stitch','finish','pack','cook','weave'...
  level smallint not null check (level between 1 and 3),
  primary key (member_id, skill)
);

-- ===== Templates =====
create table templates (
  id uuid primary key default gen_random_uuid(),
  domain text not null check (domain in ('tailoring','food','crafts')),
  name text not null,
  version int not null default 1,
  definition jsonb not null                 -- stages, units, default rates, checklists
);

-- ===== Orders & costing =====
create table orders (
  id uuid primary key default gen_random_uuid(),
  shg_id uuid not null references shgs(id),
  domain text not null,
  product text not null,
  specification text,
  quantity int not null check (quantity > 0),
  buyer text,
  deadline date not null,
  offered_price_paise bigint not null,       -- per unit
  agreed_price_paise bigint,                 -- per unit after negotiation
  status text not null default 'draft',
  accepted_despite_loss_reason text,
  created_by uuid references members(id),
  created_at timestamptz default now()
);

create table cost_lines (
  id uuid primary key default gen_random_uuid(),
  order_id uuid not null references orders(id),
  kind text not null check (kind in ('estimated','actual')),
  category text not null check (category in ('material','labour','overhead','other')),
  label text,
  per_unit_paise bigint,                     -- for estimated
  total_paise bigint,                        -- for actual
  created_at timestamptz default now()
);

-- ===== Plan =====
create table plans (
  id uuid primary key default gen_random_uuid(),
  order_id uuid not null references orders(id),
  status text not null default 'draft' check (status in ('draft','published','superseded')),
  source text not null default 'llm' check (source in ('llm','template_fallback','manual')),
  assumptions jsonb, risks jsonb,
  published_at timestamptz, published_by uuid references members(id)
);

create table stages (
  id uuid primary key default gen_random_uuid(),
  plan_id uuid not null references plans(id) on delete cascade,
  key text not null, name text not null,
  sequence int not null,
  unit text not null,
  total_quantity int not null,
  depends_on text[] default '{}',
  required_skill text not null,
  est_minutes_per_unit numeric(6,2) not null,
  rate_paise_per_unit bigint not null,       -- piece-rate for payout
  start_date date, end_date date,
  quality_checkpoints jsonb default '[]'
);

-- ===== Allocation =====
create table assignments (
  id uuid primary key default gen_random_uuid(),
  stage_id uuid not null references stages(id),
  member_id uuid not null references members(id),
  assigned_qty int not null check (assigned_qty >= 0),
  status text not null default 'proposed'
    check (status in ('proposed','accepted','flagged','confirmed','in_progress','submitted','closed')),
  reason text,
  member_note text,
  superseded_by uuid references assignments(id),
  created_at timestamptz default now()
);

create table progress_entries (
  id uuid primary key default gen_random_uuid(),
  assignment_id uuid not null references assignments(id),
  quantity int not null check (quantity > 0),
  source text not null check (source in ('tap','voice','assisted')),
  entered_by uuid not null references members(id),
  note text,
  created_at timestamptz default now()
);

create table blockers (
  id uuid primary key default gen_random_uuid(),
  assignment_id uuid not null references assignments(id),
  raised_by uuid not null references members(id),
  reason text not null,
  resolved boolean default false,
  created_at timestamptz default now()
);

-- ===== Quality =====
create table quality_checks (
  id uuid primary key default gen_random_uuid(),
  assignment_id uuid not null references assignments(id),
  stage_id uuid not null references stages(id),
  accepted_qty int not null default 0,
  rework_qty int not null default 0,
  rejected_qty int not null default 0,
  notes text, photo_path text,
  checked_by uuid not null references members(id),
  created_at timestamptz default now()
);

-- ===== Monitoring =====
create table replan_proposals (
  id uuid primary key default gen_random_uuid(),
  order_id uuid not null references orders(id),
  status text not null default 'proposed' check (status in ('proposed','approved','rejected')),
  diff jsonb not null,                       -- [{from_member,to_member,stage_id,quantity,reason}]
  projected_end_date date,
  explanation text,
  decided_by uuid references members(id),
  decided_at timestamptz
);

-- ===== Rules & payout =====
create table fairness_rules (
  id uuid primary key default gen_random_uuid(),
  shg_id uuid not null references shgs(id),
  config jsonb not null,                     -- see doc 08 defaults
  status text not null default 'proposed' check (status in ('proposed','approved')),
  approved_at timestamptz
);

create table payout_runs (
  id uuid primary key default gen_random_uuid(),
  order_id uuid not null references orders(id),
  status text not null default 'proposed'
    check (status in ('proposed','correction_requested','approved','locked')),
  rules_snapshot jsonb not null,
  totals jsonb not null,                     -- revenue, costs, surplus, fund, payable
  supersedes_id uuid references payout_runs(id),
  created_at timestamptz default now()
);

create table payout_lines (
  id uuid primary key default gen_random_uuid(),
  payout_run_id uuid not null references payout_runs(id) on delete cascade,
  member_id uuid not null references members(id),
  assigned_qty int, submitted_qty int, accepted_qty int, rework_qty int, rejected_qty int,
  piece_earnings_paise bigint not null,
  surplus_share_paise bigint not null,
  total_paise bigint not null,
  explanation jsonb not null                 -- step-by-step breakdown
);

create table payout_approvals (
  id uuid primary key default gen_random_uuid(),
  payout_run_id uuid not null references payout_runs(id),
  approver_id uuid not null references members(id),
  decision text not null check (decision in ('approve','request_correction')),
  note text,
  created_at timestamptz default now(),
  unique (payout_run_id, approver_id)
);

-- ===== Audit & agents =====
create table audit_log (
  id bigint generated always as identity primary key,
  shg_id uuid not null,
  actor_id uuid,
  entity text not null, entity_id uuid,
  action text not null,
  before jsonb, after jsonb,
  reason text,
  created_at timestamptz default now()
);

create table agent_runs (
  id uuid primary key default gen_random_uuid(),
  order_id uuid, agent text not null,
  input_hash text, output jsonb,
  status text, model text, prompt_version text, latency_ms int,
  created_at timestamptz default now()
);
```

## 2. Derived views (create as SQL views or compute in code)
- `v_assignment_totals`: per assignment → submitted (sum of progress), accepted/rework/rejected (sum of quality checks).
- `v_order_progress`: per stage and per order progress percentages.

## 3. Row-level security (policies to implement)
Helper: `current_member()` returns the member row for `auth.uid()`; `is_leader_or_approver()` checks role.

| Table | Member | Leader / Approver |
|-------|--------|-------------------|
| shgs | select own SHG | select |
| members | select **own row** | select/insert/update all in SHG |
| member_skills | select own | all in SHG |
| orders, plans, stages | select only stages/orders they are assigned to (read-only) | all in SHG |
| assignments | select/update (accept/flag note only) **own** | all in SHG |
| progress_entries | insert/select own (or on behalf if leader for assisted members) | all |
| blockers | insert/select own | all |
| quality_checks | select own | insert/select all |
| payout_runs | no direct access (use own-line view) | select all |
| payout_lines | select **own line only**, only when run is `proposed`+ | select all |
| payout_approvals | none | insert (approvers), select |
| audit_log | none | select |
| agent_runs | none | select |

Rules:
- `audit_log` and approved/locked payout tables: **no update/delete policy for anyone**; use triggers to block UPDATE/DELETE.
- Block UPDATE on `payout_runs`/`payout_lines` once `status in ('approved','locked')` via trigger.
- Test every policy in `tests/rls/` with real member and leader JWTs.

## 4. Seed notes
See doc 10. Seed templates first, then SHG, members, skills, rules, then the demo order in `draft` state.

## 5. Template `definition` JSON example (tailoring)
```json
{
  "unit": "pieces",
  "stages": [
    { "key": "cutting",   "name": "Cutting",   "required_skill": "cut",    "est_minutes_per_unit": 3,  "rate_paise_per_unit": 600,  "depends_on": [],            "quality_checkpoints": ["Size matches pattern", "Edges straight"] },
    { "key": "stitching", "name": "Stitching", "required_skill": "stitch", "est_minutes_per_unit": 9,  "rate_paise_per_unit": 1800, "depends_on": ["cutting"],   "quality_checkpoints": ["Seams even", "Handle firmly attached"] },
    { "key": "finishing", "name": "Finishing & packing", "required_skill": "finish", "est_minutes_per_unit": 3, "rate_paise_per_unit": 600, "depends_on": ["stitching"], "quality_checkpoints": ["Threads trimmed", "Logo clear", "Packed as specified"] }
  ],
  "default_cost_per_unit_paise": { "material": 3800, "overhead": 400 }
}
```
Create analogous templates for `food` (procure, prepare, cook/process, package, label; units kg/packs) and `crafts` (prepare material, make, finish, check, pack).
