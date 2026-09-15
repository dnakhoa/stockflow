# Solvers and data for min-cost flow on free-tier infrastructure

Timings below were **measured**, not quoted. Environment: Apple M3, single-threaded, Node 26.8.2,
Python 3.9.6. The benchmark instance matches this project's shape — 1 DC + 50 stores (51 nodes),
integer supplies and demands summing to zero — in a sparse variant (each store linked to 10
neighbours plus the DC both ways, 600 arcs) and a dense variant (full lateral 50x49 plus DC,
2,550 arcs). Every solver returned an identical objective on the same instance, so the
cross-solver comparison is like-for-like.

Caveats are listed at the end and should be read before relying on any number.

## 1. The field

The JavaScript optimization ecosystem is genuinely thin. `highs`, `glpk.js`, `jsLPSolver`/`YALPS`,
and now `or-tools-wasm` — that is the whole of it.

| Library | Version | Licence | Maintained | Browser | Node | Python |
|---|---|---|---|---|---|---|
| Google OR-Tools | 9.15 | Apache-2.0 | very active | **no official build** | no | yes |
| `or-tools-wasm` (community) | 0.9.1 | Apache-2.0 | young — repo created 2026-05-13 | yes (needs COOP/COEP) | yes | — |
| HiGHS | 1.15.1 | MIT | very active | via `highs` | via `highs` | `highspy` |
| npm `highs` (*not* `highs-js`) | 1.15.3 | MIT | active, 2026-09-11 | yes | yes | — |
| SciPy | 1.18.1 | BSD-3 | yes | no | no | yes |
| NetworkX | 3.6.1 | BSD-3 | yes | no | no | yes |
| glpk.js | 5.0.0 | **GPL-3.0** | borderline, ~9 months | yes | node subpath only | — |
| jsLPSolver | 1.0.3 | Unlicense | yes | yes | yes | — |
| YALPS | 0.6.4 | MIT | borderline, ~9 months | yes | yes | — |
| lp_solve | 5.5.2.14 | LGPL-2.1 | revived, moved to GitHub 2025-07 | no | no | `lpsolve55` |

**Unmaintained — do not use:** npm `min-cost-flow` (last publish 2020-12, and semantically wrong
— it solves source-to-sink flow for a desired value, not supply/demand transshipment, and
returned a different objective from every other solver); the `lp_solve` npm binding (2022-12);
`glpk-wasm` (2021-05); `linear-program-solver`, `osqp.js`, `simplex-solver`, `milp`.

**`glpk.js` is GPL-3.0**, which rules it out for a permissively licensed project regardless of
its merits.

**OR-Tools has no official WASM or JavaScript build.** Its README states the wrappers are Python,
C# and Java only. The community `or-tools-wasm` port does expose `SimpleMinCostFlow` with the
upstream API and produces objectives identical to HiGHS — but it is **332 MB unpacked**, against
Vercel's **250 MB uncompressed Node bundle limit**. The bulk is duplicated browser and node WASM
trees; the min-cost-flow path needs only `graph_runtime_node.wasm` at **0.88 MB**, so file
tracing brings it to roughly 2 MB. That must be verified in a real deploy — naive tracing will
exceed the limit. It is also one person's four-month-old project: a supply-chain risk.

## 2. Measured performance

### 51-node instance — this project's actual shape

| Solver | sparse (600 arcs) | dense (2,550 arcs) |
|---|---|---|
| OR-Tools 9.15, native Python | **0.212 ms** | — |
| glpk.js (node, WASM) | 2.01 ms | — |
| or-tools-wasm (Node) | **2.22 ms** | 9.7 ms |
| SciPy `linprog(method='highs')` | 2.5 ms | 5.5 ms |
| NetworkX `network_simplex` | 3.0–3.5 ms | 8.0 ms |
| YALPS (pure JS) | 3.8 ms | — |
| npm `highs` | 6.8–16.3 ms | 20.0 ms |
| jsLPSolver (pure JS) | 7.1 ms | — |

`highs` is slower than it should be because its only API accepts an **LP-format text string** —
serialisation and re-parse dominate the cost at this size, not the simplex itself.

### Scaling — the decisive result

| nodes / arcs | OR-Tools (Py) | or-tools-wasm | NetworkX | SciPy+HiGHS | npm `highs` |
|---|---|---|---|---|---|
| 51 / 600 | 0.21 ms | 2.2 ms | 3.0 ms | 2.5 ms | 6.8 ms |
| 201 / 2,400 | 0.89 ms | 16.2 ms | 15.5 ms | 8.6 ms | 19.4 ms |
| 1,001 / 12,000 | 6.8 ms | 92.6 ms | 131 ms | 388 ms | 368 ms |
| 5,001 / 60,000 | 47.8 ms | 419 ms | 1,922 ms | 7,531 ms | 7,871 ms |
| 20,001 / 240,000 | 280 ms | 1,798 ms | — | **293,422 ms** | — |
| 501 / 250,500 (dense) | 39.7 ms | — | — | 2,970 ms | — |

