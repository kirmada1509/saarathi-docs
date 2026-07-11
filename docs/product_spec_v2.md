# Threshold — Product Specification v2

> **The AI travel strategist that tells you exactly what would change its mind.**

One line for judges: every flight site shows you options. Threshold shows you the decision — what to book, what it costs you, and the precise conditions under which a different choice becomes right.

---

## 0. What changed from v1, and why

This is a replacement, not an edit. The v1 spec was an engineering document wearing a product costume. Here is the disposition of every major piece:

| v1 element | Verdict | Reason |
|---|---|---|
| Recommendation dashboard concept | **Replaced** | "It recommends flights" is what every team will say. Not memorable. |
| Preference inference (rules + embeddings) | **Kept, demoted to plumbing** | It works. It feeds the Decision Screen. It is no longer the pitch. |
| Deterministic ranking (`ranking.ts`) | **Kept, promoted** | Its linear scoring function is what makes honest counterfactuals *computable*. This is now the technical centerpiece — see §4. |
| Multi-city graph optimizer as "primary showpiece" | **Demoted to supporting act** | It's a rubric line, so it stays. But nobody remembers a permutation solver. It gets one demo beat, not the whole show. |
| Trade-off summary (direct vs 1-stop string) | **Replaced** | One hardcoded comparison → a full opportunity-cost panel (§5). |
| LLM explanation (`explain.ts`) | **Kept, narrowed** | LLM phrases decisions. It never makes them. Same fallback discipline. |
| NestJS + Prisma + SQLite backend | **Challenged — cut recommended** | See §8. You told me to keep it. I'm overruling that with a reason: your deadline is ~36 hours away, your `package.json` is already Next-only, and a second deployable buys zero judge-visible value. Core modules stay framework-independent so this remains reversible. |
| Reasoning shown as processing animation | **Respec'd** | Your pipeline runs in milliseconds. A staged animation of instant work reads as fake to a technical judge. It becomes an *inspectable trace* — real payloads behind every stage (§7). |
| "94% Match / Confidence: High" | **Kept, with teeth** | Displayed confidence must be derived, not decorated. Formulas in §6. If a judge asks "where does 94% come from?" and the answer is "vibes," the entire explainability story collapses. |
| Recommendation audit-trail DB table | **Cut** | Nice for production. Invisible in a 5-minute demo. Time goes to the screen instead. |
| Season/demand multiplier, evidence trail, CSV data model | **Kept unchanged** | Working code. Rubric coverage. Don't touch. |

---

## 1. The problem

Booking a flight is not a search problem. It's a decision under competing constraints — cost, time, fatigue, flexibility, loyalty, arrival windows — and every existing tool responds by showing *more information*. Kayak gives you 200 rows and a sort dropdown. The user still does all the deciding alone.

The unmet need: **someone who takes a position, defends it with evidence, and is honest about when they'd be wrong.**

That's what a good human travel agent does. That's what Threshold does.

## 2. The product in one screen

Threshold is a single screen — the **Decision Screen** — that a judge understands in ten seconds with no narration. Six zones:

```
┌─────────────────────────────────────────────────────────────┐
│  VERDICT            Delta 412 · Direct · $612 · lands 9:40a │
│                     91% match · Confidence: High             │
├──────────────────────────────┬──────────────────────────────┤
│  WHY THIS ONE                │  WHAT WOULD CHANGE MY MIND   │
│  ✓ Direct — you avoid        │  United 88 wins if:          │
│    connections (strong)      │  → its price drops below $543│
│  ✓ Morning arrival           │  → you accept one stop      │
│  ✓ Delta = your carrier      │  Air France 21 wins if:     │
│    on 7 of last 9 trips      │  → bags matter more than $80│
├──────────────────────────────┼──────────────────────────────┤
│  WHAT YOU'RE GIVING UP       │  HOW I DECIDED               │
│  Cheapest  save $120 / +5.1h │  request → preferences →     │
│  Fastest   −2h / +$260       │  constraints → candidates →  │
│  Flexible  refundable / +$80 │  trade-offs → counterfactuals│
│  Comfort   business / +$700  │  (every stage inspectable)   │
└──────────────────────────────┴──────────────────────────────┘
```

