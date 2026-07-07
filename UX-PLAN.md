# Dunk Protocol — UX/UI Overhaul Plan

**Audience for this doc:** an AI coding agent (Opus/Sonnet-class) executing improvements to this app.
**App:** single-file PWA (`index.html`, ~1600 lines) + `sw.js` + vendored libs. No build step, no framework — keep it that way.
**Live:** https://wallacereeseiv-commits.github.io/dunk-protocol/ (GitHub Pages, repo `wallacereeseiv-commits/dunk-protocol`, push to `main` auto-deploys).

---

## 0. Context: who this is for and what "better" means

The user is one person mid-journey: **building general health and athleticism over 12 weeks with the goal of dunking**. Two usage contexts:

1. **In the gym, phone in hand, between sets.** Sweaty, arm's length, 10-second glances. Needs: what's next, how heavy, check it off, rest timer. Big targets, big type, zero hunting.
2. **At home, planning or reviewing.** Needs: am I on track, is my vert improving, did I eat enough protein this week.

The current app is a **well-organized reference document**. It is not yet a **training companion**. Every change below serves one test: *does this help a beginner walk into the gym, execute today's session, and see proof they're getting closer to dunking?*

### Hard constraints — do not break these

- **Single-file architecture.** All CSS/JS inline in `index.html`. Vendored `chart.umd.js` and `firebase-*-compat.js` stay local (offline requirement). No CDNs, no npm, no build step.
- **Offline-first.** Everything must work with no network (except sync + Google sign-in). After any asset change, bump the cache name in `sw.js` (`dunk-protocol-v2` → `-v3`, etc.) or phones keep stale copies.
- **Sync schema is shared state.** Cloud sync (Firestore, `attachSyncListener` line ~568) does whole-blob last-write-wins on `updatedAt`. New persisted fields must be **additive with defaults** so old blobs load cleanly (see `loadState` ~439 — every field is guarded). Never rename existing fields (`COMPLETION`, `ALT_LOG`, `weighIns`, `foodLog`, `currentWeek`, `logId`). Add a `schemaVersion` field when you first touch the schema.
- **No personal info in the repo/site.** Height/weight/name were deliberately stripped. Don't reintroduce them in code, copy, or commits.
- **Keep the Cowork-only AI food scan degrade path** (`analyzeFood` guards on `window.cowork`).
- **Verify at 375×812** with the preview tools before pushing. Test: fresh localStorage, populated state, and signed-in sync if feasible.
- Commit and push per phase (not per file) with clear messages.

---

## 1. Findings (deep audit, mobile 375px)

### F1. The app has no concept of "today" in the program
- `currentWeek` is whatever was last tapped; the weekday defaults to today but week does not advance. A 12-week program where the user must remember which week they're in will drift by week 3.
- Day tabs show a red outline for "today" but **no completion state** — you can't glance and see "Mon ✓ Tue ✓ Wed ·".
- Opening screen spends ~280px on chrome (header + sync bar + nav + week chips + day chips) before any training content.

### F2. Exercise cards are read-only prescriptions
- The only interactions are "🔄 Alternatives" and one day-level "✅ Mark Complete" at the *bottom of the page* (`renderCompleteSection` ~1171).
- No per-set check-off, no "weight I actually used," no reference to last week's numbers. The program's entire engine is progressive overload ("+5 lbs/week" appears in weight text), yet the app **cannot tell you what you lifted last week.** This is the single biggest gap.
- No rest timer, despite prescriptions like "Rest 2 min between."
- `COMPLETION` stores only `{date, notes}` — no performance data to build any progress view on.

### F3. Nothing tracks the actual goal: dunking
- Zero vertical-jump measurement anywhere. The only trend chart is body weight. For motivation — the make-or-break factor of a 12-week solo program — the app needs a **vert/reach test protocol and chart**. This is the emotional core of the product and it's missing.

