# 03 — API Configurations

Everything the AI tool needs to wire up external services and define the internal API. Provider endpoints and model names change; **verify against current provider docs** before coding.

## 1. Environment variables

### `.env.example` (commit this; never commit `.env`)
```bash
# ---------- Supabase ----------
VITE_SUPABASE_URL=https://YOUR-PROJECT.supabase.co
VITE_SUPABASE_ANON_KEY=YOUR-ANON-KEY          # safe for browser (RLS protects data)
SUPABASE_SERVICE_ROLE_KEY=YOUR-SERVICE-KEY    # SERVER ONLY. Never expose to the frontend.

# ---------- LLM (server only) ----------
LLM_PROVIDER=anthropic
ANTHROPIC_API_KEY=YOUR-KEY
LLM_MODEL=claude-sonnet-5-5                   # configurable; keep in env, not in code
LLM_TIMEOUT_MS=20000
LLM_MAX_RETRIES=1

# ---------- Voice (server only) ----------
VOICE_PROVIDER=sarvam                         # sarvam | bhashini | browser
SARVAM_API_KEY=YOUR-KEY
STT_LANGUAGE_DEFAULT=hi-IN
TTS_LANGUAGE_DEFAULT=hi-IN
SUPPORTED_LANGUAGES=en-IN,hi-IN,kn-IN         # trim to what is verified

# ---------- App ----------
APP_ENV=development
PAYMENTS_MODE=SIMULATED                       # must stay SIMULATED
DEFAULT_CURRENCY=INR
USE_LLM_FALLBACK_TEMPLATES=true               # use template plan if LLM fails
```

Rules:
- Only `VITE_`-prefixed variables reach the browser. Keys for LLM, voice and service role stay on the server.
- Load config through one module (`src/server/config.ts`) that validates with zod and fails fast if a variable is missing.

## 2. Supabase setup
1. Create project. Enable Email/OTP or phone auth (for the demo, seeded users with magic-link or password are fine).
2. Run migrations from doc 07 in order.
3. Enable RLS on **every** table (doc 07 lists policies).
4. Storage bucket `quality-photos` (private; signed URLs).
5. Create demo users: 1 leader, 2 approvers, 10 members (synthetic).
6. Service-role key only inside Edge Functions / backend.

## 3. LLM provider interface
Define one interface so the provider can be swapped.

```ts
// src/server/llm/types.ts
export interface LlmClient {
  generateStructured<T>(args: {
    system: string;
    user: string;
    schemaName: string;
    jsonSchema: object;      // JSON Schema for the expected output
    zod: import('zod').ZodType<T>;
    timeoutMs?: number;
  }): Promise<{ ok: true; data: T; raw: unknown } | { ok: false; error: string }>;
}
```
Implementation notes:
- Use tool-use / structured output so the model must return JSON matching the schema.
- Validate with zod. If invalid, retry once with the validation error appended. If still invalid → fallback template.
- Log prompt version, model, latency and validity to `agent_runs` (doc 07). Do not log personal data beyond synthetic IDs.
- Temperature low (0–0.3) for planning.

## 4. Voice provider interface
```ts
export interface VoiceClient {
  transcribe(audio: Blob, lang: string): Promise<{ text: string; confidence?: number }>;
  speak(text: string, lang: string): Promise<{ audioUrl: string }>;
}
```
- Route through the server so keys stay hidden.
- **Browser fallback**: `SpeechRecognition` / `speechSynthesis` when the provider is unavailable.
- Always run **read-back confirmation** (doc 04, Voice agent) before saving anything.
- Number words: parse spoken quantities with a small deterministic parser (e.g. "pachaas" → 50) plus a numeric regex; do not rely on the LLM for the final number.

## 5. Internal REST API contract
Base path `/api/v1`. All endpoints require auth. JSON in/out. Money in **paise** (integers). Timestamps ISO-8601 UTC.

Standard error shape:
```json
{ "error": { "code": "VALIDATION_ERROR", "message": "Human readable", "details": {} } }
```

### SHG and members
| Method | Path | Role | Purpose |
|--------|------|------|---------|
| GET | /shg | any | Current SHG profile |
| GET | /members | leader, approver | List members with skills |
| POST | /members | leader | Add member |
| PATCH | /members/:id | leader | Update member, skills, availability |
| GET | /members/me | member | Own profile |

