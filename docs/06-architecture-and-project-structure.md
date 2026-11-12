# 06 — Architecture and Project Structure

## 1. High-level architecture
```
 Member phone / Leader laptop (React PWA, mobile-first)
          │  HTTPS
          ▼
 Supabase Auth ─── Postgres (RLS) ─── Storage (private photos)
          ▲                 ▲
          │                 │ service role (server only)
 Edge Functions / API  ─────┘
   ├─ domain engines (pure TS): pricing, allocation, monitor, payout
   ├─ agents: planner, pricing-explainer, voice
   ├─ LlmClient  → Claude API
   └─ VoiceClient → STT/TTS provider (+ browser fallback)
```

**Boundary rule:** `src/domain/*` is pure TypeScript with no I/O, no LLM, no React. It is shared by server, tests and (for previews) the frontend. Agents call the domain engines; engines never call agents.

## 2. Repository layout
```
/
├─ CLAUDE.md                     # copy of docs/12-ai-tool-rules.md
├─ docs/                         # this pack
├─ supabase/
│  ├─ migrations/                # 0001_init.sql ... (doc 07)
│  ├─ seed.sql
│  └─ functions/                 # edge functions (api routes)
├─ src/
│  ├─ domain/                    # PURE logic
│  │  ├─ money.ts
│  │  ├─ pricing.ts
│  │  ├─ allocation.ts
│  │  ├─ monitor.ts
│  │  ├─ payout.ts
│  │  ├─ voiceParser.ts          # number words → integers
│  │  └─ types.ts
│  ├─ server/
│  │  ├─ config.ts
│  │  ├─ llm/                    # LlmClient + provider impl
│  │  ├─ voice/                  # VoiceClient + provider impls
│  │  ├─ agents/
│  │  │  ├─ planner.ts
│  │  │  ├─ pricingExplainer.ts
│  │  │  ├─ voiceAgent.ts
│  │  │  └─ prompts/             # planner.v1.md, pricing.v1.md ...
│  │  ├─ orchestrator.ts
│  │  └─ audit.ts                # write audit rows
│  ├─ web/
│  │  ├─ routes/                 # leader/, member/, approver/
│  │  ├─ components/             # BigButton, SpeakButton, MoneyText, ProgressBar ...
│  │  ├─ features/               # orders, plan, allocation, progress, quality, payout, audit
│  │  ├─ i18n/                   # en.json, hi.json, kn.json ...
│  │  └─ lib/                    # supabase client, query hooks
│  └─ shared/                    # zod schemas shared by web and server
├─ tests/
│  ├─ unit/                      # domain engine tests, golden numbers
│  ├─ rls/                       # role access tests
│  └─ e2e/                       # one Playwright happy path
├─ scripts/
│  ├─ seed-demo.ts
│  └─ reset-demo.ts
└─ .env.example
```

## 3. State machines

**Order:** `draft → priced → planned → allocated → in_progress → quality_review → payout_proposed → payout_approved → closed`

**Plan:** `draft → published → superseded`

**Assignment:** `proposed → accepted | flagged → confirmed → in_progress → submitted → (accepted | rework | rejected by quality)`

**Replan proposal:** `proposed → approved | rejected`

**Payout run:** `proposed → correction_requested → (new run) | approved → locked`

Enforce transitions in one function per machine (`assertTransition(from, to)`), tested.

## 4. Data flow for the three demo moments
1. **Pricing:** form → `computePricing()` → numbers → pricing explainer (LLM text only) → warning card.
2. **Replanning:** progress event → `detectRisks()` → `proposeResplit()` → proposal stored → leader approves → assignments updated → audit row.
3. **Spoken earnings:** payout line → script template → `VoiceClient.speak()` → audio + on-screen text.

## 5. Security model
- RLS on every table; client uses the anon key + user JWT.
- Service role key only in server functions.
- Member endpoints filter by `auth.uid()`-linked member id.
- Audit rows written server-side in the same transaction as the change.
- Order text fields are data, never instructions (prompt-injection guard).

## 6. Error and fallback strategy
| Failure | Behaviour |
|---------|-----------|
| LLM timeout/invalid | Template plan; badge "Template plan" |
| STT fails | Show tap buttons; keep manual entry |
| TTS fails | Show on-screen text; browser speechSynthesis fallback |
| Offline | Queue entry locally, sync later (P1) |
| Quorum not reached | Payout stays `approved_pending`; cannot lock |