Everything else in the system exists to fill these six zones. Any work that doesn't land on this screen is cut or deferred.

Design language: closer to Bloomberg Terminal than to a chat bubble — dense, confident, numeric. But with ruthless hierarchy: the Verdict is readable from across the room; the trace is there for the judge who leans in.

## 3. Zone by zone

### 3.1 Verdict
One flight. Not a ranked list of ten. The ranked list still exists (collapsed, one tap away — judges checking rubric box "ranked recommendations" will find it), but the product takes a position. Taking a position is the point.

### 3.2 Why this one
Rendered directly from the existing `evidence[]` trail in `preferences.ts` — already built, already auditable. Each reason is tagged **strong** or **weak** per §6. The LLM rewrites evidence strings into plain English; it cannot add reasons that aren't in the trail (same grounding rule as v1's `explain.ts`).

### 3.3 What you're giving up (opportunity cost)
Five deterministic alternatives, each an `argmin/argmax` over the already-scored candidate set:

- **Cheapest** — min price. Show: money saved, hours lost.
- **Fastest** — min duration. Show: hours saved, money lost.
- **Flexible** — cheapest refundable. Show: what refundability costs.
- **Comfort** — cheapest cabin upgrade. Show: the premium.
- **Date shift** — best flight within ±`date_flexibility_days` of the requested date. Show: *"Leaving Tuesday instead of Monday saves $140."* This is the toolkit's "flexible date constraints" dimension made visible on the main screen, using a field (`date_flexibility_days`) the user data already carries. If the user's flexibility is 0 days, the row says so — respecting the constraint is itself evidence of constraint-based reasoning.

Each row shows both directions of the trade — what you gain *and* what you sacrifice. If a category is empty (no refundable flights on this route), say so explicitly. Never hide an empty state.

Cost to build: one afternoon. Impact: this is the panel non-technical judges will screenshot.

### 3.4 What would change my mind → **§4, the flagship**

### 3.5 Confidence → **§6**

### 3.6 How I decided → **§7**

## 4. Counterfactuals: the flagship, and why it's honest

Every recommender shows rank #2. Threshold shows the **decision boundary**: the exact conditions under which #2 becomes #1. No other team will have this, because most teams' ranking will be an opaque LLM call — and you can't invert an opaque LLM call.

You can invert yours. The v1 scoring function is a weighted linear sum. That "boring" deterministic choice is now the whole trick:

