# 02 — Requirements

## 1. Problem recap
SHGs taking bulk orders (garments, food, crafts) lose orders and trust because dividing work, holding quality, consolidating output and sharing pay is chaotic. Members underprice by 30–40% because material, labour and margin are never calculated. Many members have low literacy. Allocation and payout must be fair; quality is subjective; every member must be able to see and hear her own record; the group approves every payout.

## 2. Users and roles
| Role | Description | Key needs |
|------|-------------|-----------|
| Leader | Coordinates 10–20 members, receives orders | Fast order entry, price check, plan, allocation, monitoring, payout sheet |
| Member | Works from home, may be low-literacy | Hear/see own task, report quantity by tap/voice, hear own earnings |
| Approver | Leader or elected members | Review payout basis, approve or request correction |
| Judge/Demo viewer | Hackathon audience | Clear visible agent behaviour |

## 3. Functional requirements
IDs map to features in doc 01.

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-01 | Create/edit SHG and member profiles with multi-skill tags and availability | P0 |
| FR-02 | Provide tailoring, food, crafts templates; editable per order | P0 |
| FR-03 | Create an order with product, quantity, spec, deadline, buyer, price | P0 |
| FR-04 | Compute cost per unit, break-even, margin, verdict and warn on loss | P0 |
| FR-05 | Generate an AI draft plan (stages, tasks, dependencies, timeline) as structured JSON, validated, saved as draft | P0 |
| FR-06 | Leader can edit tasks and publish the plan | P0 |
| FR-07 | Recommend allocation using skill, availability, workload and fairness with a reason per assignment | P0 |
| FR-08 | Members accept or flag an assignment; leader confirms; changes logged | P0 |
| FR-09 | Record progress quantities and blockers | P0 |
| FR-10 | Store estimated and actual costs separately | P0 |
| FR-11 | Compute payout deterministically from accepted work; explain each line | P0 |
| FR-12 | Group approval workflow with quorum; lock approved payout | P0 |
| FR-13 | Append-only audit log of all critical changes | P0 |
| FR-14 | Row-level security: members see only own record | P0 |
| FR-15 | Stage-wise quality check: accepted/rework/rejected + notes/photo | P1 |
| FR-16 | Monitor at-risk tasks; propose re-split; leader approves | P1 |
| FR-17 | Voice progress capture with read-back confirmation | P1 |
| FR-18 | Spoken task and earnings in member's language | P1 |
| FR-19 | Downloadable report | P1 |
| FR-20 | Group-configurable fairness rules, visible to members | P1 |
| FR-21 | Local save and later sync for poor connectivity | P1 |

## 4. Non-functional requirements
| ID | Requirement |
|----|-------------|
| NFR-01 | Mobile-first, works on low-end Android browsers; responsive up to desktop for the leader dashboard |
| NFR-02 | Large tap targets (min 48px), icon + label buttons, high contrast |
| NFR-03 | UI strings externalised (i18n); English plus at least one Indian language for the pilot |
| NFR-04 | Money stored as integer paise; no floating point in money code |
| NFR-05 | Payout engine reproducible: same inputs → same outputs; covered by unit tests |
| NFR-06 | LLM calls have timeout (≤20s), retry once, schema validation, and a template fallback if they fail |
| NFR-07 | No secrets in the frontend bundle; LLM and voice keys only on the server |
| NFR-08 | Demo must survive network failure: seeded data and a recorded backup demo |
| NFR-09 | Accessibility: every key number has a spoken equivalent (TTS) |
| NFR-10 | Privacy: synthetic data only; no real phone numbers or Aadhaar-like identifiers |

## 5. Constraints and design rules
1. **AI proposes, rules calculate, humans approve.** LLM never computes money, allocation quantities or payout.
2. Plans, replans and allocations are drafts until a human confirms.
3. Fairness is configurable and visible; the group agrees the rules.
4. Quality is subjective: use checklists, human decision, AI flags only.
5. Keep it simple: no vector DB, no blockchain, no multi-agent framework.
6. Payments are simulated and labelled.

## 6. Tech stack (default)
| Layer | Choice |
|-------|--------|
| Frontend | React + TypeScript + Vite, Tailwind CSS, React Router, TanStack Query, react-i18next |
| Backend/data | Supabase (Postgres, Auth, RLS, Storage) |
| Server logic | Supabase Edge Functions (Deno/TypeScript) **or** a small FastAPI service; pick one in Task 0 and stay with it |
| LLM | Anthropic Claude API (structured output via tool use / JSON schema); provider behind an interface so it can be swapped |
| Allocation | Pure TypeScript rules; OR-Tools optional, only if time permits |
| Voice | Browser Web Speech API as fallback; Indian-language STT/TTS provider (e.g. Sarvam AI or Bhashini) for the pilot language — verify language support first |
| Testing | Vitest (unit), Playwright (one e2e happy path) |
| Hosting | Vercel/Netlify (frontend), Supabase (backend) |

## 7. Assumptions
- Pilot language chosen by the team (default Hindi; Kannada as second). Confirm STT/TTS coverage.
- One SHG, 10 members, one order for the demo.
- Surplus-sharing default is proportional to accepted work value; group can change it.

## 8. Definition of done (MVP)
See doc 11. In short: the 3-minute demo runs end to end on seeded data, all golden-number tests pass, RLS proven with a member login, and a backup recording exists.
