# Saarathi — Agent Context

AI travel strategist for the Expedia hackathon. One screen (the Decision Screen)
answers: what to book, why, what you're giving up, and exactly what would change
the AI's mind (counterfactuals — the flagship). Deadline: Jul 12, 11:45 PM IST.

## Read before writing code (do NOT re-read every session; trust this file)
- docs/product_spec_v2.md    — what & why (screen zones §2-3, counterfactual math §4, confidence §6)
- docs/architecture_v2.md    — exact how (module contracts §4, API §5, tokens/components §6)
- docs/implementation_plan.md — phase order & definitions of done

## Non-negotiable rules
1. core/ imports NO framework code (no next/react) — ESLint-enforced. Pure functions + tests.
2. The LLM (Groq via LangChain) phrases explanations. It NEVER ranks, filters, or decides.
3. Both AI touchpoints (embeddings, Groq) have never-throw deterministic fallbacks.
4. Every number shown in the UI traces to a function (match%, tier, thresholds — no decoration).
5. No raw hex / bare <div>/<p>/<button> in components — tokens in globals.css + primitives only.

## Data gotchas (from real toolkit files)
- Booleans are Python-style "True"/"False" (capitalized). Coerce at load, once, in core/data.ts.
- flight_numbers & layover_airports: semicolon-delimited. layover_airports empty for nonstops.
- raw_history: pipe-delimited (" | ") quotable phrases — pre-split; evidence cites them verbatim.
- departure_utc date component is load-bearing (date-shift feature groups candidates by date).
- ~50k flights: index Map<origin> and Map<`${origin}-${destination}`> at boot; normalize per-route.
- NEVER load full CSVs into your own context — inspect via scripts (head/wc/node one-liners).

## Token discipline
- Answers from this file > re-reading docs. Re-read a doc section only when implementing it.
- Prefer running tests over pasting long file contents into chat.

## Current phase  <!-- UPDATE as phases complete -->
Phase 0 — scaffold & de-risk. Done. (Next.js scaffolded, Xenova cached, Groq verified, dates are dense [avg 8.78 dates/route over 50k flights] -> date-shift feature is go).
Phase 1 — Data layer. Done. (Served 50k in-memory dataset, sub-millisecond lookup times, robust typing/coercion).
Phase 2 — Core decision engine. Done. (Completed preferences, ranking, algebraic price solver, toggle perturbations, confidence, multicity routing, explain templates, and all 22 tests passing).
Current Phase: Phase 3 — API.

## Benchmark prompts summary  <!-- FILL after reading data/benchmark_prompts.json -->
- **B01 (User U01, Cape Town)**: Single city to Tokyo next month. Must respect direct_preference="strong", max_layover_minutes=120, price_sensitivity="low".
- **B02 (User U02, Mexico City)**: Multi-city (London + Paris + Rome). Must respect direct_preference="none", max_layover_minutes=420, price_sensitivity="high".
- **B03 (User U03, Amsterdam)**: Single city to Bali (DPS) over summer. Must respect direct_preference="strong", max_layover_minutes=150, price_sensitivity="medium", date flexibility.
- **B04 (User U04, Melbourne)**: Single city to New York (meeting Tuesday, return Thursday). Must respect direct_preference="moderate", max_layover_minutes=300, price_sensitivity="high".
- **B05 (User U05, Lisbon)**: Single city to Sydney around the holidays. Must respect direct_preference="strong", max_layover_minutes=90, price_sensitivity="none", seasonal/holiday multipliers, seat scarcity.
- **B06 (User U06, Tokyo/HND)**: Multi-city Asia trip with 3 weeks flexibility. Must respect direct_preference="none", max_layover_minutes=480, price_sensitivity="high".
No mismatches found. The spec's single-leg and multi-city routing engines are fully aligned with these benchmark expectations.
