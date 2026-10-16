# QuantumFlow AI — Quantum-Inspired Traffic Flow Optimizer

Full-stack Next.js + PostgreSQL app that simulates a four-way intersection and compares
**A · Fixed timing**, **B · Classical adaptive** and **C · Quantum-inspired (simulated quantum annealing heuristic)**
signal control on identical, seeded traffic demand.

> Stack note: the platform for this build is Next.js/TypeScript, so the "backend" (simulation engine,
> optimizers, REST API, validation) is implemented as Next.js route handlers instead of FastAPI/Python.
> The same engine (`src/lib/engine.ts`) runs on the server and in the browser, so the results match exactly.

## Run
```bash
cp .env.example .env          # set DATABASE_URL
npm install
npx drizzle-kit push          # creates the `experiments` table
npm run dev                   # http://localhost:3000
node --experimental-strip-types --test tests/engine.test.mjs   # unit tests (11)
npm run build && npm start    # production
```
Without a reachable database, the live dashboard, `/api/optimization/run` (returns results with a
warning) and `/api/benchmark` still work. Saving and listing experiments need PostgreSQL.

## Pages
- `/` Live dashboard: animated SVG intersection driven by sim state, Start/Pause/Resume/Step/Reset, speed, scenarios, all inputs, live metrics, live reduction vs a fixed-timing run in lock-step on the same arrivals, decision explanation.
- `/compare` Experiments: create / re-run (same seed) / delete / load config, CSV + Markdown report export, comparison charts, queue history, green allocation, light/medium/heavy/asymmetric benchmark with 1–5 trials.
- `/about` Hackathon overview with measured results from the latest experiment.

## API (`GET /api/docs` for the list)
`POST /api/simulation/start|pause|reset`, `GET /api/simulation/state`, `POST /api/optimization/run`,
`POST /api/benchmark`, `GET /api/experiments`, `GET|DELETE /api/experiments/{id}`,
`GET /api/experiments/{id}/comparison|export|report`, `POST /api/experiments/{id}/rerun`.
Invalid input returns 422 with a list of validation errors.

## Algorithms
- **Safety controller:** fixed ring NS-green → NS-yellow → all-red → EW-green → EW-yellow → all-red. New splits are applied only when NS-green begins. Every plan is checked (`min ≤ g ≤ max`, `gNS+gEW+lost = cycle`); invalid plans fall back to fixed timing. A runtime invariant counts conflicting greens (always 0).
- **A Fixed:** equal split of effective green.
- **B Adaptive:** critical-lane demand (queue + observed rate × cycle) → proportional split, clamped.
- **C Quantum-inspired:** 4 coupled replicas anneal over a 3-cycle horizon. Moves are local ±1 s steps plus "tunneling" jumps whose width shrinks like a decaying transverse field. The cost model forward-simulates *expected* demand only. The first cycle of the best schedule is applied (MPC). Cost weights can be set in the UI. **This is a classical CPU heuristic. It is not quantum computing and it does not show quantum advantage.**

## Sample results — MEASURED (single run, seed 42, 900 s, default timing, generated Poisson demand)
| Scenario | Fixed | Adaptive | Quantum-inspired |
|---|---|---|---|
| Light | 12.08 s | 12.79 s | 12.83 s |
| Medium | 13.66 s | 14.40 s | 15.34 s |
| Heavy | 36.64 s | 28.47 s | 28.40 s |
| Asymmetric | 47.93 s | 10.88 s | 10.89 s |

Under light and medium demand, both optimizers do slightly worse than fixed timing. Under heavy and asymmetric demand, they cut wait time a lot. The QI heuristic costs about 230–380 ms of CPU per run, while the adaptive method needs less than 1 ms.

## Known limitations
Single-lane point-queue model with no turns or spillback; synthetic demand only; no Qiskit/SUMO module; one shared in-memory server simulation instance.

## Deploy
Any Node 20+ host (Vercel, Render, Fly, Docker) with a PostgreSQL `DATABASE_URL` secret: `npm ci && npx drizzle-kit push && npm run build && npm start`.