Three conclusions:

1. **A dedicated network simplex crushes general LP on flow problems.** At 5,001 nodes OR-Tools
   is 157x faster than HiGHS-as-LP. At 20,001 nodes the general LP takes nearly five minutes and
   would blow a Vercel invocation; OR-Tools finishes in 280 ms.
2. **NetworkX's pure-Python network simplex beats SciPy+HiGHS from ~1,000 nodes up** — 131 ms
   vs 388 ms, 1.9 s vs 7.5 s. The right algorithm in a slow language beats the wrong algorithm
   in a fast one. This is the cleanest empirical statement of the project's premise available.
3. **NetworkX's practical ceiling** is ~1,000 nodes comfortably, ~5,000 painfully, unusable
   beyond ~10–20k. For a 51-node decomposition it is entirely adequate, and it makes an
   excellent cross-check oracle.

## 3. Postgres cannot solve this

**pgRouting is available on Supabase** (current upstream 4.0.2, GPL-2.0). Its entire *official*
Flow family — `pgr_maxFlow`, `pgr_pushRelabel`, `pgr_edmondsKarp`, `pgr_boykovKolmogorov` — is
**max-flow, uncosted**. None solves min-cost flow.

The only cost-aware functions, `pgr_maxFlowMinCost` and `pgr_maxFlowMinCost_Cost`, are
**experimental**, and pgRouting's own documentation carries this warning on that section:
*"Possible server crash — These functions might create a server crash."* They also solve
*max-flow at minimum cost*, which is a different problem from supply/demand transshipment.
Betting a replenishment run on an experimental function upstream warns may crash the server, on
shared-CPU free-tier Postgres, is a bad trade.

**`plpython3u` is not available on Supabase** — it is an untrusted language requiring superuser.
Available procedural languages are `plpgsql`, `pljava`, and the deprecated `plv8`/`plcoffee`/`plls`.
So there is no calling SciPy from a stored procedure.

**Recursive CTEs cannot express min-cost flow.** They handle reachability, transitive closure and
shortest paths, but min-cost flow requires pivoting and augmentation against a *global* optimality
condition on reduced costs — an iterative fixed point over mutable state, not a monotone closure.
Simulating it with a PL/pgSQL loop means writing a solver, badly, in the wrong place, while
burning the shared-CPU budget the database needs for queries.

**Use Postgres for what it is good at here:** store the graph, aggregate demand and on-hand per
(store, SKU), do the per-SKU grouping in SQL, and write the resulting plan back. Pull the
instance out, solve in application code, write results back.

## 4. Sizing — 50 stores x 5,000 SKUs

Assumptions: single-commodity decomposition into 5,000 independent 51-node sparse instances
(multiply by ~3–4x for full lateral); a **2x derating estimate** from Apple M3 to Vercel Hobby's
1 vCPU; daily run, 30 days. Verified Hobby limits: 300 s max duration (default *and* maximum),
2 GB / 1 vCPU fixed, **4 Active-CPU-hours = 14,400 CPU-seconds per month**, 250 MB uncompressed
Node bundle (500 MB for Python functions).

| Solver | ms/SKU | x5,000 | daily CPU (derated) | monthly | % of budget |
|---|---|---|---|---|---|
| OR-Tools native Python | 0.212 | 1.06 s | **2.1 s** | 64 s | **0.44%** |
| or-tools-wasm (Node) | 2.22 | 11.1 s | **22 s** | 666 s | **4.6%** |
| SciPy `linprog(highs)` | 2.5 | 12.5 s | 25 s | 750 s | 5.2% |
| NetworkX `network_simplex` | 3.15 | 15.8 s | 32 s | 947 s | 6.6% |
| npm `highs` | 6.8–16.3 | 34–82 s | 68–164 s | 2,040–4,920 s | 14–34% |

**It fits one invocation comfortably** — worst case ~164 s against a 300 s ceiling, sensible
options 20–40 s. **The monthly CPU budget is not a constraint**, with recommended options under
5%. Two things to watch: the 4 hours is **account-wide** across every project, so dev and preview
runs draw on the same pool; and **Active CPU excludes I/O**, so Supabase round trips are free on
the meter.

