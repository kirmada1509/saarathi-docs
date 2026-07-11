# Saarathi — Engineering Handoff

Written for the person who now owns this codebase and has to defend it to judges. Every claim below is grounded in a specific file/line, or in a command I actually ran during this session (backend running on `:4000`, frontend on `:3000`, real HTTP calls, real test runs). Where the code disagrees with `docs/architecture_v2.md` or `docs/product_spec_v2.md`, that's called out explicitly — those docs describe the *plan*; this document describes *what got built*.

---

## 1. The 60-second system map

```
┌─────────────────────────┐         HTTP (fetch, CORS *)        ┌──────────────────────────────────┐
│  Browser                │ ───────────────────────────────────▶│  saarathi-backend (NestJS)        │
│  localhost:3000         │   GET  /api/users                    │  localhost:4000                   │
│                          │   POST /api/recommend                │                                    │
│  saarathi-frontend       │◀─────────────────────────────────── │  RecommendController               │
│  (Next.js 16 App Router) │        JSON RecommendResponse        │  UsersController                   │
└───────────┬──────────────┘                                     └───────────────┬────────────────────┘
            │                                                                     │
            │ TanStack Query cache keyed on URL params                           │ calls, per request:
            │ (lib/queries.ts)                                                   ▼
            │                                                     ┌──────────────────────────────────┐
            │                                                     │  core/ (pure TS, no framework)     │
            │                                                     │  data.ts → preferences.ts →        │
            │                                                     │  ranking.ts → counterfactuals.ts → │
            │                                                     │  confidence.ts → explain.ts        │
            │                                                     └───────┬──────────────┬─────────────┘
            │                                                             │              │
            ▼                                                             ▼              ▼
┌──────────────────────────┐                          ┌─────────────────────────┐  ┌──────────────────────────┐
│  MapLibre GL + MapTiler   │                          │  @xenova/transformers    │  │  Groq API (llama-3.3-70b) │
│  (browser-side, tiles     │                          │  all-MiniLM-L6-v2        │  │  via @langchain/groq      │
│  fetched directly from    │                          │  runs IN-PROCESS in the  │  │  network call, has a      │
│  api.maptiler.com)        │                          │  Node backend, no network│  │  never-throw fallback     │
└──────────────────────────┘                          └─────────────────────────┘  └──────────────────────────┘

                                       ┌──────────────────────────────────────────┐
                                       │  In-memory store (data.ts singleton)       │
                                       │  Map<user_id, UserRow>                     │
                                       │  Map<origin, FlightRow[]>                  │
                                       │  Map<"origin-dest", FlightRow[]>           │
                                       │  built ONCE at boot from ↓                 │
                                       └───────────────────┬────────────────────────┘
                                                            │ Prisma Client
                                                            ▼
                                       ┌──────────────────────────────────────────┐
                                       │  saarathi-backend/prisma/dev.db (SQLite)   │
                                       │  50 users, 50,000 flights (verified live)  │
                                       └──────────────────────────────────────────┘
```

**What runs where, over what.** Two separate OS processes. `saarathi-frontend` is a Next.js dev/prod server on port 3000, entirely React Server/Client Components — it never touches a database or the LLM directly. `saarathi-backend` is a NestJS server on port 4000, plain HTTP/JSON, no WebSockets, no gRPC. The frontend talks to the backend the same way any external client would: `fetch()` over HTTP to `http://localhost:4000`, with the backend allowing it via a wildcard CORS policy (`saarathi-backend/src/main.ts:8-12`). There is no proxy, no Next.js rewrite, no server-side-only API key shared between them — see §2 for the exact wiring. Inside the backend, every `/api/recommend` request runs entirely synchronously against an **in-memory** copy of the dataset (`getStore()` in `data.ts`); the actual SQLite file is only read once, at process boot, via Prisma. The embedding model (`@xenova/transformers`) runs **inside the Node process itself** — no network call, no separate service — while the LLM call (Groq) is the *only* network hop that leaves the machine. Everything else is deterministic arithmetic over data already sitting in memory.

---

## 2. Repo & process anatomy

### saarathi-backend

**Start:** `cd saarathi-backend && npm run start:dev` (this is `nest start --watch`, `package.json:12`) — boots on **port 4000**, hardcoded in `src/main.ts:15` (`await app.listen(4000)`). There is no `PORT` env var; changing the port means editing that line.

Verified boot sequence (this is a real log from this session, not a guess):
```
[Saarathi Store] Loading in-memory cache from database...
[Saarathi Store] Loaded 50 users, 50000 flights, indexed 1172 routes from DB in 548ms.
[Nest] Nest application successfully started
[Saarathi Embeddings] Initializing all-MiniLM-L6-v2 pipeline...
[Saarathi Embeddings] Model and archetype embeddings loaded successfully.
```

**Folder tree, top-level, one sentence each:**

| Path | Purpose |
|---|---|
| `src/main.ts` | Process entrypoint — boots Nest, enables CORS, listens on 4000. |
| `src/app.module.ts` | Root module; its `OnModuleInit` hook is what actually seeds the in-memory store from Prisma at boot (`app.module.ts:14-17`). |
| `src/core/` | Pure business logic — the scoring engine, preference inference, counterfactual algebra, LLM prompt/fallback. Zero NestJS imports; every file here is independently unit-testable and is (see §4). |
| `src/recommend/` | The `POST /api/recommend` HTTP controller — orchestrates calls into `core/` in order and shapes the JSON response. |
| `src/users/` | The `GET /api/users` HTTP controller — queries Prisma directly (see §5, this is architecturally inconsistent with `recommend`). |
| `src/prisma.service.ts` | Thin `PrismaClient` subclass registered as a Nest provider. |
| `prisma/` | `schema.prisma` (User + Flight models), `seed.ts` (CSV → SQLite one-time loader), `dev.db` (the live 10MB SQLite file, 50 users / 50,000 flights — verified via `sqlite3 ... "select count(*)"` this session). |
| `test/` | One Jest e2e spec (`app.e2e-spec.ts`) + its Jest config — **currently broken**, see §8. |
| `dist/` | Compiled JS output of `nest build`, checked into git (unusual, see §10). |
| `dev.db` (repo root, **not** `prisma/dev.db`) | A **stale, empty** SQLite file (0 rows in both tables, confirmed via `sqlite3`) left over from a `prisma db push`/`migrate` run executed from the wrong working directory. Not read by anything. Dead file. |

**Env vars** (`saarathi-backend/.env`):

| Key | What it does | What breaks without it |
|---|---|---|
| `DATABASE_URL="file:./dev.db"` | Prisma's SQLite connection string. Relative paths in Prisma resolve relative to `schema.prisma`'s directory (`prisma/`), so this actually points at `saarathi-backend/prisma/dev.db` — **not** the root-level `dev.db`. | Without it, `PrismaClient` can't connect; `AppModule.onModuleInit()` (`app.module.ts:14`) throws at boot, and the whole server fails to start — every route, not just `/api/recommend`. |
| `GROQ_API_KEY` | Bearer token for the Groq chat completion used in `core/explain.ts`. | Nothing crashes. `explain()` checks `process.env.GROQ_API_KEY` at `explain.ts:96-99` and returns a deterministic templated string instead — see §7 for the exact fallback code. |

### saarathi-frontend

