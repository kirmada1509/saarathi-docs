# Saarathi: Expedia AI Hackathon Submission

**Saarathi** (meaning "chariot driver" or "guide") is an advanced AI-powered flight decision engine that parses traveler preferences, ranks flight choices through a multi-attribute linear utility function, computes decision boundaries (price thresholds and constraint tweaks), and renders a premium dark-navy dashboard.

The application is structured as a **decoupled, two-tier codebase** separating the user interface from the API and data layer.

---

## 1. Key Engineering & Architectural Decisions

### A. Decoupled Architecture
*   **`saarathi-frontend`:** Next.js 16 client (App Router, Turbopack, React Server Components) running on port `3000`. It consumes the backend REST endpoints and manages UI interactions using a client-side Zustand store.
*   **`saarathi-backend`:** NestJS framework backend running on port `4000`. It exposes the REST endpoints `GET /api/users` and `POST /api/recommend` and manages database operations using Prisma 5.

### B. In-Memory SQLite Hybrid Datastore
To handle the 50,000 flight records and traveler profiles:
1. **SQLite Database:** A local, single-file database (`dev.db`) initialized and queried via **Prisma 5**.
2. **Warm-up Cache on Startup:** When the NestJS application boots, it executes `initializeStoreFromDb(prisma)` which queries all database records and builds in-memory route indices. This ensures that route lookups, utility ranking, and multi-city permutations run in **sub-millisecond speed** without disk I/O bottlenecks.
3. **Seeding:** The database is seeded by parsing the packaged CSV datasets (`user_data.csv` and `flights_data.csv`) using a chunked transaction writer to avoid SQLite parameter bounds.

### C. Closed-Form Competitor Pricing Break-Even Formula
To determine exactly how much a cheaper competitor's price must drop to flip the decision and overtake the champion, we solve the utility equation analytically rather than running iterative search loops.

The final score $S_i$ of flight $i$ is calculated as:
$$S_i = W_d \cdot Score_{direct}(i) + W_c \cdot Score_{price}(i) + W_v \cdot Score_{convenience}(i) + W_l \cdot Score_{loyalty}(i) + W_b \cdot Score_{cabin}(i)$$

Let:
- $S_w$ be the score of the current champion (winner).
- $S_c$ be the score of a cheaper competitor *excluding its price score component*.
- $P_c$ be the current price of the competitor.
- $P_w$ be the current price of the champion.
- $P_{min}$ and $P_{max}$ be the minimum and maximum prices on the route.

The price score component is normalized linearly:
$$Score_{price}(i) = 1.0 - \frac{P_i - P_{min}}{P_{max} - P_{min}}$$

To flip the winner, the competitor's new score $S'_c$ must equal the champion's score $S_w$:
$$S'_c = S_c + W_p \cdot \left(1.0 - \frac{P'_c - P_{min}}{P_{max} - P_{min}}\right) = S_w$$

Solving for the new target price $P'_c$:
$$P'_c = P_{min} + \left(1.0 - \frac{S_w - S_c}{W_p}\right) \cdot (P_{max} - P_{min})$$

The required price drop $\Delta P_c$ is:
$$\Delta P_c = P_c - P'_c$$

This allows the UI to display precise, mathematically rigorous statements like:
> `"United 88 becomes the winner if its price drops by $89 (from $450 to $361)"`

If the required price drops below zero, the option is flagged as **Unachievable** (the competitor cannot win even if it were free, due to convenience or airline loyalty penalties).

### D. Vector Archetype Embedding Matcher
To map natural language queries (like *"Dates are flexible, I want to spend a week in Bali next summer"*) to structured preference weights:
- We use Xenova's ONNX runtime port of the **`all-MiniLM-L6-v2`** model.
- The model runs **entirely in-memory inside the NestJS server** (no external vector database needed).
- User query chunks are compared against pre-embedded archetype phrases for each preference dimension (Direct preference, Cost sensitivity, Convenience, Redeye flight avoidance, Cabin tier match, and Airline loyalty).
- A cosine similarity threshold of **`0.78`** is applied to infer weights, which are then boosted by the traveler's historical booking profile.