Overheads are small. The 250,000 (store, SKU) input rows measure **15.5 MB of JSON**, ~143 ms to
serialise and ~103 ms to parse — against Supabase's 5 GB monthly egress, a daily pull is ~465 MB.
`or-tools-wasm` cold start is ~57 ms, amortised to zero thereafter.

**Run it as ONE invocation looping 5,000 times.** Fanning out to 5,000 invocations pays 5,000 cold
starts and WASM inits.

**Recommendation:** OR-Tools via a Vercel **Python** function is the fastest and most
battle-tested path, and Python functions get a 500 MB bundle limit against Node's 250 MB (the
`ortools` venv measured 145 MB). `or-tools-wasm` on Node is the single-runtime alternative, with
the bundle-trimming caveat. NetworkX is a respectable fallback and a good oracle for
cross-checking. **Do not route min-cost flow through a general LP solver.**

## 5. Benchmarks and data

### Instance sets

- **LEMON MCF benchmark data** (`lemon.cs.elte.hu`, maintained by Péter Kovács) — the canonical
  min-cost-flow test set: NETGEN, GRIDGEN, GOTO, GRIDGRAPH, plus road-network and vision
  instances, all DIMACS MCF format. Kovács, *Minimum-cost flow algorithms: an experimental
  evaluation* (EGRES TR-2013-04) is the reference performance study.
- **13th DIMACS Implementation Challenge: Network Flows 2.0** — active and curated, with separate
  MCF / assignment / max-flow tracks, each with instances, generators **and reference solvers**.
  The best modern starting point.
- **MIPLIB 2017** and the **Mittelmann benchmarks** (updated 2026-09) — the latter has a *Large
  Network-LP Benchmark* page directly on point, testing HiGHS and OR-Tools GLOP among others.
- **OR-Library** (Beasley) — has capacitated warehouse location, p-median, p-hub and lot sizing,
  but **no pure transshipment or multi-echelon inventory set**.
- **Multi-echelon inventory:** the canonical public set is the **Willems (2008) MSOM data set**
  (38 real supply chains). Access terms **unverified** — INFORMS returned 403 to automated
  fetches. There is no MIPLIB equivalent for this field.

### Retail demand datasets — licence is the binding constraint

| Dataset | Per-store, per-SKU history? | Licence |
|---|---|---|
| **Corporación Favorita** (2017) — 54 stores, ~4,000 items, daily | **yes — best fit** | non-commercial, **no redistribution** |
| **M5 / Walmart** — 3,049 products x 10 stores, 1,941 days | **yes**, but only 10 stores | non-commercial, academic only |
| **VN1** (2024) — 15,053 products x 328 warehouses | **yes**, richest location structure | **no licence statement found** |
| Store Sales (Kaggle starter) | store yes, **SKU no** — product *family* only | non-commercial |
| Walmart Recruiting | store yes, **department not SKU** | rules page sign-in gated, unverified |
| **Rossmann** | **no SKU dimension at all** — daily store turnover only | unverified |
| Olist / Retailrocket | no | CC BY-NC-SA 4.0 |
| UCI Online Retail II | **no — explicitly "non-store"** | **CC BY 4.0** — the only permissive one, and useless here |
| dunnhumby Complete Journey | partial — household panel with a store dimension | registration-gated, terms unverified |

**The honest position:** only Favorita, M5 and possibly VN1 have the structure this project needs,
and **all are non-commercial with explicit no-redistribution clauses**. They cannot be bundled
into the repository or into a deployed demo. Every permissively licensed retail dataset lacks
multi-location SKU structure entirely.

**Consequence: the public demo must run on synthetic data.** Real datasets are for development
and for a published evaluation only. Generating credible synthetic multi-store demand — with
seasonality, promotions, cross-store correlation and lead-time variance — therefore becomes a
real component of the project rather than a convenience.

## Not verified

1. **Which pgRouting version Supabase ships**, and whether `pgr_maxFlowMinCost` is present at all.
   Supabase's own docs advertise only path-finding functions. Check on a live project.
2. **Rossmann and Walmart-Recruiting data-use clauses** — rules pages require sign-in.
3. **dunnhumby terms** (registration-gated) and **VN1 licence** (none found).
4. **Willems MSOM data set access terms** — INFORMS returned 403.
5. **The 2x M3-to-Vercel derating factor is an estimate**, not measured on Vercel hardware. The
   per-SKU millisecond figures are measured; the Vercel column is those figures times an assumed
   constant.
6. Benchmark instances are a **synthetic construction** matching the described topology, not a
   published set. Cross-solver objectives agree exactly, so the comparison is sound, but absolute
   times will move with real cost and capacity structure.
