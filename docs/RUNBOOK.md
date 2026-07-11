# Runbook

## Cold start (two terminals)

```bash
# Terminal 1 — backend, port 4000
cd saarathi-backend
npm install
npx prisma generate
npm run start:dev
# wait for: "[Saarathi Store] Loaded 50 users, 50000 flights, indexed 1172 routes from DB in ###ms."
# then:     "[Saarathi Embeddings] Model and archetype embeddings loaded successfully."
```

```bash
# Terminal 2 — frontend, port 3000
cd saarathi-frontend
npm install
npm run dev
# wait for: "Ready in ###ms" — note the printed port, Next.js silently moves to 3001/3002 if 3000 is busy
```

Open `http://localhost:3000` (or whatever port Terminal 2 printed).

## If the database is empty or missing

```bash
cd saarathi-backend
npx prisma migrate deploy      # or: npx prisma db push
npx prisma db seed             # runs prisma/seed.ts — reads ../data/*.csv, writes prisma/dev.db
```
Verify it worked:
```bash
sqlite3 prisma/dev.db "select count(*) from User; select count(*) from Flight;"
# expect: 50 \n 50000
```

## Run all tests

```bash
cd saarathi-backend
npm test              # Vitest — the only working test runner. Expect: 9 files, 35 tests, all pass.
```
Do **not** run `npm run test:watch`, `test:cov`, `test:debug`, or `test:e2e` for a demo — all four are currently broken (wrong Jest pattern / ESM import crash). See `docs/HANDOFF.md` §8 if you need to fix them first.

Frontend has no test suite to run (`vitest.config.ts` exists but there are zero test files and no `"test"` script).

## Run the benchmark scripts

Backend must already be running (`start:dev`, Terminal 1 above).
```bash
cd saarathi-backend
npx ts-node test_benchmarks.ts          # regenerates BENCHMARK_REPORT.md — 6 official prompts
npx ts-node custom_test_benchmarks.ts   # regenerates CUSTOM_BENCHMARK_REPORT.md — 8 extra scenarios
```
Both hit the live API on `:4000` — expect ~1s per scenario, all 14 should report PASS.

## Rebuild for demo recording

```bash
# Backend — production build
cd saarathi-backend
npm run build            # nest build → dist/
npm run start:prod       # node dist/main — same port 4000, no file-watcher overhead

# Frontend — production build
cd saarathi-frontend
npm run build            # next build
npm run start             # next start — port 3000, production-optimized, no dev overlay
```
Prefer these over `dev`/`start:dev` for the actual recording — no HMR overlay, no watch-mode recompilation stutter, more representative latency numbers.

## Pre-demo checklist

- [ ] `saarathi-backend/.env` has `DATABASE_URL="file:./dev.db"` and a non-empty `GROQ_API_KEY`
- [ ] `saarathi-frontend/.env.local` has `NEXT_PUBLIC_API_BASE_URL=http://localhost:4000` and a valid `NEXT_PUBLIC_MAPTILER_KEY`
- [ ] `sqlite3 saarathi-backend/prisma/dev.db "select count(*) from Flight;"` → `50000` (not the stale root-level `dev.db`)
- [ ] Embedding model log line ("Model and archetype embeddings loaded successfully") has printed — first request will hang until it does if you skip this
- [ ] Both ports confirmed: `lsof -i :4000` and `lsof -i :3000` each show exactly one process
- [ ] One smoke request fired and eyeballed:
  ```bash
  curl -s -X POST http://localhost:4000/api/recommend \
    -H "Content-Type: application/json" \
    -d '{"userId":"U01","requestText":"I need to get from home to Tokyo next month, what do you suggest?","destination":"NRT","perturbations":[]}' \
    | head -c 300
  ```
  Expect a `200`-shaped JSON starting with `{"mode":"single-leg",...`, not an error object.
- [ ] Load `http://localhost:3000/app/decision?userId=U01&requestText=...&destination=NRT` in the actual browser you'll demo from and confirm the map tiles render (not the "Map unavailable" empty state) — this is the one failure mode that's silent in the terminal.

## Three most likely failure modes at demo time

**1. "Address already in use" on port 4000 or 3000.**
A previous run's process is still alive.
```bash
lsof -i :4000   # or :3000 — find the PID
kill -9 <PID>
```
Then restart the affected service. (30s)

**2. First `/api/recommend` call after a fresh boot hangs for several seconds or the explanation text reads generic/templated instead of natural prose.**
The embedding model was still loading, or `GROQ_API_KEY` didn't load / Groq rate-limited you — both are silent, never-throw fallbacks, not crashes. Check the backend terminal for `[Saarathi Embeddings] Model and archetype embeddings loaded successfully` before your first live query, and confirm the key:
```bash
grep GROQ_API_KEY saarathi-backend/.env
```
If the explanation still reads templated ("For U01, the top pick is..."), Groq is unreachable — check network, or just narrate that the fallback is deterministic and working as designed. (30s to diagnose, not always fixable live)

**3. Map shows "Map unavailable" instead of tiles.**
`NEXT_PUBLIC_MAPTILER_KEY` is missing or the frontend was started before it was set.
```bash
grep NEXT_PUBLIC_MAPTILER_KEY saarathi-frontend/.env.local
```
If it's empty, add a key from `cloud.maptiler.com` (free tier) and **restart** the frontend dev/prod process — Next.js inlines `NEXT_PUBLIC_*` vars at server start, a running process won't pick up a `.env.local` edit. (30s if the key is already set, otherwise blocked on getting one)