### E. Multi-City Tour Sequence Router
For multi-destination searches (like European or Asian tours):
1. The engine generates all tour order permutations (e.g. `MEX-LHR-CDG-FCO-MEX` vs. `MEX-CDG-FCO-LHR-MEX`).
2. It filters candidate flights by checking temporal turnaround constraints:
   - Ensuring sequential leg departure dates are chronological.
   - Enforcing a minimum layover interval (e.g. at least 1 day or 24 hours between legs of a multi-city vacation).
3. It selects the optimal combination that maximizes the combined flight utility score.
4. **Leg Scoping:** The UI timeline displays the complete tour. Clicking any leg scopes the decision zones (weights, alternatives, and break-even boundaries) to that segment on the fly.

---

## 2. Technical Stack

### Frontend (`saarathi-frontend/`)
- **Framework:** Next.js 16 (App Router, Turbopack, React Server Components).
- **Styling:** Vanilla CSS styled with custom CSS variables inside Tailwind CSS v4 semantic tokens. Styled in dark navy/amber aesthetics.
- **State Management:** Zustand (for client-side traveler selection, custom query input, active perturbation toggles, and decision trace sheets).
- **Linter:** ESLint (strictly configured flat file enforcing zero bare layout tags like `div`, `p`, and `button` inside client features, driving pure primitive layout consistency).

### Backend (`saarathi-backend/`)
- **Framework:** NestJS (TypeScript, Dependency Injection, REST Modules).
- **Database:** Prisma 5 ORM with a local SQLite file database (`dev.db`).
- **AI Integration:** LangChain Core & Groq SDK for generating rationales.
- **Vector Embeddings:** ONNX Runtime (`@xenova/transformers`) running in-memory.
- **Unit Testing:** Vitest (22 test cases verifying preferences, data indexing, confidence gauges, multi-city routing, and counterfactual equations).

---

## 3. Benchmark Suite Results

Our comprehensive test suite (`src/core/benchmark.test.ts`) verifies the engine against the 6 Expedia hackathon benchmark scenarios:

| Prompt ID | Traveler | Destination(s) | Key Constraints Verified | Status |
|---|---|---|---|---|
| **B01** | `U01` | `HND` (Tokyo) | Direct flights, business cabin class preference | **PASS** |
| **B02** | `U02` | `LHR`+`CDG`+`FCO` | Multi-city Europe tour permutation solver, chronological layout | **PASS** |
| **B03** | `U03` | `DPS` (Bali) | Flexible dates, shift-dates opportunity cost check | **PASS** |
| **B04** | `U04` | `JFK` (New York) | Business meeting, Tuesday-Thursday date window matching | **PASS** |
| **B05** | `U05` | `SYD` (Sydney) | Strict 90m layover cap; triggers zero-matching fallback with advice | **PASS** |
| **B06** | `U06` | `SIN`+`KUL` | Asia multi-city routing, temporal vacation sequence | **PASS** |

---

## 4. Run & Setup Instructions

Both codebases are fully self-contained. The SQLite database is built and seeded using the packaged CSV datasets located in the root `/data` folder.

### A. Environment Configuration
Create a `.env` file inside `saarathi-backend/`:
```bash
DATABASE_URL="file:./dev.db"

# Required for AI rationale generation.
# If not provided, the app will fall back to local templates.
GROQ_API_KEY=your_groq_api_key_here
```

### B. Seed the Database
From the workspace root, compile the database tables and insert records:
```bash
cd saarathi-backend
npx prisma db push
npx prisma db seed
cd ..
```

### C. Run the Applications
Open two terminal windows from the workspace root:

*   **Terminal 1 (Backend):**
    ```bash
    npm --prefix saarathi-backend run start:dev
    ```
    *Starts the NestJS backend at `http://localhost:4000`.*

*   **Terminal 2 (Frontend):**
    ```bash
    npm --prefix saarathi-frontend run dev
    ```
    *Starts the Next.js client at `http://localhost:3000`.*

### D. Run the Test Suite
To execute all backend unit and benchmark tests:
```bash
npm --prefix saarathi-backend test
```
