# Judge Prep — Viva Sheet

Answers written in first person, grounded in the actual code and real numbers pulled from this session's live runs. Read `docs/HANDOFF.md` for the file/line citations behind each answer — this sheet is the compressed, spoken version.

---

### 1. Walk me through what happens when you click search.

I fill in a traveler and a destination in the composer form, which is just a `react-hook-form` + `zod`-validated form — no network call yet. On submit it builds a URL like `/app/decision?userId=U01&requestText=...&destination=NRT` and navigates there. The Decision Screen reads that URL, and a TanStack Query hook uses those exact params as its cache key to fire `POST /api/recommend` at my NestJS backend. The backend validates the body with Zod, pulls the traveler from an in-memory store, infers their preferences from regex plus a local embedding model, filters and scores the candidate flights with one deterministic formula, computes alternatives and counterfactuals, asks Groq for a 3-4 sentence explanation of the already-decided verdict, and returns one JSON object that the whole screen renders from. I actually fired this exact request during prep — it took 1.06 seconds end to end, and I can show you the real payload.

### 2. Why two services instead of one?

The original plan was one Next.js app with an API route — I still have that plan on record and it's honestly still valid for a smaller scope. I split it into a NestJS backend because I wanted a real database layer (Prisma over SQLite) and a proper module/DI structure once the scoring logic and the frontend both started growing, and it made the "core logic has zero framework imports" boundary enforceable by a directory split, not just a lint rule. The tradeoff is real: two processes to boot, CORS to configure, and a type contract I currently keep in sync by hand between the two repos — I'm not going to pretend that's free.

### 3. Why NestJS specifically?

I wanted decorators and dependency injection for the controller layer without writing that plumbing myself, and Nest's module system made it trivial to keep `RecommendController` and `UsersController` as thin wrappers around plain-TypeScript `core/` functions — the controllers do validation and response shaping, nothing else. It's also just a well-trodden path for a Node backend with Prisma, so there wasn't much ceremony to set up.

### 4. Where exactly does AI come in — is this just an LLM wrapper?

No, and I can point to exactly where the boundary is. Two touchpoints only: a local embedding model (`all-MiniLM-L6-v2`, runs inside the Node process, no network) that helps classify free-text history into one of four preference dimensions, and one Groq call that turns an already-fully-computed verdict into English prose. The ranking, the scoring, the alternatives, and the counterfactual price thresholds are all plain arithmetic in `core/ranking.ts` and `core/counterfactuals.ts` — zero LLM calls. I can turn the Groq key off entirely and the verdict, the ranking, and the counterfactual numbers don't change at all — only the explanation sentence swaps to a deterministic template. The LLM is called *after* the decision is made, and it can't feed anything back into the decision.

### 5. How do you compute "wins if price drops below $543" — prove it's not hallucinated.

The score formula has exactly one term that depends on price, and it's linear in price after normalization, so I can invert it algebraically instead of guessing. I take the score gap between the champion and a challenger, multiply by the price range across the candidate set, and divide by the traveler's cost weight — that gives me the exact price drop needed, and I subtract it from the challenger's real price. I ran this for real during prep: American Airlines was winning at $1571.95 with score 1.897, Korean Air was runner-up at $5093.16 with score 1.892. The math gives a break-even of $4816.22 — and that's the literal number the API returned, not something I reverse-engineered afterward. Anyone with a calculator can redo that arithmetic from the response JSON.

### 6. Why can you invert your ranking when others can't?

Because I chose a scoring function that's a weighted linear sum, not a black-box model. Most "smart" recommenders either use opaque ML rankers or ad-hoc heuristics with branching logic, and neither is algebraically invertible — you'd have to brute-force search for the break-even point. Mine is `cost_weight × priceScore + direct_weight × directScore + ...`, all linear in the flight's normalized attributes, so solving for "what price makes challenger.score equal champion.score" is one line of algebra. That constraint — keep the formula linear — was a deliberate design choice specifically so counterfactuals could be closed-form instead of simulated.

### 7. How does it handle 50,000 flights?

The whole dataset — 50 users, 50,000 flights, indexed into 1,172 unique origin-destination routes — loads once at server boot into in-memory JavaScript `Map`s, keyed by origin and by `"origin-destination"` route string. That took 548ms on the run I just watched boot. Every actual request only ever touches the flights on one route — for CPT→NRT that's 57 flights, not 50,000 — so filtering and scoring a request is sub-millisecond; the ~1 second of total latency I measured is almost entirely the Groq network call at the end, not the ranking math.