### F4. Visual hierarchy inverts importance
- The 12-step warmup list occupies a full screen before the first KEY lift; warmup, accessory, and KEY exercises all render as same-weight cards. `KEY` badge is 0.58rem.
- Body text is 0.72–0.88rem with muted grays on near-black (`#64748b` on `#111320` ≈ 3.4:1) — below WCAG AA for its size and genuinely hard to read at arm's length in a gym.
- Touch targets: week chips are ~26px tall (`.wk-btn` padding 4px 9px); Apple/Google minimum is 44px.
- Emoji as iconography (🔥📋⚖️🔄✅📷🌯) reads inconsistent; fine as accents, weak as UI signifiers.
- Nav is not sticky — switching tabs requires scrolling back to top. On phone this should be a **fixed bottom tab bar** (thumb zone), with safe-area padding (`viewport-fit=cover` is already set but no `env(safe-area-inset-*)` padding exists).

### F5. Food tab flow is backwards for a repeat-logging habit
- Order today: 4 tall macro cards (~400px) → disabled scan box → manual form → 8 restaurant menus → **Today's Log at the very bottom**. The thing you check most (what have I eaten / remaining macros) and the action you take most (log the same foods you always eat) are farthest apart.
- No "recent/frequent foods" quick-add. Logging the same breakfast daily = full manual re-entry every time.
- **Nutrition history doesn't exist:** `loadState` discards `foodLog` on a new day and `saveState` overwrites the cloud blob, so yesterday's nutrition is gone forever. No streaks, no weekly protein average, nothing to correlate with training.
- Disabled AI-scan box occupies prime space on phones where it can never work (F-grade real estate use; it's already feature-detected — collapse it).

### F6. Weigh-in and Session Log are dead-end screens
- Weigh-in asks the user to type "Week #" — the app should know the week. Empty chart renders a meaningless 0–1.0 axis. No goal/target line, no context.
- Session Log surfaces internal keys to the user ("`W1D0` · 7/7/2026"), has no grouping by week, no adherence stats (e.g., "5 of 7 sessions this week"), no streaks. This should be the "look how far you've come" screen; it's currently a debug view.

### F7. Assumes strength-training literacy the target user may not have
- "Tempo: 21X1", "8RPE", "KOT", "ATG", "triple extension" appear with no explanation anywhere. For "someone building up their health," each unexplained term is a small confidence tax. There's no first-run onboarding explaining the 3 phases or how to test progress.

### F8. Feedback & polish issues
- `alert()` used for all confirmations (lines ~1225, 1234, 1293, 1375, 1427, 1440) — jarring, blocks UI, feels 1998.
- "Date (today)" is a free-text input on session complete; should default to today with a date picker only as override.
- Sync bar prints raw Firebase error strings; empty states are bare text with no guidance; no PWA "Add to Home Screen" nudge for first-time phone visitors.
- Known sync risk (document, don't necessarily fix in a UI pass): whole-blob last-write-wins can drop one side if two devices log the same day without refreshing. Mitigation if touched: union-merge `COMPLETION`/`ALT_LOG` maps (and any new arrays) remote-vs-local before push in `attachSyncListener`/`pushStateToCloud`.

---

## 2. The plan — 5 phases, independently shippable

Execute in order; each phase is a coherent commit/push and leaves the app fully working. Sizes: P0 ≈ half day-equivalent, P1 the largest, P2–P4 moderate.

### Phase 0 — Design system & mobile shell (foundation, do first)

1. **Design tokens.** Introduce CSS custom properties at `:root` for the existing palette (bg `#0f0f13`, surface `#111320`/`#13131f`, borders, accent purple `#7b6eff`, red `#e63946`, green `#4ade80`, blue `#38bdf8`, orange `#fb923c`) and a 4-step type scale with **minimum body size 0.85rem** and minimum contrast `#94a3b8`-on-surface for secondary text (kill `#64748b` for anything that must be read mid-workout; keep it for true fine print only). Replace hardcoded values as you touch each section — full sweep not required in one pass, but all NEW code uses tokens.
2. **Bottom tab bar.** Replace the top `<nav>` (line ~243) with a fixed bottom bar: 4 tabs with icon + label (inline SVG icons, not emoji — simple line icons: dumbbell, utensils, trend-up, calendar-check). 56px + `env(safe-area-inset-bottom)` padding. Content gets matching bottom padding. Keep `showTab()` API.
3. **Compact header.** Merge sync status into the header as a small cloud icon + dot (green = synced, gray = local-only, red = error; tap opens the existing sync modal / a small status sheet). Delete the full-width sync bar (`#sync-bar`, line ~208). Header shrinks to one row.
4. **Touch targets ≥44px** for week chips, day tabs, all buttons. Week/day pickers become horizontally scrollable rows with larger chips.
5. **Toast system.** ~20-line helper: `toast(msg)` → bottom-anchored, auto-dismiss 2s, one at a time. Replace every `alert()` (6 call sites listed in F8). Keep `confirm()`-style flows inside existing modals.
6. **sw.js**: bump cache to `dunk-protocol-v3`.

**Acceptance:** no `alert(` remains; nav reachable without scrolling from anywhere; Lighthouse-style tap-target sanity at 375px; all four tabs still function with fresh + existing localStorage.

### Phase 1 — Make the Workout tab a training companion (the big one)

1. **Program anchor date.** New persisted field `programStart` (ISO date). First run (or first run after update): a small sheet asks "When did/does your 12-week program start?" with "Today" as the default button. Derive `currentWeek` = clamp(weeksSince(programStart)+1, 1..12) on load; manual week override still possible via the week picker (store override separately; a "back to this week" chip appears when overridden).
2. **Today-first layout.** Above the exercise list, a hero card: day name, phase, week N of 12 with a slim 12-segment progress strip, and a **"Start Session" / "Resume"** state. Week picker collapses behind the progress strip (tap to expand). Day tabs get completion dots (✓ = `COMPLETION[wKey(day)]` non-empty).
3. **Collapse the warmup** into an accordion (collapsed by default, "Warmup · 12 steps · ~8 min"). Optional nicety: "check as you go" checkboxes, ephemeral (not persisted).
4. **Per-exercise set logging.** This is the core feature. New persisted field `setLogs` = `{ [dayKey]: { [secIdx_exIdx]: [{w, r, done}] } }` (weight number or null for bodyweight, reps, done bool).
   - Each exercise card renders its prescribed sets as tappable set rows: `Set 1 — [weight][reps] ✓`. Tap ✓ marks done (pre-filling weight/reps from prescription or last entry); long-press or an edit icon adjusts numbers (numeric inputs, `inputmode="decimal"`).
   - **Last-week reference:** when rendering week N, look up `setLogs` for the same dayKey at week N−1 and show "Last week: 3×10 @ 35 lb" under the weight box. This single line is what makes progressive overload real.
   - Checking the final set of the final exercise prompts the existing day-complete modal (notes prefilled with auto-summary, e.g., "All sets done. Top set: 45 lb.").
   - KEY exercises: stronger card treatment (accent left border, slightly larger name) so hierarchy matches importance.
5. **Rest timer.** After checking a set, show an inline countdown chip on that card (default 120s power/strength sections, 90s accessories; tap to +30s/skip). Vibrate via `navigator.vibrate?.(200)` on finish (fails silently on iOS — fine). No audio (gym headphones vary); visual + haptic only. One active timer at a time, survives tab switches within the session (in-memory only).
6. **Mobility/yoga days** (`isMobility`, lines ~741/865): render routine items as check-as-you-go rows feeding the same day-complete flow (ephemeral checks, persisted completion).
7. Wire `setLogs` into `saveState`/`loadState`/`currentSnapshot` (additive, guarded), and set `schemaVersion: 2`.

**Acceptance:** from a fresh phone: open app → see today's session → check off sets with weights → rest timer runs → completing last set triggers day-complete → Session tab shows it; week N+1 of the same day shows "last week" numbers. Old saved blobs (no `setLogs`) load without errors.

### Phase 2 — Progress hub: make the dunk visible (highest motivational ROI)

1. **Vert test protocol.** New persisted array `vertTests: [{date, type, value, notes}]` where type ∈ `touch` (highest point touched, inches or "rim −X in") | `jump` (measured vertical in inches). Add a "Test Day" card that appears on Jump Day (day index 5) every 4th week (weeks 1, 4, 8, 12) prescribing the simple protocol: warm up fully → 3 max-effort jumps at a wall/backboard → record best touch height. Entry UI: one number + unit + optional note.
2. **Merge Weigh-In tab into a "Progress" tab** (rename nav): vert chart first (the goal), weight chart second, session adherence third. Remove the "Week #" input — prefill from derived current week (editable). Fix empty states: friendly copy + a ghosted example chart or a "Log your first test" CTA instead of a 0–1.0 axis.
3. **Adherence view** (replaces raw Session Log list): group by week — "Week 3: 5/7 sessions ✓✓✓··✓·" with per-day dots colored by workout tag; a current-streak and best-streak stat; the existing notes/alternatives lists below, translated to human labels ("Week 1 · Monday — Lower Athletic Power", never `W1D0`).
4. Charts: reuse vendored Chart.js; consistent dark styling; weight chart gains an optional dashed goal line if the user sets a target (new optional field `weightGoal`).

**Acceptance:** a user who tests in week 1 and week 4 sees two points on a vert chart and a delta ("+1.5 in"); Session Log no longer displays internal keys; weigh-in requires only typing a number.

### Phase 3 — Food tab: optimize for the daily 30-second log

1. **Reorder:** compact macro summary (one row of 4 mini-stats with progress bars, ~90px total) → **Today's Log** → **Quick Add** (recents/frequents) → manual form → restaurants (behind their tabs) → scan box **last, and only when `window.cowork` exists** (hide entirely otherwise — the current hint text moves to a one-time toast).
2. **Recents/frequents.** New persisted `recentFoods` (last ~30 unique by name, with macros, ranked by frequency then recency). One tap re-logs. This kills 80% of manual entry.
3. **Nutrition history.** New persisted `foodHistory`: on day rollover in `loadState`, before discarding, push `{date, totals:{cal,pro,carb,fat}, count}` (totals only — keep the blob small; cap 120 entries FIFO). Progress tab gains a small 7-day protein/calorie adherence strip fed by this.
4. Numeric inputs get `inputmode="decimal"`; Add Food button stays visible (sticky within section) while the keyboard is open if feasible.

**Acceptance:** logging a repeat food = 2 taps from app open (Food tab → tap recent). Yesterday's totals visible in Progress after a day rollover. Old blobs without `foodHistory`/`recentFoods` load cleanly.

### Phase 4 — Onboarding, education, and polish

1. **First-run intro** (3 short cards, swipe/skip, persisted `seenIntro`): ① the 12-week arc (Foundation → Development → Peak) and what each phase does; ② how to read a prescription (annotated example explaining sets×reps, **tempo `21X1` digit by digit**, RPE in one sentence); ③ test days + install-to-home-screen tip (with iOS Share→Add to Home Screen hint when `!navigator.standalone` on iOS).
2. **Inline glossary.** Tempo and RPE values become tappable → small bottom sheet with the explanation. Terms like KOT/ATG in exercise notes get a one-line parenthetical on first occurrence per card ("knees over toes").
3. **Copy pass.** Empty states, sync errors (map Firebase codes to plain English: "Couldn't reach sync — changes saved on this device and will sync later"), day-complete confirmation ("Week 3 Monday logged — 9 sessions total").
4. **Consistency sweep:** replace remaining emoji-as-UI with the Phase 0 SVG icon set (keep celebratory emoji in toasts/copy where it's tone, not UI); unify border-radius/spacing to tokens; ensure focus states exist for keyboard/switch access.

**Acceptance:** a brand-new user can explain what "3×5 @ 21X1, RPE 8" means after tapping it; no raw Firebase error strings reachable; icons consistent across tabs.

---

## 3. Execution notes for the agent

- **Read before writing:** `renderWorkout` (~1086), `renderExercise` (~1137), `renderCompleteSection` (~1171), `renderFoodLog` (~1314), `renderWeighChart` (~1447), `refreshLogTab` (~1261), `showTab` (~1469), persistence block (~420–470), sync block (~448–612), data: `WARMUPS` (~614), `WORKOUTS` (~661), `RESTAURANTS` (~882), `TARGETS` (~415). Line numbers drift — grep, don't trust them blindly.
- The whole UI is string-concatenation rendering with inline `onclick`. **Match that idiom**; do not introduce a framework or template system. Escape user-entered strings (notes, food names) when interpolating — there are existing quote-escaping patterns near `renderExercise`/`altDataStr`.
- State writes must go through `saveState()` (it also queues the cloud push). Never write localStorage directly elsewhere.
- After each phase: bump `sw.js` cache version, verify at 375×812 (fresh + existing state), then commit and push to `main`. Confirm the live site updated (`curl` + grep for a marker string) — the local preview sandbox cannot reach the live URL.
- If a change would break old synced blobs, stop and add a migration in `loadState` instead.