### Templates and orders
| Method | Path | Role | Purpose |
|--------|------|------|---------|
| GET | /templates | leader | List domain templates |
| POST | /orders | leader | Create order |
| GET | /orders/:id | leader, approver | Order detail |
| POST | /orders/:id/pricing-check | leader | Run deterministic costing + Pricing agent explanation |
| PATCH | /orders/:id/cost-inputs | leader | Save estimated cost inputs |

`POST /orders` body:
```json
{
  "product": "Cloth bag",
  "domain": "tailoring",
  "quantity": 300,
  "specification": "Cotton, 12x14 in, handle 20 in, printed logo",
  "deadline": "2026-12-24",
  "buyer": "Demo Buyer Pvt Ltd",
  "offered_price_paise": 6500
}
```

`POST /orders/:id/pricing-check` response:
```json
{
  "cost_per_unit_paise": 7200,
  "breakdown": { "material": 3800, "labour": 3000, "overhead": 400 },
  "target_margin_pct": 15,
  "break_even_price_paise": 7200,
  "recommended_price_paise": 8300,
  "offered_price_paise": 6500,
  "verdict": "LOSS",
  "total_impact_paise": -210000,
  "explanation": "AI-written plain-language summary (advisory)"
}
```

### Plan
| Method | Path | Role | Purpose |
|--------|------|------|---------|
| POST | /orders/:id/plan/draft | leader | Planner agent generates draft |
| PATCH | /plans/:id | leader | Edit draft stages/tasks |
| POST | /plans/:id/publish | leader | Publish (locks structure, creates tasks) |

### Allocation
| Method | Path | Role | Purpose |
|--------|------|------|---------|
| POST | /plans/:id/allocation/recommend | leader | Run rule engine, return proposals with reasons |
| POST | /allocations/:id/respond | member | `{ "response": "accept" \| "flag", "note": "..." }` |
| POST | /plans/:id/allocation/confirm | leader | Confirm assignments |
| POST | /assignments/:id/reassign | leader | Reassign with reason (logged) |

### Progress, monitoring, quality
| Method | Path | Role | Purpose |
|--------|------|------|---------|
| POST | /assignments/:id/progress | member, leader (assisted) | `{ "quantity": 20, "entered_by": "...", "source": "tap"\|"voice" }` |
| POST | /assignments/:id/blocker | member | Raise blocker |
| POST | /orders/:id/monitor/run | system, leader | Run monitor; returns risks and proposed re-split |
| POST | /replans/:id/approve | leader | Apply approved re-split |
| POST | /replans/:id/reject | leader | Reject proposal |
| POST | /assignments/:id/quality | leader, approver | `{ "stage_id": "...", "accepted": 18, "rework": 2, "rejected": 0, "notes": "...", "photo_path": "..." }` |

### Costs and payout
| Method | Path | Role | Purpose |
|--------|------|------|---------|
| POST | /orders/:id/actual-costs | leader | Record actual costs |
| POST | /orders/:id/payout/calculate | leader | Run deterministic engine → `payout_run` (status `proposed`) |
| GET | /payout-runs/:id | leader, approver | Sheet with explanation per line |
| GET | /payout-runs/:id/mine | member | Own line only |
| POST | /payout-runs/:id/approve | approver | `{ "decision": "approve" \| "request_correction", "note": "..." }` |
| POST | /payout-runs/:id/lock | leader | Lock after quorum reached |

### Voice and reports
| Method | Path | Role | Purpose |
|--------|------|------|---------|
| POST | /voice/transcribe | any | Audio → text (+ parsed quantity) |
| POST | /voice/speak | any | Text → audio URL |
| GET | /voice/earnings/:payoutLineId | member (own) | Returns spoken-script text and audio |
| GET | /orders/:id/report | leader | Downloadable report (PDF/HTML/MD) |
| GET | /audit | leader, approver | Filterable audit log |

## 6. Idempotency and safety
- `POST` endpoints that create financial or approval records accept an `Idempotency-Key` header.
- Approved payout runs are immutable; corrections create a new run linked via `supersedes_id`.
- Every mutating endpoint writes an `audit_log` row in the same transaction.

## 7. Rate and cost control
- Cache the Planner draft per (order, template version, prompt version).
- Limit LLM calls: pricing explanation, plan draft, monitor explanation, replan explanation. Payout explanation text is generated from templates, **not** the LLM.