### 8. What's deterministic vs probabilistic in this system?

Everything except two things is deterministic: given the same user and the same flight data, the ranking, the verdict, the confidence score, and every counterfactual price threshold will be bit-for-bit identical on every run — no randomness, no model calls. The two non-deterministic pieces are the embedding-similarity fallback in preference inference (though even that only adds evidence entries, never touches the score formula directly) and the Groq explanation text, which can phrase things differently between calls even for identical input. I can demonstrate the deterministic half live by calling the API twice with the same body and diffing the verdict/score/counterfactual fields — they won't move.

### 9. How do you handle the LLM being down?

There's a try/catch around the entire Groq call in `explain()`, and a separate check upfront for whether the API key is even set. Both paths — missing key, or the call throwing — fall through to the same deterministic template function that builds a real sentence out of fields already on the response object: the winning airline, price, duration, match percentage, and the last two pieces of evidence. It's never a blank string, never a thrown 500, and the rest of the response — verdict, ranking, counterfactuals — is completely unaffected either way, since `explain()` runs last and only populates one string field.

### 10. What would you build next?

Two things, in order. First, I'd close the type-drift gap between the frontend and backend — right now `core/types.ts` is manually duplicated across both repos, verified identical except for quote style, with nothing stopping them from silently diverging. Second, I'd add the day-of-week and connection-anxiety personalization the benchmark report already flagged as a real opportunity — the dataset has an `on_time_performance` column that's currently unused, and at least one seeded user's history explicitly signals fear of tight connections; that's a natural scoring penalty I haven't wired up yet.

### 11. What are the limitations?

The counterfactual price thresholds are only computed for the top 3 challengers, not the whole ranked list, so a flight ranked 15th never gets its own break-even shown even if it's theoretically achievable. The multi-city router is a brute-force permutation search — fine for the 2-4 city trips this dataset supports, but it wouldn't scale to a 10-city tour without real pruning work beyond what's there now. And the preference inference is calibrated against four dimensions and a fixed set of regex patterns and archetype sentences — it's not going to generalize to phrasing wildly outside what I tuned it against, and I know that because I can point to exactly one real phrase in the seed data ("aisle seat, front of cabin") that the system currently drops on the floor because it doesn't map to any tracked dimension.

### 12. How did you use the benchmark prompts?

The six official prompts in `data/benchmark_prompts.json` map one-to-one to the six seeded users and cover single-leg and multi-city modes, high and low price sensitivity, strong and absent direct-flight preference. I ran all six against the live API and logged mode, winner, price, and confidence for each — all six currently pass, and the report also documents three real bugs I found and fixed while doing that: a date filter that was wrongly eliminating almost everything when no date was specified, a multi-city turnaround gate that was too strict at 12 hours, and a confidence formula whose denominator wasn't tracking the actual preference weights. I also wrote eight more scenarios myself specifically to stress perturbations and multi-city edge cases, and those pass too.

### 13. What assumptions did you make about the data?

That `raw_history` is the richest, least-structured signal and deserves the most parsing effort — hence three layers (structured fields, regex, embeddings) just for that one column. That airport codes in the dataset are internally consistent (I verified this — exactly 35 unique origin airports, and I hand-built a static lat/lon table for precisely those 35 for the map, rather than depending on a live geocoding API). And that a route with zero flights, or a traveler whose hard constraints eliminate every candidate, is a legitimate outcome to show honestly rather than something to paper over — that's why there's a whole constraint-relaxation fallback path instead of just returning an empty result.

### 14. Why does your app have two databases — or does it?

It has one real one. There's a Prisma-managed SQLite file with 50 users and 50,000 flights that I verified by querying it directly — that's the one the running server actually reads at boot. There's also a stale, empty SQLite file sitting at the repo root from an earlier `prisma` command run from the wrong folder — zero rows, nothing reads it, and I know that because I checked. I'm flagging that myself rather than waiting for someone to find it.

### 15. Does the backend read the CSVs directly, or the database?

Both code paths exist, but only one runs in production. At boot, `AppModule` seeds an in-memory cache from Prisma/SQLite, and every real request reads from that cache. There's a second function that builds the same cache straight from the CSVs with Papaparse — but it only fires if the Prisma path hasn't already populated the cache, which never happens on the real server. That CSV path is actually what every one of my 35 unit tests exercises, since tests call the store function directly without booting the full Nest app. I know that's a real inconsistency and it's in my own known-debt list, not something I'm hiding.

