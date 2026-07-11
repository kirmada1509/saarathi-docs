# Saarathi — Implementation Plan

> Phase-wise build plan for the ~30 hours between now and **July 12, 11:45 PM IST**. Companion to `product_spec_v2.md` (what/why) and `architecture_v2.md` (exact how). Every phase ends demoable; every phase has a definition of done an agent can verify.

---

## 0. The codebase question, answered first

**Do you need a separate AI-services codebase? No — categorically.**

Your "AI layer" is two function calls: an in-process embedding similarity check (`transformers.js`, no network) and one LangChain→Groq call for prose. That's ~150 lines across `preferences.ts` and `explain.ts`. A separate AI service would mean: a third deploy, a network hop between your own code, a new failure mode between ranking and explanation, auth/env plumbing — all to wrap two function calls that judges will never see as a "service." Microservices earn their cost when teams need independent deploy cadence. You are one person with one deadline. The AI code lives in `core/`, isolated at the *module* level (which is the isolation judges can actually read), not the *network* level.

**Is `saarathi-backend` + `saarathi-frontend` enough? It's more than enough — my recommendation is one fewer.**

- **Option A (recommended): one app.** `saarathi/saarathi-app` — a single Next.js codebase where `core/` holds all business logic, `app/api/` is the backend, `app/page.tsx` + `components/` are the frontend. One `npm run dev`, one deploy, one place types live. This has been the on-record recommendation since spec §8, and the solo-participant + 30-hour facts have only strengthened it.
- **Option B (supported): the split you named.** `saarathi/saarathi-backend` (NestJS wrapping the same `core/`) + `saarathi/saarathi-frontend` (Next.js, presentation only). Choose this only if you *want* NestJS visible in the submission as a signal. If so, three rules make it survivable: (1) `core/` lives inside the backend, framework-free, exactly as specced — NestJS services are thin wrappers; (2) the frontend gets one hand-mirrored `lib/types.ts` — do **not** spend hours on a shared package/npm-workspace publishing setup; (3) backend on `:4000`, frontend proxies via Next `rewrites()` so the browser sees one origin and CORS never becomes a demo-day bug.

Everything below is written for **Option A**, with a marked fork at Phase 2 if you take Option B. The phase content, hours, and definitions of done are identical either way — only the wrapping differs.

---

## 1. Repository layout & agent context

The repo root is where your coding agent starts, so the root must *teach* the agent before it writes a line:

```
saarathi/
├── CLAUDE.md                      # agent entry point — see contents below
├── docs/
│   ├── product_spec_v2.md         # the what and why
│   ├── architecture_v2.md         # the exact how — contracts, tokens, components
│   └── implementation_plan.md     # this file — the build order
├── data/
│   ├── flights_data.csv           # full ~50k-row file from the toolkit
│   ├── user_data.csv
│   └── benchmark_prompts.json
├── saarathi-app/                  # Option A: the entire product (Next.js)
│   ├── core/                      # pure TS decision engine + tests
│   ├── app/                       # pages + api routes
│   ├── components/                # ui/ primitives + decision/ zones
│   └── lib/                       # zustand store, tanstack hooks
└── submission/                    # built during Phase 8, uploaded to OneDrive
    ├── solution_summary.md
    ├── deck/
    ├── demo_video/
    └── README.md                  # also mirrored at repo root
```