**Type 1 — Threshold counterfactuals (closed-form).**
Score is linear in `priceScore`, and `priceScore` is linear in price (holding the candidate set's min/max normalization bounds fixed — state this assumption in code comments; judges may probe it). So for challenger *c* and champion *w*:

```
find Δprice such that score_c(price_c − Δprice) = score_w
→ Δprice = (score_w − score_c) / (cost_weight × ∂priceScore/∂price_c)
```

Output: *"United 88 becomes my pick if its fare drops below $543."* A real number, from algebra, not from a language model. Compute it for the top 2–3 challengers. If the required drop is absurd (>60% of fare), say *"no realistic price makes this win"* — honesty about non-flippable options is itself memorable.

**Type 2 — Toggle counterfactuals (perturb and re-rank).**
Flip one preference or constraint, re-run `filterAndRank`, report only perturbations that change the winner:

- Accept one stop (relax direct preference)
- Bags matter (weight `baggage_included`)
- Evening departure OK (drop redeye/morning constraint)
- Loyalty doesn't matter (zero the airline bonus)
- Shift dates ±N days (widen the candidate window within `date_flexibility_days` — flexible travel windows as a *counterfactual*, not a separate feature)

Re-ranking a route's candidates is microseconds; brute force is correct here. Cap the display at 3–4 flips — the ones with the smallest perturbation. Below them, one quiet line: *"Nothing else within reason changes this decision."* That sentence is the confidence statement judges remember.

**Make it live.** Each counterfactual is a tappable chip. Tap "you accept one stop" → the entire Decision Screen re-renders under the new assumption, with a "1 change applied · reset" banner. The judge just *interacted with the model's decision boundary*. That's the demo moment. (Trivial to build: it's the same pipeline with one flag flipped.)

New module: `counterfactuals.ts`. Pure TypeScript, zero LLM, unit-tested. The LLM's only job is phrasing the results as prose.

## 5. Opportunity cost implementation note

All four alternatives come from the same scored candidate set — no new queries, no new scoring. `tradeOffSummary()` in `ranking.ts` generalizes from its current hardcoded direct-vs-1-stop pair into a `selectAlternatives(ranked): Alternative[]` function. Keep the old pair as the "Cheapest" seed.

## 6. Confidence: derived, never decorated

Two displayed numbers, both with defensible provenance:

**Match %** = champion's score ÷ maximum achievable score under this user's weights (i.e., the score of a hypothetical flight that maxes every component). This makes 91% mean something: "this flight captures 91% of what this traveler's profile says they want."

**Confidence tier** (High / Medium / Low) from two signals:
- **Margin**: score gap between #1 and #2. Gap > 0.10 → leans High; < 0.04 → Low ("this is close — see counterfactuals").
- **Signal agreement**: does `raw_history` evidence agree with structured fields? Both say direct-preferring → strong signal. Only one source, or conflict → weak.

Strong signals render with ✓, weak with •, exactly as you sketched. A **Low** confidence display is a feature, not a failure: *"This one's genuinely close — here's precisely what tips it"* is the most trustworthy sentence an AI can say, and it points straight at the counterfactual panel. The three zones reinforce each other.

Hard rule: no displayed number without a code path a judge can be walked through. If asked "where does 91% come from," the answer is a function, in under ten seconds.

## 7. How I decided: trace, not theater

Your instinct — expose the pipeline — is right. Your mechanism — show it during processing — is wrong, because the pipeline completes in milliseconds and simulated latency is the one thing that can make a technical judge distrust everything else on screen.

Instead: a persistent horizontal trace, rendered *after* results, every stage tappable:

```
request → preferences → constraints → candidates → trade-offs → counterfactuals → verdict
```

- **preferences** → the weight values + full evidence trail
- **constraints** → hard filters applied, and how many flights each one eliminated ("layover cap removed 12 of 47")
- **candidates** → the scored list with per-component score breakdown
- **counterfactuals** → the raw perturbation results, including the ones that *didn't* flip the winner

This is real data at every node — the API response already contains nearly all of it. The "flights eliminated per constraint" counts are the only new bookkeeping (a counter in the filter loop). If time remains on demo day, add a subtle staged reveal animation on first render — polish, strictly last.

## 8. Architecture

One principle: **deterministic core, LLM at the edges, one deployable.**

```
Next.js app
├── core/ (pure TS, zero framework imports, unit-tested)
│   ├── preferences.ts        ← exists
│   ├── ranking.ts            ← exists; generalize tradeOffSummary → selectAlternatives
│   ├── counterfactuals.ts    ← NEW, the flagship (§4)
│   ├── confidence.ts         ← NEW, small (§6)
│   ├── multicity.ts          ← permutation solver, kept from v1 plan
│   └── explain.ts            ← exists; prompt extended for counterfactuals + confidence
├── app/api/recommend/route.ts   ← one endpoint, returns everything the screen needs
└── app/page.tsx                 ← the Decision Screen
```

**On NestJS.** You listed it under "should remain." I'm challenging it anyway, which is the job you gave me. The case: (1) your actual `package.json` is already Next-only — v1's NestJS backend is aspirational, not built; (2) the toolkit confirms this is **solo participation** with ~36 hours left — a second service means a second deploy, CORS, and env plumbing, none of which a judge sees, all of which one person maintains alone; (3) the rubric's "robust backend logic" is satisfied by pure, unit-tested core modules — which judges *can* read — not by dependency-injection ceremony; (4) the toolkit's own FAQ says polished infrastructure isn't required, only that the solution be "logically sound and clearly demonstrated." Because `core/` imports no framework, wrapping it in NestJS later is a 30-minute job. Your call, but it's on the record.

**Data scale — the toolkit changes an assumption.** The flight file is **~50,000 records**, not the handful v1 planned around. Consequences, all cheap: load the CSVs once at server start into an in-memory store with a `Map<origin, FlightRow[]>` index (50k rows is ~15MB — trivially fine in memory, no database needed); route-filter *before* scoring so normalization runs over a route's candidates, not the whole file; and say so in the deck — "50,000 flights, indexed, sub-millisecond ranking" is a free credibility line. The multi-city graph also gets richer at this scale, which makes the permutation solver's optimality claim more impressive, not less.

**Everything else on your keep-list stays**: TypeScript, Next.js, LangChain, Groq (`llama-3.3-70b-versatile`, template fallback intact), the embedding-similarity layer in preference extraction (it's a rubric line: "semantic extraction, not just regex" — one caveat: `transformers.js` downloads model weights on first run; do that download *today*, not on demo day), multi-city permutation solver, deterministic scoring.

The response contract grows three fields; nothing else changes shape:

```typescript
interface RecommendResponse {
  verdict: ScoredFlight;
  ranked: ScoredFlight[];               // collapsed list, rubric coverage
  preference: InferredPreference;        // includes evidence[]
  alternatives: Alternative[];           // §3.3 — cheapest/fastest/flexible/comfort/date-shift
  counterfactuals: Counterfactual[];     // §4 — thresholds + flips
  confidence: { matchPct: number; tier: "high"|"medium"|"low";
                strongSignals: string[]; weakSignals: string[] };
  trace: TraceStage[];                   // §7 — real payloads per stage
  explanation: string;                   // LLM prose, grounded, with fallback
  itinerary?: MultiCityItinerary;        // multi-city mode
}
```

## 9. Multi-city: the second beat, not the show

Multi-city stays because the brief demands it — but it's demoed *through* the Decision Screen, not beside it: verdict = the full itinerary; opportunity cost = itinerary-level ("reversing Rome and Paris saves $210 but adds a redeye"); counterfactual = the ordering boundary ("Paris-first wins unless the ROM→PAR leg drops below $95"). Same six zones, itinerary-shaped. One demo beat, ~45 seconds, proving the decision framework generalizes beyond a single leg. Exact permutation search at ≤6 cities, as planned — the OR-Tools trade-off note from v1 survives verbatim in the README.

## 10. Build order for the hours that remain

Deadline: **July 12, 11:45 PM IST — roughly 36 hours out.** Sequenced so the product is demoable at the end of every block:

1. **`counterfactuals.ts` + `selectAlternatives` + `confidence.ts`** (~4–5h). Pure functions against the existing CSVs. Unit tests on the break-even math — this is what a technical judge will poke.
2. **API route** returning the full contract (~1h).
3. **Decision Screen, static** (~5–6h). Six zones, real data, no interactivity. *This alone beats most teams.*
4. **Tappable counterfactual chips** (~2h). The demo moment.
5. **Trace panel** (~2h).
6. **Multi-city through the screen** (~3h).
7. **Deliverables block** (~4h, protected — do not donate this block to code). The toolkit makes four artifacts mandatory, and half of them are judged on *communication quality*, so treat this as product work, not paperwork:
   - **Solution summary** (separate short doc): problem statement chosen → user problem ("booking is a decision problem, not a search problem") → solution (the Decision Screen + counterfactuals) → expected impact.
   - **Deck, 6–8 slides**: problem → solution overview → architecture/workflow → datasets used → demo video embedded → limitations & future work.
   - **Demo video, 3–5 minutes**: script it around the counterfactual chip tap as the climax. Run through the benchmark prompts on camera — that's the "demo evidence" criterion, verbatim.
   - **README**: setup, assumptions (documenting assumptions is *explicitly rewarded* in the FAQ), limitations, future improvements. The OR-Tools note and the fixed-normalization caveat from §4 live here.
   - Submission: single OneDrive folder, **"Everyone can view"** — verify the permission from an incognito window, since the FAQ warns a dead link can sink the evaluation — then email the link to careers@expediagroup.com before 11:45 PM IST.
8. Reveal animation, map, Pareto chart — only if hours remain, in that order. The map and chart from v1 are explicitly *cut from the critical path*: they decorate; the six zones argue. The toolkit's FAQ backs this scope call twice over: "a polished UI is not mandatory" and "teams may focus on one or more meaningful use cases... as long as the scope is clearly defined." Define it in the README: depth on decision reasoning over breadth of widgets.

If the schedule collapses, the floor is steps 1–3: a static Decision Screen with honest counterfactual math is still a winning demo. Everything after is amplitude.

**Worth-if-time, surfaced by the actual user data:** the flight file carries `on_time_performance` (81–93 in the sample), currently unused — and user U10's history reads "scared of missing connections, short layovers stress me." Those two facts want each other: for users whose evidence trail contains connection-anxiety signals, penalize connecting flights with tight layovers on low-OTP carriers, and add the counterfactual "a 90-minute buffer instead of 45 makes the 1-stop safe enough to win." It's ~20 lines on top of the existing scorer, it turns a dormant data column into a personalization story, and it demos as "the AI noticed this traveler is nervous." Strictly after step 7 — but if any polish item gets built, build this one before the map.

## 10.1 Demo casting — pick travelers like characters

The sample users aren't rows; they're characters, and the demo video is judged on storytelling. Cast three whose decision screens will look maximally different from each other:

- **U02 (Mexico City bargain hunter)** — "took a 7hr layover in SIN to save $120." Verdict: a cheap 2-stop. The counterfactual runs *in reverse*: "the direct wins only if its fare drops below $X" — proof the boundary logic isn't hardcoded to favor directs.
- **U05 (Lisbon, first-class, "money's not the constraint")** — the opportunity-cost panel inverts: cheapest option shown as what they're *declining*, and price counterfactuals come back "no realistic price changes this" — the honesty beat.
- **U03 (Amsterdam family, 2 kids, "direct is worth paying for," "kids melt down at night")** — richest evidence trail in the file; morning + direct + baggage constraints all fire at once, and the date-shift row respects "school breaks only."

Same screen, three unrecognizably different decisions — that's the personalization criterion demonstrated without saying the word "personalization." Verify each pick against `benchmark_prompts.json` first; if the benchmark prompts name specific users, those override this casting.

## 11. Judging-criteria mapping

The toolkit scores both the six "strong solutions" dimensions *and* a broader set: problem understanding, AI/data/reasoning approach, solution design, **innovation and differentiation**, demo evidence, **pitch clarity and storytelling**, video quality, presentation quality. Mapping:

| Judge criterion | Where it lives |
|---|---|
| Problem understanding & relevance | The reframe itself: "booking is a decision problem" — slide 1, solution summary ¶1 |
| Intelligent preference extraction | Evidence trail + strong/weak signals, on-screen (zones 2, 5) |
| Constraint-based reasoning | Trace panel: per-constraint elimination counts (zone 6); date-flexibility respected in §3.3 |
| Graph / route optimization | Multi-city beat (§9), over a 50k-flight graph |
| Personalization from behavior | Evidence cites `raw_history` phrases verbatim |
| Trade-off awareness & explanation | Zones 3 + 4 — the entire product |
| Flexible dates / seasonal awareness | Date-shift alternative + counterfactual; demand/holiday multipliers surfaced in score breakdown |
| Robust backend + clear outputs | Pure, unit-tested `core/` over 50k indexed records; one screen that needs no narration |
| Innovation & differentiation | Counterfactuals — the feature no LLM-ranking team can replicate (§4) |
| Storytelling & demo video | The chip-tap climax; §12 is the closing line of both the video and the live pitch |

## 12. The pitch, in the words judges leave with

*"Every other demo today showed you a ranking. We showed you a decision — with the evidence for it, the price of it, and the exact number at which we'd change our mind. Then we tapped that number, and the AI changed its mind in front of you."*

That's not another flight recommender.