### 16. Why Prisma at all if the data barely changes?

Honestly, the original plan explicitly argued against a database for this exact reason — "zero judge-visible value" is a direct quote from my own planning doc. I added it anyway, mainly to get typed queries and a clean seed script instead of hand-rolling CSV parsing at every boot, and because it made the traveler-listing endpoint trivial to write. In hindsight the CSV-only plan was probably right for the given scope, and if I had to defend one line item for removal under time pressure, this dual-path setup is it — though ripping it out now would mean rewriting a working, tested boot sequence.

### 17. How do counterfactual toggles actually work in the UI?

Tapping a chip doesn't call a special "toggle" endpoint — it rewrites the page's URL to include that perturbation in a `pts` query parameter, and because my data-fetching cache key is built directly from the URL, that URL change alone triggers a fresh POST to the recommend endpoint with the perturbation included. I can reload that exact URL later and get the identical state back, because the URL *is* the state — that was a deliberate design choice so every state the app can be in is shareable as a link.

### 18. What's the confidence score actually measuring?

Two independent things folded into one badge. `matchPct` is the champion's actual score divided by the theoretical best score achievable under that exact traveler's weights — for my example traveler that came out to 88%, computed from real numbers I can show. The tier — high, medium, or low — is separately about the *margin* between the top two flights and whether the evidence behind the top pick came from more than one independent source. A flight can have a high match percentage but still get a "low" tier if the runner-up is nearly tied, or if all the evidence for that pick came from just one weak signal.

### 19. What happens if a traveler's constraints eliminate every flight?

It doesn't just return an empty list. The controller finds whichever constraint step removed the most flights, tries relaxing exactly that one — first loosening the layover limit by 1.5x, and if that's still empty, additionally allowing redeye departures — and if a relaxed search finds real flights, it shows that verdict with a visible note about what was relaxed. Only if nothing works even after relaxation does it fall back to an honest "no flights matched" message naming the specific constraint that killed everything. I hit this for real with one of my six benchmark users — a 90-minute layover limit combined with a strong direct-flight preference on a route where the dataset's shortest layover is 105 minutes.

### 20. How is the multi-city routing solved?

It generates every permutation of the requested cities, and for each one does a pruned search over each leg's ranked flight candidates, rejecting any leg pairing where the next flight departs less than an hour after the previous one lands. It's brute-force by design — with 2-4 cities that's at most 24 permutations, trivially fast — rather than a heuristic that might miss the actual optimum. It compares the winning itinerary's total score against the runner-up itinerary and builds one ordering-level counterfactual using the same closed-form price algebra as the single-leg case.

### 21. What's actually tested, and what isn't?

Thirty-five real tests across nine files, covering every core module — ranking, confidence, counterfactuals, preferences, multi-city, and two dedicated benchmark suites — and they all pass; I re-ran them during this prep session. What's *not* tested well: I don't have coverage that exercises the exact Prisma-backed boot path the production server actually uses, since my unit tests go through the CSV-loading function instead, and there's a Jest e2e test that currently can't even run because of a module-format conflict with one of my dependencies. I'd rather tell you that than have you find it live.

### 22. What's the single riskiest line of code in this system?

The line in the recommend controller that decides whether counterfactuals get computed against the traveler's original inferred preferences or their currently-perturbed ones — right now it uses the original ones in the normal path, which may be intentional (so the decision boundary stays stable while you're experimenting with chips), but it's inconsistent with how the constraint-relaxation fallback path does it a few lines away. It's not a crash risk, it's a "does this number mean what I claim it means" risk, and I'd rather resolve that deliberately than have a judge find the inconsistency by testing edge cases.

### 23. Why did you choose a light theme when your own spec called for a dark "Bloomberg Terminal" look?

That was a deliberate, explicit redesign decision, not drift — I was asked to rebuild the whole frontend later in the project toward a different visual reference, and the density language from the original spec (tight numeric rows, mono type for prices) survived into the Decision Screen specifically, just without the dark background driving it. I can show both the old and new token definitions; the dark theme still exists as an opt-in toggle, it's just not the default anymore.

### 24. How do you know your scoring formula isn't just marketing math that happens to sound good?