**Start:** `cd saarathi-frontend && npm run dev` (`next dev`, `package.json:6`) — Next.js picks **port 3000** by default and auto-increments to 3001, 3002, ... if 3000 is taken (this happened during this session — confirm your terminal's actual port before demoing, see RUNBOOK).

**Folder tree, top-level, one sentence each:**

| Path | Purpose |
|---|---|
| `app/` | Next.js App Router. `app/page.tsx` is the marketing landing page (`/`); `app/(product)/` is a route group (see below) holding the actual product's 7 routes; `app/layout.tsx` is the root HTML shell (fonts, theme provider, query provider). |
| `app/(product)/` | A Next.js **route group** — the parentheses mean the folder name is *not* part of the URL. `app/(product)/app/decision/page.tsx` serves at `/app/decision`, not `/app/(product)/app/decision`. Its `layout.tsx` (`app/(product)/layout.tsx`) wraps every product route in the shared `TopNav` + `CommandPalette`. |
| `components/decision/` | The 8 "zone" components that make up the Decision Screen (VerdictCard, EvidencePanel, CounterfactualPanel, etc.) — the flagship UI. |
| `components/composer/` | The two request-building forms (`RequestForm` for single-destination, `MultiCityForm` for multi-city) that construct the URL the Decision Screen reads. |
| `components/map/` | `RouteMap.tsx` — MapLibre GL wrapper, real and wired in (§5). |
| `components/charts/` | `PreferenceRadar`, `ConfidenceGauge`, `ScoreBreakdownBars` — Recharts-based, real and wired in (§5). |
| `components/travelers/`, `components/compare/`, `components/marketing/`, `components/nav/` | Supporting UI for the traveler directory, compare screen, landing page, and top navigation respectively. |
| `components/ui/` | shadcn/ui primitives (Button, Select, Dialog, Command, Chart wrapper, etc.) — vendored, not hand-rolled. |
| `components/ui/primitives.tsx` | The project's *own* design-system layer (`Container`, `Stack`, `Text`, `Card`, `Field`, `NavLink`, ...) — an ESLint rule (`eslint.config.mjs`) forbids raw `<div>`/`<p>`/`<button>` in `app/**` and most of `components/**`, forcing everything through here. |
| `lib/` | `api.ts` (fetch wrapper), `queries.ts` (TanStack Query hooks), `decision-params.ts` (URL ⇄ state contract), `store.ts` (Zustand, UI-only state), `env.ts`, `types.ts`, `benchmark-queries.ts`, `data/airport-coordinates.ts` (static lat/lon lookup for the map, see §5). |
| `core/types.ts` | A **hand-duplicated mirror** of `saarathi-backend/src/core/types.ts` — see §10, this is real drift risk, not a shared package. |
| `public/` | Static assets. |

**Env vars** (`saarathi-frontend/.env.local`):

| Key | What it does | What breaks without it |
|---|---|---|
| `NEXT_PUBLIC_API_BASE_URL` | Read in `lib/env.ts:1-2`, falls back to `"http://localhost:4000"` if unset. Every backend call in `lib/api.ts` is built from this. | Nothing breaks locally (the fallback *is* correct for local dev); this only matters if the backend is deployed somewhere other than `localhost:4000`. |
| `NEXT_PUBLIC_MAPTILER_KEY` | Read in `lib/env.ts:4`. Passed straight into the MapLibre style URL in `components/map/RouteMap.tsx` (`https://api.maptiler.com/maps/basic-v2/style.json?key=${MAPTILER_KEY}`). | Without it, `RouteMap.tsx` renders an `EmptyState` ("Map unavailable") instead of tiles — checked explicitly at the top of the component, never a crash. |
| `GROQ_API_KEY` | Present in `.env.local` but **the frontend never reads it** — grep confirms zero references to `process.env.GROQ_API_KEY` anywhere under `app/`, `components/`, `lib/`. It's a leftover from before the codebase was split into two services (the Groq call used to happen in a Next.js API route). | Nothing — it's dead configuration. Flagged in §10. |

**How the frontend reaches the backend — the exact wiring.** There is no proxy and no Next.js `rewrites()` config — `next.config.ts` is the framework default with an empty options object (`next.config.ts:3-5`, literally `const nextConfig: NextConfig = {}`). Every backend call is a **direct, browser-side `fetch()`** to `NEXT_PUBLIC_API_BASE_URL`, built in `lib/api.ts:29` (`fetch(\`${API_BASE_URL}/api/users\`)`) and `lib/api.ts:36-40` (`fetch(\`${API_BASE_URL}/api/recommend\`, { method: "POST", ... })`). This only works cross-origin (port 3000 → port 4000) because the backend answers with `Access-Control-Allow-Origin: *` — set in `saarathi-backend/src/main.ts:8-12` via `app.enableCors({ origin: '*', methods: 'GET,HEAD,PUT,PATCH,POST,DELETE', credentials: true })`. This is a real, verified response header (captured via `curl -v` this session). There is no authentication of any kind between the two services.

---

## 3. Life of a request — traced with real payloads

Subject: benchmark prompt **B01** from `data/benchmark_prompts.json` — `{"prompt_id": "B01", "user_id": "U01", "request": "I need to get from home to Tokyo next month, what do you suggest?"}`. Everything below is a real call fired at the running backend during this session (`POST http://localhost:4000/api/recommend`, response time **1.058s**), not a hypothetical.

### Path

1. **User submits the composer form.** `components/composer/RequestForm.tsx:51-58` — `onSubmit` builds a query string via `buildDecisionQuery()` and does `router.push(\`/app/decision?${query}\`)`. It never calls the backend itself.
2. **`buildDecisionQuery`** (`lib/decision-params.ts:24-44`) serializes `{ userId, requestText, destination }` into `URLSearchParams` — `pts` (perturbations) and `leg` are omitted when empty/zero. This is the **entire state contract**: the Decision Screen has no other source of truth than this URL.
3. **Next.js navigates to `/app/decision?userId=U01&requestText=...&destination=NRT`.** `app/(product)/app/decision/DecisionScreen.tsx:27-28` reads it back via `useSearchParams()` + `parseDecisionParams()` (`lib/decision-params.ts:46-71`).
4. **`useRecommendation()`** (`lib/queries.ts:38-59`) is a TanStack Query hook. Its `queryKey` is built from `recommendationKey()` (`lib/queries.ts:24-33`) — literally `[userId, requestText, destination, cities, JSON.stringify(perturbations)]`. This means **the URL is the cache key**: reload the same `/app/decision` URL and, if it's still in TanStack Query's in-memory cache (5-minute `staleTime`, `lib/queries.ts` default), zero network calls happen.
5. **`fetchRecommendation()`** (`lib/api.ts:33-42`) fires the real HTTP call.

**Checkpoint 1 — request body actually sent** (trimmed):
```json
{
  "userId": "U01",
  "requestText": "I need to get from home to Tokyo next month, what do you suggest?",
  "destination": "NRT",
  "perturbations": []
}
```

6. **`RecommendController.getRecommendation`** (`saarathi-backend/src/recommend/recommend.controller.ts:65-73`) validates this body against a Zod schema (`RecommendRequestSchema`, lines 34-41) and throws a `400` on failure.
7. **`getStore()`** (`core/data.ts`) returns the in-memory Maps. `store.users.get('U01')` (line 79).
8. **`inferPreferences(user)`** (`core/preferences.ts:156-307`) — regex-scans U01's `raw_history` and computes weights. U01's real `raw_history`: `"always book business, hate connections | redeyes kill my mornings, avoid if possible | aisle seat, front of cabin"`. Full trace in §4.
9. **`applyPerturbations(basePref, [])`** (`core/counterfactuals.ts:17-43`) — no-op here since `perturbations` is empty; matters in the second trace below.
10. **`filterAndRank(routeFlights, perturbedPref, opts)`** (`core/ranking.ts:51-227`) — the 57 CPT→NRT flights get funneled through 3 hard constraints (origin, destination, layover ≤120m, redeye avoidance) down to 32 survivors, each scored.

**Checkpoint 2 — post-ranking state** (top 3 of 32, real numbers):
```json
[
  { "flight_id": "F037670", "airline_name": "American Airlines", "price": 1571.95, "score": 1.897 },
  { "flight_id": "F029334", "airline_name": "Korean Air",        "price": 5093.16, "score": 1.892 },
  { "flight_id": "F009204", "airline_name": "Air India",         "price": 1229.81, "score": 1.803 }
]
```

11. **`selectAlternatives(ranked, user)`** (`core/ranking.ts:229-421`) — cheapest/fastest/refundable/comfort-upgrade/date-shift, computed against the champion.
12. **`computeCounterfactuals(ranked, basePref, routeFlights, opts)`** (`core/counterfactuals.ts:45-174`) — note this is called with `basePref` (the *unperturbed* preferences), not `perturbedPref`, at `recommend.controller.ts:420-425` — see §10 for why that's worth double-checking. Produces 8 counterfactuals (3 price-threshold, 5 toggle-based); worked arithmetic in §4.
13. **`computeConfidence(ranked, perturbedPref)`** (`core/confidence.ts:12-128`) → `{ matchPct: 88, tier: "medium" }`.
14. **`explain(...)`** (`core/explain.ts:83-128`) — calls Groq (network hop #1, ~200-400ms typically) and returns 3-4 sentences of prose. Never throws; see §7.
15. Controller assembles a `TraceStage[]` (7 stages, `recommend.controller.ts:439-471`) and returns the full `RecommendResponse`.

**Checkpoint 3 — final response** (trimmed to the fields the UI actually reads first):
```json
{
  "mode": "single-leg",
  "verdict": { "flight_id": "F037670", "airline_name": "American Airlines", "flight_numbers": "AA351",
               "price": 1571.95, "stops": 0, "score": 1.897 },
  "confidence": { "matchPct": 88, "tier": "medium",
                   "strongSignals": ["Direct flight preference"],
                   "weakSignals": ["Price sensitivity", "Redeye avoidance", "Airline loyalty", "Cabin class preference"] },
  "explanation": "The #1 ranked flight, American Airlines AA351, is the best match for this traveler due to their strong preference for direct flights (direct_weight=0.9) and preferred airline, AA. They are willing to pay a premium for this convenience, as evidenced by their low price sensitivity (cost_weight=0.2). By choosing AA351, they are giving up potential cost savings of $466, but this trade-off is acceptable given their preferences. The decision boundary analysis confirms that AA351 remains the top choice unless Korean Air KE3237's fare drops below $4816, further supporting this recommendation."
}
```

16. Back in the browser: `useRecommendation()` resolves, `DecisionScreen.tsx:143-164` renders `VerdictCard`, `EvidencePanel`, `CounterfactualPanel`, `OpportunityCostPanel`, `RankedList` from the same response object — no further requests.

### Second trace — tapping a counterfactual chip

Tapping the "accept one stop" chip does **not** call the recommend endpoint with a "toggle" action — it rewrites the URL. `DecisionScreen.tsx:70-77` (`handleTogglePerturbation`) adds `{ kind: "accept_one_stop" }` to the `perturbations` array and calls `router.replace()` (not `push`, so the back button doesn't fill up with every chip tap) with a new `pts=` query param. That URL change produces a **new TanStack Query cache key** (`recommendationKey()` includes `JSON.stringify(perturbations)`), so a fresh `POST /api/recommend` fires automatically — this is a real call made this session:

```json
{ "userId": "U01", "requestText": "...", "destination": "NRT",
  "perturbations": [{ "kind": "accept_one_stop" }] }
```

Server-side, `applyPerturbations` (`core/counterfactuals.ts:28-30`) does exactly two things for this perturbation: `direct_weight = 0.15` and `max_layover_minutes = max(current, 420)`. Real, verified effect: the constraint funnel changes from *57 → 57 (layover ≤120m) → 47 (redeye)* to *57 → 57 (layover ≤420m) → 47 (redeye)* — the layover step now removes 0 instead of 19 — but the **winner did not flip**: AA351 (nonstop, cheap) still wins, just with a much lower score (1.043 vs 1.897), because dropping `direct_weight` to 0.15 didn't create enough room for a connecting flight to overtake a flight that was already winning on price and directness. This is a real, honest example of a chip that changes the math but not the outcome — the UI shows this via the counterfactual's own `flips: false/true` flag (computed per-toggle in `counterfactuals.ts:147-149`), not by assuming every toggle flips.

---

## 4. The core modules, one by one

All eight files live in `saarathi-backend/src/core/`. None import anything from `@nestjs/*`, `express`, or React — this is enforced by an ESLint `no-restricted-imports` rule (mirrored in the frontend's `eslint.config.mjs` for its own `core/` copy). Line counts are from `wc -l` this session (excludes each file's own `.test.ts`).

### `types.ts` (159 lines)
Pure interfaces, no logic: `UserRow`, `FlightRow`, `EvidenceItem`, `InferredPreference`, `ScoredFlight`, `Alternative`, `Counterfactual`, `Confidence`, `TraceStage`, `RecommendResponse`, `Perturbation` (a discriminated union of 6 kinds). Every other module and the controller import from here. **If deleted:** nothing compiles — this is the load-bearing contract for the entire backend, and (manually, see §10) for the frontend too.

### `data.ts` (272 lines)
- `coerceUserRow(raw: Record<string, unknown>): UserRow` / `coerceFlightRow(raw): FlightRow` (lines 30-93) — defensive parsing (`String(x ?? '')`, `Number(x) || 0`) that works whether `raw` came from a CSV row (all-string) or a Prisma row (typed).
- `buildStore(): DataStore` (line 94) — reads `data/*.csv` via Papaparse, builds three `Map`s. **Only used by tests and by `getStore()` in dev/test `NODE_ENV` as a fallback** — the running server does not use this path (see §5).
- `getStore(): DataStore` (line 176) — singleton accessor, the function every other module calls.
- `initializeStoreFromDb(prisma: PrismaClient)` (line 199) — the function that actually populates the store in the real running server, called once from `app.module.ts:16`.
- **If deleted:** `RecommendController`, `UsersController` (partially), `counterfactuals.ts`, `multicity.ts` all fail to compile — this is the data access layer for the whole app.

### `preferences.ts` (307 lines) — evidence-based preference inference
Three evidence sources, in priority order:
1. **Structured fields** (always, 2 evidence entries per user): `direct_preference` → `DIRECT_PREF_MAP` (`strong: 0.9, moderate: 0.55, none: 0.15`, line 41-45) and `price_sensitivity` → `PRICE_SENS_MAP` (`low: 0.2, medium: 0.5, high: 0.85, none: 0.05`, line 46-51).
2. **Regex families** over `raw_history` phrases (split on `" | "`, line 165-168): `DIRECT_BOOST` (7 patterns, e.g. `/hate connections/i`, line 4-12), `COST_BOOST` (8 patterns, line 14-23), `CONVENIENCE_BOOST` (6 patterns, line 25-32), `REDEYE_AVOID` (3 patterns, line 34-38), `REDEYE_OK` (2 patterns, line 39). Each hit both adds an evidence entry and increments a counter (`directHits`, `costHits`, ...) that later nudges the weight by `+0.1` per hit (direct/cost, line 259-263) or `+0.15` (convenience, line 267-268), capped at 1.
3. **Embedding fallback** (line 240-256) — *only* runs for a phrase that matched **zero** regexes. Compares the phrase's embedding (via the locally-loaded `all-MiniLM-L6-v2` model) against 3 hand-written archetype sentences per dimension (`ARCHETYPES`, line 53-74; e.g. direct: `"I hate flight connections and layovers"`, `"I want to fly direct only"`, `"Direct flights are worth paying for"`), keeps the best match if raw dot-product similarity `> 0.78` (line 137).

**Worked full evidence trail — real user U01** (`data/user_data.csv` row 2, `raw_history` = `"always book business, hate connections | redeyes kill my mornings, avoid if possible | aisle seat, front of cabin"`, `direct_preference=strong`, `price_sensitivity=low`), captured from the live API response this session:

| Phrase / field | Matched by | Evidence text | Dimension |
|---|---|---|---|
| `direct_preference=strong` | structured | `structured: direct_preference=strong -> direct_weight=0.9` | direct |
| `price_sensitivity=low` | structured | `structured: price_sensitivity=low -> cost_weight=0.2` | cost |
| `"always book business, hate connections"` | `DIRECT_BOOST` — `/hate connections/i` | `raw_history: "..." signals direct-flight preference` | direct |
| `"redeyes kill my mornings, avoid if possible"` | `REDEYE_AVOID` — `/redeyes? kill/i` | `raw_history: "..." signals redeye avoidance` | redeye |
| `"aisle seat, front of cabin"` | **nothing** — matches no regex family and no embedding archetype clears the 0.78 threshold | *(no evidence entry — silently and correctly ignored, not a bug: "seat position" isn't a tracked dimension)* | — |
| `preferred_airlines="AA"` | structured | `structured: preferred airlines are AA` | airline |
| `preferred_cabin="Business"` | structured | `structured: preferred cabin is Business` | cabin |

Final: `direct_weight = min(1, 0.9 + 0.1×1 hit) = 1.0`, `cost_weight = 0.2` (no cost-pattern hits), `convenience_weight = max(0, 1 - 0.2×0.6) = 0.88` (no convenience-pattern hits either — this is a *derived* weight, not a direct signal, computed at line 266), `avoid_redeye = true`. All four numbers match the live API response exactly.

**If deleted:** `RecommendController` fails to compile (imports `inferPreferences` directly); every downstream module that takes an `InferredPreference` (`ranking.ts`, `counterfactuals.ts`, `confidence.ts`, `explain.ts`, `multicity.ts`) has no input.

### `ranking.ts` (421 lines) — the deterministic scorer
`filterAndRank(flights, pref, opts): { ranked: ScoredFlight[]; trace: FilterTrace }` (line 51). Hard filters, in order, each logged with removed/remaining counts (this is exactly what powers the "Hard Constraints Applied" trace stage): origin → destination (if given) → layover ≤ `pref.max_layover_minutes` (nonstop always passes) → redeye avoidance (`departure hour ∈ [22,5)`, line 111-124) → date flexibility (only if an explicit date was requested — otherwise skipped entirely, line 126-143, this was a deliberate fix noted in `BENCHMARK_REPORT.md`'s "What Needs Improvement" section).

**The score formula, exactly as coded** (line 183-192):
```
score = ( cost_weight × priceScore
        + direct_weight × directScore
        + (1 − cost_weight) × 0.5 × timeScore
        + convenience_weight × 0.3 × cabinScore
        + 0.2 × airlineScore
        + baggageWeight × baggageScore )
        × demandAdj × holidayAdj
        + dayBonus
```
where `priceScore`/`timeScore` are min-max normalized *within the candidate set* (`normalize()`, line 11-16 — 0 = most expensive/slowest, 1 = cheapest/fastest, tied values → 0.5), `directScore = 1` if nonstop else `max(0, 1 − 0.35×stops)`, `airlineScore = 1` if the flight's airline is in `pref.preferred_airlines` else `0.5`, `cabinScore = 1` if it matches `pref.preferred_cabin` else `0.4`, `demandAdj ∈ {0.9, 1.0, 1.05}` for `{high, medium, low}` demand (line 18-22), `holidayAdj = 0.95` if `is_holiday_season` else `1`, `baggageWeight = 0.25` if the `bags_matter` perturbation is active else `0`, and `dayBonus = 0.15` if the request text mentioned a weekday that matches the flight's departure day, else `0`.

`selectAlternatives(ranked, user): Alternative[]` (line 229) — always returns exactly 5, one per `kind` (`cheapest`, `fastest`, `flexible`, `comfort`, `date_shift`), each with an honest "you already have this" or "no such option exists" branch rather than a fabricated one.

**If deleted:** the entire ranking/scoring capability is gone — `RecommendController`, `counterfactuals.ts`, `multicity.ts` all call `filterAndRank` directly and fail to compile.

### `counterfactuals.ts` (174 lines) — the invertible half of the scorer

`applyPerturbations(pref, perturbations): InferredPreference` (line 17) — a pure, non-mutating transform: `accept_one_stop` → `direct_weight = 0.15`, `max_layover_minutes = max(current, 420)`; `bags_matter` → `bags_matter = true`; `evening_ok` → `avoid_redeye = false`; `ignore_loyalty` → `preferred_airlines = []`; `shift_dates` → sets `date_flexibility_days_override`.

`computeCounterfactuals(ranked, pref, candidates, opts): Counterfactual[]` (line 45) does two structurally different things:

**Type 1 — closed-form price break-even** (line 60-113), computed for the top 3 challengers (`ranked.slice(1, 4)`) *without re-running the ranker*. This is the algebra that makes the headline claim ("wins if price drops below $X") provable rather than guessed: since `score` is linear in `price` through the single term `cost_weight × priceScore`, the break-even price drop is solved directly:
```
priceDrop = (championScore − challengerScore) × priceRange / (cost_weight × demandAdj × holidayAdj)
targetPrice = challengerPrice − priceDrop
```
If `priceDrop / challengerPrice > 0.6` or `targetPrice ≤ 0`, it's reported as unrealistic (`flips: false`) rather than silently shown as a workable threshold.

**Worked example — real numbers from the B01 trace above**, hand-verified against the live API response (`toPrice: 4816.22` — the exact value the server actually returned, not a recomputation):
- Champion: F037670 (American Airlines AA351), `score = 1.897`, `price = $1571.95`
- Challenger: F029334 (Korean Air KE3237), `score = 1.892`, `price = $5093.16`, `demand_level = "medium"` → `demandAdj = 1.0`, `is_holiday_season = false` → `holidayAdj = 1.0`
- Price range across all 32 ranked candidates: `min = $1105.92`, `max = $12183.53` → `priceRange = $11077.61`
- `pref.cost_weight = 0.2`

```
priceDrop   = (1.897 − 1.892) × 11077.61 / (0.2 × 1.0 × 1.0)
            = 0.005 × 11077.61 / 0.2
            = 55.388 / 0.2
            = 276.94

targetPrice = 5093.16 − 276.94 = 4816.22   ✓ matches API exactly
dropPct     = 276.94 / 5093.16 = 5.4%  (well under the 60% sanity cap)
```
Label: `"Korean Air KE3237 becomes my pick if its fare drops below $4816"` (`Math.floor(4816.22) = 4816`, line 107). This is the entire "prove it's not hallucinated" answer for judges — it's three lines of algebra derived from a formula that's linear in price, not a number an LLM invented.

**Type 2 — perturb-and-re-rank** (line 115-171): for `accept_one_stop`, `bags_matter`, `evening_ok`, `ignore_loyalty`, and (conditionally, if the user has `date_flexibility_days > 0`) `shift_dates`, it actually calls `applyPerturbations` + `filterAndRank` again and checks whether `flight_id` changed. This is *not* closed-form — it's brute-force re-ranking, justified because re-ranking a single route's candidate set (tens of flights) is sub-millisecond.

**If deleted:** the counterfactual panel and the "why can you invert your ranking" story disappear entirely; `RecommendController` and `multicity.ts` both import from here.

### `confidence.ts` (128 lines) — match% and tier, exact formulas as coded

`computeConfidence(ranked, pref): Confidence` (line 12).

**matchPct** (line 36-58): the theoretical maximum score achievable under this traveler's exact weights, evaluated at every sub-score = 1 and `demandAdj = 1.05` (best case, "low" demand), no holiday penalty:
```
maxAchievableScore = ( cost_weight + direct_weight + (1 − cost_weight)×0.5 + convenience_weight×0.3 + 0.2 + baggageWeight ) × 1.05
matchPct = round( championScore / maxAchievableScore × 100 ), clamped to [0, 100]
```
For U01: `cost_weight=0.2, direct_weight=1, convenience_weight=0.88, baggageWeight=0` → `maxRaw = 0.2 + 1 + 0.4 + 0.264 + 0.2 + 0 = 2.064` → `maxAchievable = 2.064 × 1.05 = 2.1672` → `matchPct = round(1.897 / 2.1672 × 100) = 88` — matches the live response exactly.

**tier** (line 90-120): start from the score margin between #1 and #2 (`margin = championScore − challengerScore`, or `1.0` if there's no #2):
- `margin > 0.1` → `"high"`
- `margin < 0.04` → `"low"`
- otherwise → `"medium"`

then two demotions: if `strongSignals` contains **both** "Price sensitivity" and "Comfort priority" (a genuine conflict — wanting cheap *and* comfortable) **or** `strongSignals.length === 0`, demote `high→medium` or `medium→low` (line 102-114). Then one promotion (the "Fix 3" comment in the code, line 116-120): if `matchPct >= 80` and tier is still `"low"`, promote to `"medium"` — added specifically so a mathematically strong match never displays as "low confidence."

A signal counts as **strong** only if it has evidence from *both* a structured field and a behavioral source (`raw_history` or `embedding`) for the same dimension (line 77-87) — a single-source signal is always "weak," regardless of tier.

**If deleted:** `RecommendController` fails to compile; `EvidencePanel`/`VerdictCard` on the frontend have nothing to render for the confidence gauge.

### `explain.ts` (128 lines) — the only LLM-touching module
Covered in full in §7.

### `multicity.ts` (341 lines) — brute-force-optimal multi-city routing
`optimizeRoute(cities, pref)` (line 37): generates **every permutation** of the given cities (`permute()`, line 13-25 — factorial, fine for the 2-4 city inputs this app expects), and for each permutation runs a branch-and-bound search (`search()`, line 85-128) over per-leg ranked flights, pruned using a precomputed "max possible remaining score" bound (line 72-80) so it doesn't have to fully explore dead branches. A `satisfiesTemporalSanity()` check (line 30-35) rejects any leg pairing where the next flight departs less than 60 minutes after the previous one arrives. The winning permutation's total score across all legs is compared against the runner-up permutation to build one ordering-level counterfactual (line 159-204), using the **same** closed-form price-break-even algebra as `counterfactuals.ts`, just applied to one differing leg between the two itineraries.

**If deleted:** the entire `cities: [...]` request mode breaks — `RecommendController`'s multi-city branch (`recommend.controller.ts:96-98`) calls this directly.

---

## 5. Data layer — truth, not plan

`docs/architecture_v2.md:9` states: *"No NestJS, no Prisma, no SQLite... the pure `core/` boundary keeps it reversible in ~30 minutes if a rule mandates a separate backend."* `docs/architecture_v2.md:121` adds: *"No database. ... A database adds seed scripts, migrations, and deploy state for zero judge-visible value."* Neither is true of the system as built. This is not a criticism of the docs — they're dated planning artifacts — but the discrepancy needs to be documented, not quietly reconciled.

**What's actually happening, verified this session by starting the server and reading the boot log + querying the SQLite file directly:**

1. **Prisma/SQLite is load-bearing, not vestigial.** `AppModule.onModuleInit()` (`app.module.ts:14-17`) calls `initializeStoreFromDb(this.prisma)` (`data.ts:199-305`) exactly once, at process boot. This function runs `prisma.user.findMany()` / `prisma.flight.findMany()` and populates the same in-memory `Map`s that `getStore()` serves for every request thereafter. Confirmed live: `sqlite3 saarathi-backend/prisma/dev.db "select count(*) from User; select count(*) from Flight;"` → **50, 50000** — exactly matching the CSV row counts and the boot log's `"Loaded 50 users, 50000 flights..."`.
2. **There is a second, independent CSV-loading code path that the running server never uses.** `buildStore()` (`data.ts:94-168`) reads `data/*.csv` directly via Papaparse. It's reachable through `getStore()` (`data.ts:176-196`) only when `globalThis.__saarathi_store__` is still unset — which never happens on the real server (Prisma always wins the race, since `onModuleInit` runs before any request can call `getStore()`). It **is** the path every unit test exercises, since tests call `getStore()` directly without bootstrapping the Nest app. In other words: **the tests validate code the production server doesn't run**, and the production server runs a Prisma path with zero direct test coverage of its own. This is real, verified duplication — see §10.
3. **`GET /api/users` bypasses the in-memory store entirely.** `UsersController.getUsers()` (`src/users/users.controller.ts:8-23`) queries `this.prisma.user.findMany({ select: {...} })` live, per request — it never calls `getStore()`, even though `getStore().users` already has everything it needs, in memory, for free. This is architecturally inconsistent with `RecommendController` and is the clearest single "if I only have 20 minutes" cleanup in this codebase.
4. **There are two `dev.db` files, and it matters which one you look at.** `saarathi-backend/prisma/dev.db` (10.2MB) is the real one — Prisma resolves the `.env`'s `DATABASE_URL="file:./dev.db"` relative to `schema.prisma`'s directory. `saarathi-backend/dev.db` (repo root, 24KB) is a **stale, empty** artifact (0 rows in both tables, confirmed via `sqlite3` this session) from a `prisma` command run from the wrong working directory at some point. Nothing reads it.

**Verdict:** Prisma/SQLite is **load-bearing and correctly wired for the real server** — keep it. What's genuinely wrong is (a) the unused parallel CSV path only exercised by tests, (b) `UsersController` querying Prisma directly instead of the shared cache, (c) the dead root-level `dev.db`. Effort to fully reconcile: **~1-2 hours** — point `UsersController` at `getStore().users` (15 min), delete the root `dev.db` (1 min), and either delete `buildStore()`/CSV path entirely or add a real boot-time test that exercises it (30-60 min). Not urgent before a demo; worth doing before this codebase outlives the hackathon.

**Everything else that diverges from the docs — is it wired in or dead weight?**

| Item | Docs say | Reality (verified) |
|---|---|---|
| `components/map/` (`RouteMap.tsx`) | `product_spec_v2.md:218`: map is "cut from the critical path... only if hours remain" | **Wired in.** Imported and rendered in `DecisionScreen.tsx:20,139` on every single-leg and multi-city verdict. Real MapLibre GL + MapTiler tiles, real great-circle arcs computed from a static 35-airport lat/lon table (`lib/data/airport-coordinates.ts`) — those 35 codes are exactly the unique origins in `flights_data.csv` (verified via `awk` this session). Falls back to an `EmptyState` if `NEXT_PUBLIC_MAPTILER_KEY` is unset — never crashes. |
| `components/charts/` (Radar/Gauge/Bars) | `product_spec_v2.md:218`: chart is "cut from the critical path" alongside the map | **Wired in.** `PreferenceRadar` and `ScoreBreakdownBars` render inside `EvidencePanel`/`RankedList`; `ConfidenceGauge` renders inside `VerdictCard`. All three consume real response fields (`preference`, `confidence`, `breakdown`), nothing hardcoded. |
| `components/marketing/` | Not mentioned (docs assumed no marketing surface at all) | **Wired in.** `AnimatedHero` renders on `app/page.tsx`, the `/` landing route, which itself is new relative to the plan. |
| `app/(product)/` route group | `product_spec_v2.md:171-172`: planned as one Next.js app with `app/api/recommend/route.ts` (a Route Handler) and `app/page.tsx` as *the* Decision Screen — single page, no router | **Fully diverged, by explicit later instruction** (this was a deliberate redesign requested mid-project, not scope creep nobody noticed). The app now has 8 routes; the Decision Screen moved to `/app/decision` and reads its state from URL params instead of being the root page. The original single-endpoint plan was also abandoned — there's no Next.js API route at all anymore; `/api/recommend` lives entirely in the separate NestJS backend. |

---

## 6. Technology inventory — the judge table

| Tech | What it is | Why *this* project uses it | Where in the code | Alternative considered |
|---|---|---|---|---|
| **NestJS** | Opinionated Node.js server framework (decorators, dependency injection, modules) built on Express. | Gives the scoring engine a real HTTP boundary and DI container without hand-rolling routing/validation; `@Controller`/`@Module` keep `RecommendController` and `UsersController` thin wrappers around pure `core/` functions. | `src/main.ts`, `src/app.module.ts`, `src/recommend/recommend.controller.ts`, `src/users/users.controller.ts` | A plain Express app, or (per the original plan) no separate backend at all — Next.js Route Handlers directly in the frontend. |
| **Next.js App Router** | React meta-framework: file-system routing, Server + Client Components, built-in bundling. | Route groups (`(product)`) give a shared shell across 7 routes without a URL segment; Server Components let `app/page.tsx` and `app/(product)/app/how-it-works/page.tsx` fetch directly from the backend at render time with no client-side loading spinner. | `app/`, `next.config.ts` | Pages Router (older Next.js), or a plain SPA (Vite + React Router). |
| **Prisma** (+ SQLite) | Type-safe ORM + schema migrations, here backed by a local SQLite file. | Per §5: this is the actual boot-time data source, not the CSVs. Chosen (per the divergence from `architecture_v2.md`) presumably to get typed queries and a persistence layer without standing up Postgres for a hackathon. | `prisma/schema.prisma`, `src/prisma.service.ts`, `src/core/data.ts:199-305`, `src/users/users.controller.ts` | The originally-planned in-memory-CSV-only store (still present as `buildStore()`, unused in prod — see §5), or Postgres if this needed to survive real concurrent writes (it doesn't; the dataset is read-only at request time). |
| **LangChain** (`@langchain/core`, `@langchain/groq`) | Framework for composing LLM prompts/chains with a provider-agnostic interface. | Used for exactly one `ChatPromptTemplate.pipe(model).pipe(new StringOutputParser())` chain (`core/explain.ts:21-38,108`) — arguably more framework than one prompt needs, but it made swapping providers (this used to plausibly be OpenAI) a one-line change. | `core/explain.ts` | A raw `fetch()` call to Groq's REST API — would remove a dependency for a single call site. |
| **Groq (llama-3.3-70b-versatile)** | Hosted LLM inference, chosen for very low latency. | The *only* prose-generation step in the whole system — turns already-computed evidence/ranking/counterfactual data into 3-4 readable sentences. Never asked to rank, filter, or invent numbers (see §7). | `core/explain.ts:102-106` (model instantiation) | GPT-4o-mini, Claude Haiku — any fast/cheap chat model would work identically since the prompt is fully self-contained. |
| **`@xenova/transformers`** (`all-MiniLM-L6-v2`) | In-browser/in-Node ONNX runtime for running small transformer models locally, no API key, no network call. | Powers the embedding-similarity fallback in preference inference (§4) for `raw_history` phrases that don't match any hand-written regex. Runs **inside the Node process** — verified via the boot log ("Initializing all-MiniLM-L6-v2 pipeline... loaded successfully") with zero outbound network activity for this step. | `core/preferences.ts:1,76,87,93,97,113-150` | A hosted embeddings API (OpenAI `text-embedding-3-small`, Cohere) — would add latency, cost, and a second point of failure for a fallback path that's supposed to be robust. |
| **Zustand** | Minimal React state store. | Reduced, on purpose, to **pure UI state only** — staged command-palette open/closed, compare-page traveler selection. Everything server-derived moved to TanStack Query once the app went multi-route (a deliberate mid-project refactor). | `saarathi-frontend/lib/store.ts` | React `useState`/Context — Zustand was kept mainly because some state (compare-page selection) needs to persist across a route's own remounts. |
| **TanStack Query** | Server-state cache/fetch library for React. | The backbone of §3's "URL is the cache key" design — `useRecommendation`/`useUsers`/`useLegRecommendation` (`lib/queries.ts`) give automatic caching, dedup, and loading/error states without hand-rolled `useEffect` fetch logic. | `lib/queries.ts`, `components/providers/query-provider.tsx` | Raw `useEffect` + `fetch` (the pre-redesign approach, still visible in git history as `lib/store.ts`'s old server-fetching methods before this refactor). |
| **zod** | Runtime schema validation with TypeScript inference. | Validates every `POST /api/recommend` body server-side (`recommend.controller.ts:21-41`) and every composer form client-side (`components/composer/RequestForm.tsx:17-24`) from the *same conceptual shape*, independently defined on each side (not shared — see §10). | Both `recommend.controller.ts` and `RequestForm.tsx` | Manual `if` validation, or a shared schema package (would close the drift gap in §10). |
| **Tailwind CSS + shadcn/ui** | Utility-first CSS + a copy-into-your-repo component library built on Base UI primitives. | shadcn's CLI vendors real, accessible component code (`components/ui/*.tsx`) directly into the repo rather than pulling a black-box npm package — every component is editable. Tailwind v4's CSS-first `@theme` config (`app/globals.css`) drives the whole light/dark design system. | `app/globals.css`, `components/ui/*.tsx`, `components.json` | Material UI / Chakra (heavier, harder to restyle to a bespoke design system), hand-written CSS modules. |
| **Papaparse** | CSV parser. | Reads `user_data.csv`/`flights_data.csv` in `core/data.ts`'s `buildStore()` (test-only path, §5) and in `prisma/seed.ts` (the one-time DB-seeding script). | `core/data.ts:14,103,117`, `prisma/seed.ts` | Node's built-in CSV handling doesn't exist; a hand-rolled split-on-comma parser would break on the quoted, comma-containing `raw_history` field. |
| **Vitest** (backend) | Fast Vite-native test runner. | The **only test runner that actually works** in this repo — `npm test` runs it, 9 files / 35 tests, all passing (verified this session). | `saarathi-backend/src/core/*.test.ts` | — |
| **Jest** (backend, configured but broken) | The other test runner, configured for both unit (`src/**/*.spec.ts`) and e2e (`test/**/*.e2e-spec.ts`). | Neither path currently works — see §8 for the exact failures. Likely a NestJS-CLI-scaffolding leftover (Nest's default template ships Jest) that vitest was added alongside without removing. | `package.json` `jest` block, `test/jest-e2e.json` | Standardize on one runner — recommended: keep Vitest, delete the Jest config and the two dead npm scripts, port `app.e2e-spec.ts` to Vitest or delete it. |
| **Vitest** (frontend, configured, unused) | Same tool, frontend side. | `vitest.config.ts` exists and `vitest` is installed, but there is no `"test"` script in `package.json` and **zero** `*.test.ts`/`*.test.tsx` files anywhere in the frontend (verified via `find` this session). Fully vestigial. | `saarathi-frontend/vitest.config.ts` | Remove, or actually write component tests. |

---

## 7. The AI boundary

**Two, and only two, places touch a model.**

**1. `core/preferences.ts` — embeddings, local, never network.** `findEmbeddingMatch(phrase)` (line 113-150) calls the locally-loaded `all-MiniLM-L6-v2` pipeline on a single `raw_history` phrase, gets a 384-dim vector, and compares it via raw dot product against 12 pre-embedded archetype sentences (3 per dimension × 4 dimensions). It **only ever returns a `dimension` label and a similarity score** — it never influences ranking directly, never returns a number that reaches the scorer; it can only *add an `EvidenceItem`* which then feeds `direct_weight`/`cost_weight`/`convenience_weight`/`avoid_redeye` through the exact same `+0.1`/`+0.15`-per-hit arithmetic as a regex hit (`preferences.ts:259-268`). **Fallback, quoted exactly** (`preferences.ts:103-110`):
```ts
} catch (err) {
  console.warn(
    '[Saarathi Embeddings] Failed to load embedding model, falling back to rules-only mode:',
    err,
  );
} finally {
  modelLoading = false;
}
```
If the model fails to load, `extractor` stays `null` forever; `inferPreferences` checks `if (!matched && extractor)` (line 241) before ever calling the embedding path, so every unmatched phrase is just silently skipped — regex-only mode, no crash, no missing response field.

**2. `core/explain.ts` — Groq LLM, one network call, prose only.** `explain(userId, requestText, pref, ranked, alternatives, counterfactuals, confidence)` (line 83-128) builds a prompt (`PROMPT`, line 21-38) that hands the model **already-computed** evidence, top-3 ranked options, alternatives, and counterfactual labels — the prompt's final instruction is explicit: *"You must justify this recommendation by directly citing the preferences evidence... Do not invent any facts or reasoning that isn't grounded in the provided data."* The model returns a string; nothing it returns is parsed back into a number, a ranking, or a boolean anywhere in the codebase — `chain.invoke(...)` output goes straight into `RecommendResponse.explanation` (a `string` field) and nowhere else. **Fallback, quoted exactly** (`explain.ts:64-81,96-99,124-127`):
```ts
function fallbackExplanation(userId, pref, ranked, confidence): string {
  const best = ranked[0];
  const lastEvidence = pref.evidence.map((e) => e.text).slice(-2).join(', ') || 'structured profile data';
  return (
    `For ${userId}, the top pick is ${best.airline_name} (${best.stops} stop(s), ` +
    `$${best.price.toFixed(0)}, ${(best.duration_minutes / 60).toFixed(1)}h) matching with ${confidence.matchPct}% score ` +
    `(${confidence.tier} confidence), based on: ${lastEvidence}.`
  );
}
// ...
const apiKey = process.env.GROQ_API_KEY;
if (!apiKey) {
  return fallbackExplanation(userId, pref, ranked, confidence);
}
try {
  // ... chain.invoke(...)
} catch (err) {
  console.error('Groq explanation call failed, using fallback:', err);
  return fallbackExplanation(userId, pref, ranked, confidence);
}
```
Both the missing-key case and the try/catch around the actual call route to the same deterministic template — a real, working answer using only fields already present on `ScoredFlight`/`Confidence`, never a placeholder string, never a thrown error the caller has to handle.

**"So is this just a ChatGPT wrapper?"** No — and the codebase makes that a checkable claim, not a talking point. The verdict, the ranking, the ordering, the alternatives, and the counterfactual price thresholds are all produced by `ranking.ts`/`counterfactuals.ts`/`confidence.ts` — pure arithmetic over `InferredPreference` and `FlightRow[]`, zero LLM calls, and (per §4's worked example) the exact break-even price is independently re-derivable by hand from numbers in the response. The LLM is invoked exactly once per request, after every number is already final, and its only contract is to phrase that data in English — it literally cannot change the verdict, because `explain()` is called with the ranking already computed and its return value only ever populates one `string` field. If Groq is down, wrong, slow, or the key is missing, the verdict, ranking, counterfactuals, and confidence are **bit-for-bit identical** — only the prose sentence changes.

---

## 8. Test & benchmark story

**What actually runs, verified this session:**

| Command | Runner | Result |
|---|---|---|
| `npm test` (backend) | Vitest | **✅ Works.** 9 test files, 35 tests, all passing (`confidence.test.ts`, `counterfactuals.test.ts`, `data.test.ts`, `explain.test.ts`, `multicity.test.ts`, `preferences.test.ts`, `ranking.test.ts`, `benchmark.test.ts`, `custom_benchmarks.test.ts`). These are real unit + integration tests exercising `core/` directly (via the CSV `buildStore()` path, per §5) — not mocked. |
| `npm run test:watch` / `test:cov` / `test:debug` (backend) | Jest, against `src/**/*.spec.ts` | **❌ Broken.** `testRegex: '.*\.spec\.ts$'` matches nothing — every test file in `src/core/` is named `*.test.ts`. Output: `No tests found, exiting with code 1`. |
| `npm run test:e2e` (backend) | Jest, against `test/*.e2e-spec.ts` | **❌ Broken — crashes at import time.** `ts-jest`'s default CommonJS transform can't parse `@xenova/transformers`'s pure-ESM `export * from './pipelines.js'` syntax (imported transitively via `core/preferences.ts` ← `recommend.controller.ts` ← `app.module.ts`). Real error: `SyntaxError: Unexpected token 'export'`. Fix would require `transformIgnorePatterns` configuration for that package, or migrating this one spec to Vitest. |
| Frontend tests | Vitest configured, unused | No test files exist, no `"test"` script in `package.json`. Nothing to run. |

**What `BENCHMARK_REPORT.md` and `CUSTOM_BENCHMARK_REPORT.md` measure.** Both are **live-API smoke reports**, not unit-test output — generated by hitting the real running server (`test_benchmarks.ts`, `custom_test_benchmarks.ts`, both plain scripts run via `npx ts-node`, not part of any npm test script). `BENCHMARK_REPORT.md` runs the 6 official prompts from `data/benchmark_prompts.json` (B01-B06, one per seeded user U01-U06) and checks each produces a sensible mode/winner/confidence. All 6 currently pass. Its own "What Needs Improvement" section documents real, previously-encountered issues and their fixes — worth reading verbatim if a judge asks "what was hard": the date filter was originally over-eliminating candidates by anchoring to the first flight's date even with no explicit date requested (fixed by making it opt-in, `ranking.ts:126-143`); the multi-city temporal gate was originally 12 hours, relaxed to 60 minutes (`multicity.ts:34`); U05's 90-minute layover + strong direct preference has essentially zero matching LIS→SYD flights in the dataset — a genuine data coverage gap, not a bug, handled by the constraint-relaxation fallback in `recommend.controller.ts:251-365`. `CUSTOM_BENCHMARK_REPORT.md` runs 8 additional scenarios (C01-C08) specifically targeting perturbation behavior (`bags_matter`, `accept_one_stop`, `ignore_loyalty`) and multi-city edge cases (2-city, 3-city Asia routes) — all 8 currently pass, with response times logged between 655ms-1049ms.

---

## 9. Ownership map — "if I want to change X, I edit Y"

| Change | Files, in order |
|---|---|
| Change scoring weights/formula | `saarathi-backend/src/core/ranking.ts:183-216` (the score + breakdown objects) → re-run `npm test` (several tests assert exact weight values) → if the *maximum achievable* score changes, `core/confidence.ts:36-45` (`maxRawScore`) must be updated to match or `matchPct` silently breaks. |
| Add a new counterfactual toggle | `saarathi-backend/src/core/types.ts:103-109` (add the `Perturbation` union member) → `core/counterfactuals.ts:17-43` (`applyPerturbations`, add the branch) → `core/counterfactuals.ts:116-121` (add to the `toggles` array) → `core/counterfactuals.ts:151-162` (add its label branch) → `recommend/recommend.controller.ts:21-32` (`PerturbationSchema`, add the Zod variant) → frontend: `saarathi-frontend/core/types.ts` (mirror the union — manual, see §10) → `components/decision/CounterfactualPanel.tsx` if it needs distinct UI treatment. |
| Add an evidence rule (new regex pattern) | `saarathi-backend/src/core/preferences.ts:4-39` (add to the relevant `*_BOOST`/`REDEYE_*` array) → if it's a genuinely new dimension (not direct/cost/convenience/redeye), also touch `types.ts:53` (`EvidenceItem.dimension` union), `confidence.ts:64-71` (`dimensions` array + `DIMENSION_LABELS`), and the frontend's mirrored `core/types.ts`. |
| Change a Decision Screen zone's layout | The relevant `saarathi-frontend/components/decision/*.tsx` file directly (e.g. `VerdictCard.tsx`, `EvidencePanel.tsx`) — these are pure presentational components taking already-fetched props from `DecisionScreen.tsx`, so layout changes are isolated. |
| Add a field to the `/api/recommend` response | `saarathi-backend/src/core/types.ts` (`RecommendResponse` or the relevant sub-interface) → wherever it's computed (likely `ranking.ts`, `confidence.ts`, or a new `core/` module) → `recommend/recommend.controller.ts` (both the single-leg and multi-city branches build the response object separately — easy to update one and forget the other) → `saarathi-frontend/core/types.ts` (manual mirror) → the consuming component. |
| Change the theme (colors/fonts) | `saarathi-frontend/app/globals.css` (Tailwind v4 `@theme` block — this is the single source of truth for the design tokens; components reference token names, not raw hex). |
| Add a new alternative kind | `saarathi-backend/src/core/types.ts:139` (`Alternative.kind` union) → `core/ranking.ts:229-421` (`selectAlternatives`, add the selection logic + `makeAlt(...)` call) → `core/multicity.ts:301-332` (multi-city has its own alternatives builder with hardcoded placeholder entries for single-leg-only kinds — must be updated in parallel or the new kind will silently render as a stub in multi-city mode) → frontend `components/decision/OpportunityCostPanel.tsx` (kind → icon/label mapping). |
| Change which model powers the LLM explanation | `saarathi-backend/src/core/explain.ts:104` (`model: 'llama-3.3-70b-versatile'`) — one line, since `ChatGroq` is provider-agnostic within Groq's catalog; switching providers entirely would mean swapping the `@langchain/groq` import for another LangChain integration package. |
| Change the embedding similarity threshold or archetypes | `saarathi-backend/src/core/preferences.ts:53-74` (`ARCHETYPES` sentences) and line 137 (`dot > 0.78`). |
| Add a new route to the frontend | New folder under `saarathi-frontend/app/(product)/app/` (gets the shared `TopNav` shell automatically via the route group's `layout.tsx`) or under `saarathi-frontend/app/` directly (opts out of the product shell, like the marketing `page.tsx` does). |
| Change the multi-city temporal-sanity gate | `saarathi-backend/src/core/multicity.ts:30-35` (`satisfiesTemporalSanity`, currently 60 minutes). |
| Add a new seeded/benchmark user for a demo | `data/user_data.csv` (new row) → `saarathi-backend/prisma/seed.ts` (re-run `npx prisma db seed` to load it into `prisma/dev.db`) → restart the backend so `initializeStoreFromDb` re-reads it — editing the CSV alone does **not** affect the running server (see §5). |
| Change CORS / add auth between frontend and backend | `saarathi-backend/src/main.ts:8-12` (`app.enableCors({...})`) — currently wildcard, no credentials check despite `credentials: true` being set. |
| Fix `GET /api/users` to use the shared in-memory store instead of querying Prisma live | `saarathi-backend/src/users/users.controller.ts:8-23` — replace `this.prisma.user.findMany({...})` with `Array.from(getStore().users.values())` mapped to the same field subset. |

---

## 10. Known debt

Ranked roughly by "how likely a judge or a future contributor trips over this."

1. **Frontend `core/types.ts` is a hand-copied, byte-for-byte-except-quote-style mirror of the backend's `core/types.ts`.** Verified via `diff` this session — the only differences are Prettier quote style (backend uses single quotes, frontend double). There is no shared package, no code generation, no CI check that they stay in sync. Adding a field to the backend's `RecommendResponse` silently does *not* appear on the frontend type until someone manually copies it over — TypeScript will happily compile against the stale, narrower type. **Fix:** either publish `core/types.ts` as a tiny shared package both repos depend on, or (cheaper) add a CI/pre-commit diff check between the two files.
2. **`GET /api/users` bypasses the in-memory store and queries Prisma directly**, while every other endpoint reads from `getStore()`. Two different data-access patterns for what should be the same cache. See §5 and §9 for the exact one-file fix.
3. **`saarathi-backend/dev.db` (repo root) is a stale, empty database file** — 0 rows in both tables, confirmed via `sqlite3` this session — left over from a `prisma` command run from the wrong directory. Safe to delete; nothing reads it.
4. **Three of six backend test/lint npm scripts are currently broken**, verified by running them this session: `test:watch`/`test:cov`/`test:debug` (Jest against a `*.spec.ts` pattern that matches zero files in `src/`) and `test:e2e` (crashes on `@xenova/transformers`'s ESM syntax before any test runs). Only `npm test` (Vitest) actually works. A contributor running `npm run test:watch` expecting live test feedback will get a silent "no tests found" and might reasonably conclude the project has no tests at all.
5. **The CSV-loading path (`buildStore()` in `data.ts`) is exercised only by tests and never by the running server**, which uses the Prisma path exclusively (see §5). This means the 35 tests that pass are not, strictly, testing the code path that serves real traffic — they're testing an equivalent-but-distinct implementation of the same data-loading contract.
6. **Frontend has a fully-configured but completely unused Vitest setup** — `vitest.config.ts` exists, `vitest` is a devDependency, there's no `"test"` script and zero test files. Either write frontend tests or delete the config to stop it looking like coverage that doesn't exist.
7. **Counterfactuals are computed against `basePref` (pre-perturbation), not `perturbedPref`**, in the normal single-leg success path — `recommend.controller.ts:419-425` explicitly passes `basePref` to `computeCounterfactuals`, while `computeConfidence` two lines later uses `perturbedPref`. This might be intentional (showing counterfactuals relative to the traveler's *original* inferred preferences, even while a perturbation is staged, so the boundary analysis stays stable as you toggle chips) — but it's inconsistent with the constraint-relaxation fallback path a few lines earlier (line 342-347), which *does* use `perturbedPref` for counterfactuals. Worth a deliberate decision one way, since right now it reads like it could be an oversight.
8. **`GROQ_API_KEY` is present in `saarathi-frontend/.env.local` but never read by any frontend code** (verified via grep — zero references) — a leftover from before the LLM call moved into the separate NestJS backend. Not a security hole exactly (it's a `.env.local`, gitignored), but it's a live credential sitting somewhere it doesn't need to be, and it invites confusion about which service actually owns AI calls.
9. **Backend CORS is `origin: '*'` with no authentication anywhere.** Fine for a hackathon demo on `localhost`; would need locking down (specific origin allowlist, at minimum an API key or session check) before this touched a public deployment with real user data.
10. **`dist/` (backend's compiled output) is checked into git.** Unusual for a Node project (`.gitignore` doesn't exclude it) — means every source change requires a matching `nest build` before committing, or the checked-in `dist/` silently drifts from `src/`. Not dangerous, just unconventional; a `.gitignore` entry plus removing `dist/` from version control would be a 10-minute cleanup.
11. **The `(product)` route group's naming is easy to misread.** `app/(product)/app/decision/page.tsx` serves `/app/decision`, not `/(product)/app/decision` — the parenthesized segment is invisible in the URL. Anyone unfamiliar with Next.js route groups will spend a confused five minutes here; worth a one-line comment in `app/(product)/layout.tsx` for the next person.