**`CLAUDE.md` contents** (write this in Phase 0 — it's the highest-leverage 20 minutes of the project, because every agent session inherits it):
- One paragraph: what Saarathi is (Decision Screen, counterfactuals as flagship) and the deadline.
- Reading order: `docs/product_spec_v2.md` §2–§7 → `docs/architecture_v2.md` §3–§6 → this plan's current phase.
- The five non-negotiable rules, verbatim: `core/` imports no framework (ESLint-enforced); the LLM phrases, never decides; every AI touchpoint has a never-throw fallback; every displayed number traces to a function; no raw hex/bare HTML in components (token + primitive system).
- Data gotchas from the sample analysis: `"True"/"False"` capitalized booleans, semicolon-delimited lists, pipe-delimited `raw_history`, dates inside `departure_utc` are load-bearing for date-shift.
- Current phase pointer + definition of done, updated as you go.

---

## 2. Applications, accounts, and tools needed

**To build:**
- Node.js 20+, npm (or pnpm), Git
- A coding agent (Claude Code / Cursor) pointed at `/saarathi`
- VS Code or equivalent

**Services & keys (all free, all needed in Phase 0, none needed at demo time thanks to fallbacks):**
- **Groq API key** — free tier, no card. Set as `GROQ_API_KEY` in `.env.local`. ~30 req/min limit: never call `explain()` in a test loop; `explain.test.ts` tests the *fallback* path only.
- **transformers.js model** (`Xenova/all-MiniLM-L6-v2`) — downloads a few hundred MB on first run. **Trigger this in Phase 0**, not on demo day.
- **Nothing else.** No database, no vector-DB service (in-memory cosine similarity at 50 users), no deploy platform — the deliverable is a demo *video* plus code, so `localhost` is the production environment. Vercel deploy is a Phase 9 luxury, not a requirement.

**To submit:**
- Screen recorder for the 3–5 min video (OBS / Loom / QuickTime — whatever you already know; don't learn a tool tomorrow)
- Deck tool (Google Slides / PowerPoint / Canva)
- OneDrive account + a second browser profile or incognito window to verify "Everyone can view" from the outside
- Email client → careers@expediagroup.com before 11:45 PM IST

---

## 3. Phases

Budget assumes ~30 working-ish hours remain and you will sleep at least a little. Phases 1–4 are the floor; the cut line moves *up* the list as hours burn, never past Phase 4.

### Phase 0 — Scaffold & de-risk (1.5h) · tonight
- `create-next-app` in `saarathi-app` (TS, App Router, Tailwind); install deps: `papaparse`, `zod`, `zustand`, `@tanstack/react-query`, `langchain` + `@langchain/groq` + `@langchain/core`, `@xenova/transformers`, `vitest`, shadcn/ui init.
- Write `CLAUDE.md` (§1 above). Copy the full toolkit CSVs + `benchmark_prompts.json` into `data/`.
- **De-risk now, in order:** (1) run a one-off script that loads the embedding model and embeds one sentence — forces the model download tonight; (2) hit Groq once with the key — confirm auth; (3) **open `benchmark_prompts.json` and read every prompt** — confirm which users/routes/modes it expects, and adjust Phase 6 scope and §10.1 demo casting if it names specific ones; (4) load the *full* flights CSV once and print: row count, distinct routes, departure-date spread per route — this decides whether date-shift ships as specced or degrades to its empty state (spec flagged this as the one unverifiable assumption; verify it now).
- ESLint: add `react/forbid-elements` + the `core/` `no-restricted-imports` purity rule.
- **Done when:** app boots, model cached, Groq answers, benchmark prompts read and summarized into `CLAUDE.md`, date-density question answered.

### Phase 1 — Data layer (2h)
- `core/data.ts`: Papaparse at boot, `coerceFlightRow()` / `coerceUserRow()` (capitalized booleans, semicolon lists, pipe-split `raw_history`, ISO date extraction), `Map` indexes by origin and route, `globalThis` cache for dev hot-reload.
- `data.test.ts`: coercion truth table; index sanity against the real file; "unknown route → empty array, not throw."
- **Done when:** `getStore()` serves the 50k-row file in memory; a timed route lookup + log line proves sub-millisecond access (save that number — it goes in the deck).

### Phase 2 — Core decision engine (5h) · the judged artifact
- Port `preferences.ts`; upgrade evidence to `EvidenceItem { text, source, dimension }`; wire the embedding-archetype layer with its rule-only fallback.
- Extend `ranking.ts`: `FilterTrace` elimination counts, per-component `breakdown`, `selectAlternatives()` with all five kinds and honest `flight: null` empty states.
- **`counterfactuals.ts`** — the flagship. Closed-form price thresholds (fixed-normalization caveat in a comment), the five toggle perturbations, `applyPerturbations()`.
- `confidence.ts`: match%, tier, strong/weak signal partition from evidence-source agreement.
- Tests, in priority order: **break-even verification** (solve → set price $1 under threshold → re-rank → assert flip — this is the test a judge gets shown), alternatives empty categories, confidence tier boundaries, `benchmark.test.ts` running every toolkit prompt end-to-end (explanation on fallback path).
- **Done when:** `npx vitest run` is green including the benchmark suite. *No UI exists yet and that's correct.*
- **⑂ Option B fork:** if splitting repos, this `core/` lands in `saarathi-backend/src/core/`; scaffold NestJS + one `RecommendationController` now (+1.5h) and mirror `types.ts` into the frontend when Phase 4 starts.

### Phase 3 — API (1h)
- `POST /api/recommend` (zod-validated, orchestration only, assembles the full `RecommendResponse` incl. `trace` and echoed `appliedPerturbations`), `GET /api/users`.
- Port `explain.ts` behind it — grounded prompt extended with alternatives/counterfactuals/confidence; template fallback verified by unplugging the key once.
- **Done when:** one `curl` returns the complete contract for a benchmark prompt, with and without `GROQ_API_KEY` set.

### Phase 4 — Decision Screen, static (6h) · the floor
- `globals.css` tokens + Tailwind mapping (paste from architecture §6.2); fonts with the Fraunces/Inter fallback.
- Primitives first (`Container/Stack/Text/Button/Card/Badge/Skeleton/EmptyState`), then zones in demo-value order: `VerdictCard` + `ConfidenceBadge` → `EvidencePanel` → `OpportunityCostPanel` → counterfactual list (static text, not yet tappable) → `RankedListCollapse`. `TravelerPicker` + `RequestInput` in the header.
- **Done when:** selecting U02, U03, U05 produces three visibly different, screenshot-worthy decisions with real data in five zones. **This is the submission floor — if everything after this burns, you still record a winning video here.**

### Phase 5 — The demo moment (2h)
- `CounterfactualChip` → appends to `useDecisionStore.appliedPerturbations` → refetch → `PerturbationBanner` with reset. (~15 lines of state; the pipeline already does the work.)
- `TraceBar` + `TraceStageSheet` over the `trace` payloads (elimination counts, breakdowns, non-flips).
- **Done when:** tapping "you accept one stop" visibly changes the verdict and the banner appears; every trace node opens real data.

### Phase 6 — Multi-city through the same screen (3h)
- `multicity.ts`: graph from the route index, exact permutation search ≤6 cities, temporal sanity across legs, ordering counterfactual via the same break-even solver, itinerary-level alternatives.
- `ItineraryTimeline` as the zone-1 verdict; leg tap scopes zones 2–4.
- Scope to what `benchmark_prompts.json` actually asks (Phase 0 findings) — do not gold-plate past the prompts.
- **Done when:** a multi-city benchmark prompt renders through the six zones and the solver test proves the returned order beats a worse permutation.

### Phase 7 — Hardening pass (1.5h)
- Run all benchmark prompts through the *UI* by hand. Empty states everywhere data can be missing. Unplug the network → app degrades to template explanations without visible breakage. `npm run lint && npm run build` clean.
- **Done when:** you cannot break the screen with the toolkit's own inputs.

### Phase 8 — Deliverables (4h, PROTECTED — starts no later than ~6h before deadline)
Per spec §10 step 7, all four artifacts into `submission/`:
1. Solution summary (problem chosen → "booking is a decision problem" → Decision Screen + counterfactuals → impact).
2. Deck, 6–8 slides — include the architecture diagram from `architecture_v2.md` §1 and the sub-millisecond/50k line from Phase 1.
3. **Demo video 3–5 min, scripted:** cold open on the Decision Screen (10s silence — it explains itself, that's the point) → U03 family walkthrough → U02 reverse counterfactual → **the chip tap as climax** → trace panel peek → multi-city beat → close on the Saarathi/charioteer line from spec §12.
4. README: setup, assumptions (fixed-normalization caveat, OR-Tools note), limitations, future work.
5. OneDrive folder → "Everyone can view" → **verify from incognito** → email careers@expediagroup.com. Target send: 10:30 PM IST, not 11:44.
- **Done when:** the link opens in a browser that has never logged into your Microsoft account.

### Phase 9 — Only if hours remain, in this order
1. On-time-performance × connection-anxiety signal (spec §10 worth-if-time — build before anything below)
2. Staged reveal animation on first render
3. Vercel deploy (nice for judges who click, irrelevant to scoring)
4. Route map · 5. Pareto chart — last, per the standing cut

---

## 4. Clock map (IST, adjust to your reality)

| Window | Phases |
|---|---|
| Tonight, Jul 11 · 3–4h | Phase 0 + Phase 1, start Phase 2 |
| Jul 12 morning · ~5h | Finish Phase 2, Phase 3 |
| Jul 12 afternoon · ~6h | Phase 4 (floor secured), Phase 5 |
| Jul 12 evening · ~4h | Phase 6, Phase 7 |
| Jul 12, ~5:45 PM onward | **Phase 8, no matter what state the code is in** |

## 5. Standing rules while building

- The cut line only moves upward: threatened schedule → cut Phase 9, then 6, then 5's trace sheets. Never cut Phase 8, never cut the counterfactual verification test.
- Any hour spent past Phase 4 on visual polish before Phase 8 is complete is stolen from a judged deliverable.
- When an agent proposes a new abstraction, the test is spec §0's: does it land on the Decision Screen or in a mandatory deliverable? No → reject.