Because every constant in it is used identically in three separate places that would visibly disagree if it were fudged: the actual ranking formula, the "maximum achievable score" denominator that produces the match percentage, and the counterfactual break-even algebra. If I changed one weight without updating the other two, my own tests would start failing — several of them assert exact score and confidence values for known inputs. That cross-checking is incidental to testing, not something I built specifically to prove a point, which is why I trust it.

### 25. What happens on a cold start — walk me through boot.

The backend connects to SQLite via Prisma, pulls all 50 users and 50,000 flights into memory — I watched this take 548ms just now — then starts loading the local embedding model in the background, which takes a few seconds more and logs its own success line. The HTTP server is actually listening and answering `/api/users` before the embedding model finishes loading; the very first `/api/recommend` call after boot will `await` that model load inline if it hasn't finished yet; every call after that is instant. The frontend is a completely separate `next dev` process on port 3000 with no dependency on the backend's boot sequence beyond needing it up before the first API call succeeds.

### 26. What's your actual API surface — how many endpoints?

Two. `GET /api/users` returns the 50 seeded travelers with a subset of fields the UI needs for pickers and profile cards. `POST /api/recommend` does everything else — single-leg and multi-city, base request and every counterfactual perturbation, all through one endpoint with one Zod-validated body shape. I didn't split counterfactual toggling into its own endpoint on purpose — it's the same computation with different inputs, so it's the same code path.

### 27. Why does the frontend have 8 routes when the flight-decision logic only needs one screen?

Because the Decision Screen alone doesn't tell the whole product story — there's no way to browse the 50 seeded travelers, compare two of them side by side, or explain the pipeline to someone unfamiliar with it, without dedicated screens for each. The Decision Screen is still the flagship and the most heavily engineered route; the other seven exist to make the traveler data and the reasoning process explorable rather than requiring you to already know a `userId` to type into a form.

### 28. If I gave you another week, what's the highest-leverage single change?

Turning `core/types.ts` into an actual shared package instead of a hand-copied file in two repos. Everything else I'd fix is either cosmetic or a small, contained bug; that one is the change most likely to cause a real, silent production bug later — someone adds a field on the backend, forgets the frontend copy, and TypeScript won't catch it because the stale type still compiles fine.

---

## Flashcard — numbers I must know cold

| Fact | Value |
|---|---|
| Users seeded | **50** |
| Flights seeded | **50,000** |
| Unique origin-destination routes | **1,172** |
| Unique origin airports | **35** |
| Unique airlines | **18** |
| Cabin classes | **4** (Economy, Premium Economy, Business, First) |
| Backend port | **4000** (hardcoded, `main.ts`) |
| Frontend port | **3000** (Next.js default, auto-increments if busy) |
| Boot time (seed → ready) | **~548ms** for the in-memory store; embedding model finishes a few seconds after HTTP starts listening |
| Backend test files / tests | **9 files / 35 tests**, all passing (Vitest — the only working test runner) |
| Backend LLM model | **`llama-3.3-70b-versatile`** via Groq, through LangChain |
| Backend embedding model | **`Xenova/all-MiniLM-L6-v2`**, runs locally, no network call |
| Real end-to-end request latency | **~1.06s** (single-leg, includes the Groq call) |
| Benchmark prompts | **6 official (B01-B06)**, all passing; **8 custom (C01-C08)**, all passing |
| Score formula | `score = (cost_w·priceScore + direct_w·directScore + (1−cost_w)·0.5·timeScore + conv_w·0.3·cabinScore + 0.2·airlineScore + baggage_w·baggageScore) × demandAdj × holidayAdj + dayBonus` |
| Break-even (counterfactual) formula | `priceDrop = (championScore − challengerScore) × priceRange / (cost_weight × demandAdj × holidayAdj)` |
| Worked break-even example | Champion AA351 @ $1571.95 (score 1.897) vs. Korean Air KE3237 @ $5093.16 (score 1.892) → break-even **$4816.22** |
| matchPct formula | `round(championScore / maxAchievableScore × 100)`, where `maxAchievableScore` = every sub-score at 1, best-case demand (1.05×) |
| Confidence tier thresholds | margin `> 0.1` → high; `< 0.04` → low; else medium — demoted on signal conflict/absence, promoted to medium if matchPct ≥ 80% |
| Trace stages per response | **7**: request → preferences → constraints → candidates → tradeoffs → counterfactuals → verdict |
| CORS policy | Wildcard (`origin: '*'`), no auth |
