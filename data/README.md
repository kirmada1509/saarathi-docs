# Data (sample)

This repo holds documentation and a **sample** of the dataset for reference — not the dataset the running application actually seeds from.

- `user_data.csv` — full file, 50 rows (small enough to include as-is).
- `benchmark_prompts.json` — full file, the 6 official benchmark prompts (B01-B06).
- `flights_data.sample.csv` — a **500-row stratified sample** (every 100th row) of the real 50,000-row `flights_data.csv`, covering 400 of the dataset's 1,172 unique origin-destination routes. Same header/schema as the full file. Generated with:
  ```bash
  awk 'NR==1 || NR%100==2' flights_data.csv > flights_data.sample.csv
  ```

The full 50,000-row `flights_data.csv` (~9.5MB) is intentionally **not** included here to keep this repo small — it lives in `saarathi-backend/../data/flights_data.csv` and `saarathi-frontend/../data/flights_data.csv` in the main project repos, and is what `saarathi-backend/prisma/seed.ts` actually loads into the live database. See `docs/HANDOFF.md` §5 for the full data-layer story.
