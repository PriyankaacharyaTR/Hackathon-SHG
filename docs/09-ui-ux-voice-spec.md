# 09 — UI / UX and Voice Spec

Design for SHG members with **low literacy**, on **low-end Android phones**, with the leader on phone or laptop. The product must not look like a generic task tracker: make agent moments visible.

## 1. Design principles
1. **Icon + number + voice first.** Text is secondary.
2. One primary action per screen, large (≥ 56 px height, ≥ 48 px touch target).
3. Every important number has a 🔊 **Speak** button.
4. Colour is never the only signal (use icons and words too): green ✓ accepted, amber ↻ rework, red ✕ rejected.
5. Show the *basis* of every amount ("150 × ₹18 = ₹2,700").
6. Confirm before saving anything entered by voice.
7. Assisted mode: always show "Entered by [name] for [member]".
8. Mark all payments **SIMULATED**.

## 2. Roles and navigation

### Member workspace (mobile)
Bottom bar with 3 big icons: **My work**, **Report**, **My pay**.
- **My work:** card per task — stage icon, quantity, due day, 🔊. Buttons: ✓ Accept / ⚑ Problem.
- **Report:** big number pad or − / + stepper, 🎤 mic, **Save** (confirm dialog: "20 bags — correct?").
- **My pay:** assigned / submitted / accepted / rework quantities, rate, total; 🔊 "Hear my earnings"; status chip "Waiting for group approval" / "Approved (simulated)".

### Leader dashboard (mobile + desktop)
Tabs: **Order**, **Plan**, **People**, **Progress**, **Quality**, **Pay**, **Rules**, **History**.
- Home shows: order progress ring, days left, risk badges, pending actions (flags, proposals, approvals).

### Approver view
Payout sheet, rules, approve / request correction.

## 3. Key screens

| Screen | Contents | Agent moment |
|--------|----------|--------------|
| Create order | Form; 🎤 speak-order (P1) | — |
| Pricing check | Cost breakdown bars; verdict banner; recommended price; "Negotiate" / "Accept anyway (reason)" | **Pricing agent** — red loss warning |
| Draft plan | Stage timeline (Gantt-like), tasks, assumptions, risks; badge "AI draft" or "Template plan"; Edit; **Publish** | Planner |
| Allocation | Table: member → task → qty → reason; capacity bars; Accept/Flag status chips; **Confirm** | Allocation |
| Progress | Per-task bars; member rows; blockers | — |
| Replan proposal | Side-by-side "before → after" diff; projected finish; **Approve / Reject** | **Monitor agent** |
| Quality | Checklist per stage; counts for accepted/rework/rejected; notes; photo | — |
| Payout sheet | Table of members with each column, expandable "how calculated"; totals reconcile line; **SIMULATED** banner | Payout |
| Spoken earnings | Big total, quantities, 🔊 auto-play option | **Voice agent** |
| Audit | Filterable timeline | — |

## 4. Voice interaction spec

### 4.1 Progress by voice
1. Tap 🎤 (hold or toggle). Show animated waveform and "Listening…".
2. Show transcript text under the mic.
3. Parse quantity; show it large: **20**, with unit icon.
4. Ask aloud and on screen: "20 bags — correct?" → big **✓ Yes** / **✕ No**.
5. Yes → save, success sound + tick. No → "Say again" or use keypad.
6. If no number is recognised after 2 tries → open keypad automatically.

### 4.2 Spoken earnings script template (English, translate per language)
```
"{name}, for {order_product}: you were given {assigned}. You submitted {submitted}.
{accepted} were accepted. {rework} need rework. 
Your pay for work is {piece_total}. Your share of surplus is {surplus_share}. 
Total {total}. This is a simulated payment and is waiting for group approval."
```
Build the script from the payout line's `explanation` object (doc 08 §4). Same text appears on screen.

### 4.3 Language handling
- Member language from profile; leader can switch.
- All UI strings in `i18n/*.json`; no hard-coded text in components.
- Number formatting: Indian grouping (₹1,23,456) with `Intl.NumberFormat('en-IN')`; speak amounts as words via the TTS provider.
- **Verify** STT/TTS support for each pilot language before promising it; fall back to the browser engine or on-screen text.

## 5. Components to build
`BigButton`, `IconLabel`, `SpeakButton`, `MicButton`, `QuantityStepper`, `MoneyText` (paise → ₹, with 🔊), `ProgressBar`, `RiskBadge`, `ReasonChip`, `DiffTable`, `ExplainAmount` (expandable breakdown), `SimulatedBanner`, `RoleGuard`.

## 6. Visual style
- High-contrast, warm palette; minimum 18 px body text on mobile, 24 px for key numbers.
- Rounded cards, generous spacing, no dense tables on member screens.
- Leader desktop views can use tables, but collapse into cards on mobile.
- Loading and error states are friendly and spoken where possible ("No internet. Your entry is saved.").

## 7. Accessibility checklist
- [ ] Every action reachable by tap with large targets.
- [ ] Screen text scales with system font size.
- [ ] Icons have `aria-label`s in the active language.
- [ ] Audio never auto-plays without a visible stop button.
- [ ] Works at 360 px width.
- [ ] No information conveyed by colour alone.
