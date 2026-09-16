# Lateral transshipment, min-cost flow, and the total unimodularity boundary

Theory only — solvers, datasets and empirical benchmarks are researched separately.
Citations were resolved against CrossRef or publisher sources; items marked *inferred* are
deductions rather than published findings, and are flagged in place.

## 1. The taxonomy

The reference survey is Paterson, Kiesmüller, Teunter & Glazebrook (2011), *Inventory models
with lateral transshipments: A review*, EJOR 210(2):125–136,
[10.1016/j.ejor.2010.05.048](https://doi.org/10.1016/j.ejor.2010.05.048).

**By trigger.** *Proactive* (also preventive, redistribution): stock is redistributed among
stocking points at predetermined moments, **before demand is realised**. Because it is
arranged in advance, per-unit handling cost is low — the review identifies this as the retail
instrument. *Reactive* (also emergency): triggered by an actual or imminent stockout while
another location holds stock. Suited to regimes where transshipment cost is small relative to
holding and shortage costs; the canonical setting is spare parts.

**By pooling.** *Complete pooling* — the sender shares all on-hand stock. *Partial pooling* —
part is held back for the sender's own demand. The review notes partial pooling systems "are
more difficult to control and optimize… as there is the additional managerial decision of how
much inventory to reserve."

**Lateral means same echelon.** The review excludes reallocation of in-pipeline stock and
excludes emergency shipments from another echelon unless lateral transfers are also present.

Two results worth carrying: Tagaras (1989) finds complete pooling superior in his system, but
**Herer & Rashit (1999) show that with positive ordering costs, partial pooling can beat
complete pooling.** And Zhao, Deshpande & Ryan (2006) prove a three-parameter (S,K,Z) policy
optimal in a decentralized backorder model, where Z>0 gives partial pooling and Z=0 complete,
K≤0 shortage-triggered and K>0 risk-triggered — so one policy family spans the whole taxonomy.

## 2. The min-cost flow equivalence

**Claim.** With net positions known (post-demand, or deterministic demand), single-period
single-SKU reallocation is *exactly* a minimum-cost flow problem — a transportation problem
when the lane graph is bipartite.

With surpluses `s_i` at `i ∈ S`, deficits `d_j` at `j ∈ D`, per-unit lane cost `c_ij`:

    min  Σ c_ij · x_ij
    s.t. Σ_j x_ij ≤ s_i,   Σ_i x_ij ≤ d_j,   0 ≤ x ≤ u

which is `min { cᵀx : Nx = b, l ≤ x ≤ u }` for `N` the node–arc incidence matrix.

### Conditions for exact equivalence

All must hold:

1. **Single commodity** — one SKU, or several sharing no resource (then they decompose into
   independent flows).
2. **Linear, lane-additive, per-unit costs.** Separable **convex** piecewise-linear arc costs
   are also admissible: replace each arc with parallel arcs, one per segment, capacity =
   segment width, cost = segment slope. Increasing slopes mean the LP fills cheap segments
   first automatically, so the transformation is *exact* (Ahuja, Magnanti & Orlin 1993, Ch. 14).
3. **No fixed charge**, and no cost depending on which set of lanes is used.
4. **Box bounds only.** Integral lower bounds `l ≥ 0` are fine. The semi-continuous
   "0 or at least Q" is not.
5. **Lossless conservation** — arc gains `γ = 1`. Generalised flows with shrinkage or yield
   loss give a generalised network matrix, which is **not TU**; the LP stays polynomial but
   extreme points can be fractional.
6. **Demand known at decision time.** Otherwise reallocation is the *recourse* stage of a
   two-stage stochastic program.
7. **No side constraints** — no capacity shared across SKUs, no cardinality cap, no routing
   (a lane is a direct link, not a tour).
8. For **integral** optima additionally: `b, l, u` integral.

### Published statements

- **Karmarkar & Patel (1977)**, NRLQ 24(4):559–575,
  [10.1002/nav.3800240405](https://doi.org/10.1002/nav.3800240405) — the cleanest early
  statement: the one-period N-location problem **decomposes into a transportation problem plus
  decoupled newsvendor problems**, with optimal policies characterised via the transportation
  dual.
- **Herer, Tzur & Yücesan (2006)**, IIE Transactions 38(3):185–200,
  [10.1080/07408170500434539](https://doi.org/10.1080/07408170500434539).
- **Özdemir, Yücesan & Herer (2006)**, EJOR 175(1):602–621,
  [10.1016/j.ejor.2005.06.004](https://doi.org/10.1016/j.ejor.2005.06.004) — transshipment
  capacity as a capacitated network flow. Single-item, so lane capacities are exactly the
  `l ≤ x ≤ u` that TU accommodates; nothing breaks.
- **Van Mieghem & Rudi (2002)**, *Newsvendor networks*, M&SOM 4(4):313–335,
  [10.1287/msom.4.4.313.5728](https://doi.org/10.1287/msom.4.4.313.5728) — the correct general
  theoretical home.
- **Anupindi, Bassok & Zemel (2001)**, M&SOM 3(4):349–368,
  [10.1287/msom.3.4.349.9973](https://doi.org/10.1287/msom.3.4.349.9973) — LP duality on the
  second stage yields a core allocation for the decentralized game.

## 3. Total unimodularity

**Definition.** `A ∈ ℤ^(m×n)` is totally unimodular iff every square submatrix has determinant
in {0, +1, −1}.

**Theorem (Hoffman & Kruskal, 1956).** `A` is TU **iff** for every integral `b`, the polyhedron
`{x : Ax ≤ b, x ≥ 0}` is integral. *Integral boundary points of convex polyhedra*, in Kuhn &
Tucker (eds.), Linear Inequalities and Related Systems, Annals of Math. Studies 38, Princeton.

Two points routinely elided: this is a **characterisation**, not merely a sufficient condition;
and the quantifier is **for all integral b** — a non-TU matrix can yield an integral polyhedron
for one particular `b`, so observing integrality on an instance is no evidence of TU.

**Theorem.** For a **directed** graph, the node–arc incidence matrix (one +1 and one −1 per
column) is TU. Ahuja, Magnanti & Orlin (1993) Ch. 11; Schrijver (1986) §19.3.

**Caveat.** The **undirected** incidence matrix is TU *iff the graph is bipartite* — an odd
cycle's incidence matrix has determinant ±2. Relevant if lanes are modelled as undirected edges.

**Corollary.** `min { cᵀx : Nx = b, l ≤ x ≤ u }` with integral data admits an integral optimum;
every basic optimum is integral; the LP relaxation is exact.

Recognition is decidable: Seymour (1980), JCTB 28(3):305–359,
[10.1016/0095-8956(80)90075-1](https://doi.org/10.1016/0095-8956(80)90075-1) — every TU matrix
decomposes by k-sums into network matrices, their transposes, and two specific 5×5 matrices,
yielding the only known polynomial TU test.

Min-cost flow itself is strongly polynomial (Tardos 1985; Orlin 1993), and now almost-linear:
Chen, Kyng, Liu, Peng, Probst Gutenberg & Sachdeva (2022, FOCS),
[arXiv:2203.00671](https://arxiv.org/abs/2203.00671).

## 4. The boundary — which realistic constraints break it

| | Extension | TU? | Complexity | Approach |
|---|---|---|---|---|
| a | Shared lane capacity across SKUs | **destroyed** | NP-complete at **2 commodities** | Column generation, branch-and-price |
| b | Fixed transfer cost per lane | **destroyed** | strongly NP-hard | MILP + flow covers, Benders |
| c | Minimum shipment quantity | **destroyed** | strongly NP-complete **and inapproximable** | MILP only; no guarantee exists |
| d | Cardinality cap on transfers | **split** | P if unit shipments, else NP-hard | min-cost flow / MILP |
| e | Concave (volume-discounted) cost | **preserved** | NP-hard anyway | MILP (SOS2), DP on special structure |
| f | Multi-period + inventory carryover | **preserved** | strongly polynomial | plain min-cost flow |

### (a) Shared capacity across SKUs — destroyed

Commodity blocks plus **bundle constraints** `Σ_k x^k_ij ≤ u_ij` give a block-angular matrix.
The cleanest disproof is structural: Hu (1963) shows undirected two-commodity max-flow has an
optimum that is only **half-integral**, which TU with integral data would forbid.

Fractional multicommodity flow is in P (it is an LP). **Integer** multicommodity flow is
**NP-complete with two commodities**, directed and undirected, even at unit capacity — Even,
Itai & Shamir (1976), SIAM J. Comput. 5(4):691–703,
[10.1137/0205048](https://doi.org/10.1137/0205048).

The boundary is unusually sharp. Two commodities are *exactly* half-integral (Hu 1963), and
under the **Euler condition** — even degree-capacity at every vertex — an integral
two-commodity flow attains the maximum in polynomial time (Rothschild & Whinston 1966,
[10.1287/opre.14.3.377](https://doi.org/10.1287/opre.14.3.377)). So: halves are forced, rounding
the halves is the NP-complete part, and parity of the capacity data is the knife edge. At K ≥ 3
even half-integrality fails.

*Inferred (from Geoffrion's integrality property):* because dualising the bundle rows leaves K
min-cost flows, each with an integral polyhedron, the **Lagrangian and Dantzig–Wolfe bounds
equal the arc-formulation LP bound.** Decomposition buys speed and better branching, **not a
tighter root bound.**

Paterson et al. (2011) §5 names multi-item transshipment with shared capacity as an **open
research area**, not a solved one.

### (b) Fixed transfer costs — destroyed

`min Σ c·x + Σ f·y` with `0 ≤ x_ij ≤ u_ij·y_ij`, `y` binary. TU fails twice: the variable-upper-
bound rows carry entries `−u_ij ∉ {0,±1}`; and more importantly the LP charges `f·(x/u)`,
replacing the fixed charge by its **lower convex envelope**, so the bound is arbitrarily weak on
lightly-loaded lanes. This is Fixed-Charge Network Flow, strongly NP-hard — Garey & Johnson
(1979) problem **[ND32]**, p. 214.

**The frontier in this exact domain is published and sharp:**

- **Herer & Tzur (2001)**, NRL 48(5):386–408,
  [10.1002/nav.1025.abs](https://doi.org/10.1002/nav.1025.abs) — **two locations** with fixed
  replenishment *and* fixed transshipment costs: **polynomial time**.
- **Herer & Tzur (2003)**, IIE Transactions 35(5):419–432,
  [10.1080/07408170304389](https://doi.org/10.1080/07408170304389) — the **multi-location**
  version is proved **NP-hard**.

So the fixed-charge complexity frontier sits **between two and three locations.**

Strengthen with flow-cover inequalities: Padberg, Van Roy & Wolsey (1985), Operations Research
33(4):842–861, [10.1287/opre.33.4.842](https://doi.org/10.1287/opre.33.4.842). Use tight
variable upper bounds — the minimum of reachable supply and reachable demand — never a generic
big-M.

*Inferred:* unlike (a), Lagrangian relaxation here **can** beat the LP bound — but only if the
**flow-conservation** rows are dualised. Dualising the VUB rows instead leaves a min-cost flow
plus free binaries, which has the integrality property, collapsing the bound back to the LP.

### (c) Minimum shipment quantity — destroyed, and the worst of the six

`x_ij ∈ {0} ∪ [Q_ij, u_ij]` — semi-continuous. The feasible set is a **union of polyhedra**,
non-convex before integrality even enters.

The relaxation is not merely weak, it is **blind**: relaxing `y ∈ [0,1]`, the constraints
`x/u ≤ y ≤ x/Q` are satisfiable for any `x ∈ [0,u]` whenever `Q ≤ u`, so **the LP relaxation is
exactly ordinary min-cost flow with the minimum-quantity restriction entirely absent.**

Worse than (b) in a second way: there, feasibility is trivial (`y ≡ 1`). Here, finding *any*
feasible flow is itself NP-complete.

**Krumke & Thielen (2011)**, *Minimum cost flows with minimum quantities*, IPL 111(11):533–537,
[10.1016/j.ipl.2011.03.007](https://doi.org/10.1016/j.ipl.2011.03.007): strongly NP-complete,
and **not approximable within any polynomially computable function unless P = NP** — even on
bipartite graphs, even when a feasible solution is guaranteed to exist.

That is categorical. No constant factor, no polylog, no FPTAS. Any claimed approximation ratio
for minimum-quantity flow is, absent P=NP, false.

### (d) Cardinality cap — splits on shipment granularity

**Unit shipments ⇒ TU preserved, in P.** If each lane carries at most one unit, "at most K
transfers" ≡ "total flow ≤ K", enforced by routing all flow through one artificial
super-source→super-sink arc of capacity K. Still a pure network, still TU. This underlies the
k-cardinality assignment problem, polynomially solvable — Dell'Amico & Martello (1997), DAM
76(1–3):103–121, [10.1016/S0166-218X(97)00120-0](https://doi.org/10.1016/S0166-218X(97)00120-0).
*(The re-expression is inferred; the k-cardinality result is published.)*

**General quantities ⇒ destroyed, NP-hard.** Counting *used* lanes needs binaries, giving case
(b) with unit fixed charges plus a budget row. The decision version is precisely Garey & Johnson
**[ND32]** with all edge costs 1. Separately, appending a single generic side row to a TU matrix
destroys TU in general.

Practical reading: a cardinality cap is cheap when transfers are naturally unit-sized and
expensive the moment they are not.

### (e) Concave cost — TU preserved, tractability destroyed

**The constraint matrix is untouched.** The polyhedron is identical to the linear case: still
TU, still integral. Only the objective changes. Since a concave function attains its minimum
over a compact polyhedron at an **extreme point**, and every extreme point is integral, **an
integral optimum is guaranteed to exist.** Concave costs do not cost integrality.

What dies is convexity, hence LP-ness, hence any polynomial optimality certificate. NP-hard:
Murty & Kabadi (1987), Math. Programming 39(2):117–129,
[10.1007/BF02592948](https://doi.org/10.1007/BF02592948); for networks, Guisewite & Pardalos
(1990), Annals of OR 25(1):75–99, [10.1007/BF02283688](https://doi.org/10.1007/BF02283688) —
even single-source uncapacitated is NP-hard. A fixed charge *is* a concave cost, so (b) is a
special case of (e).

**The decisive contrast.** Convex piecewise-linear costs split exactly into parallel arcs
(§2, condition 2). The split fails for concave costs for one reason: **the cheap segment comes
last.** With increasing slopes the LP fills segments in the correct order by itself; with
decreasing slopes it would take the deep-discount segment without first paying for the earlier
volume. That asymmetry is the whole of case (e).

Croxton, Gendron & Magnanti (2003), Management Science 49(9):1268–1273,
[10.1287/mnsc.49.9.1268.16570](https://doi.org/10.1287/mnsc.49.9.1268.16570): the LP relaxations
of the incremental, multiple-choice and convex-combination formulations all approximate the cost
by its lower convex envelope and therefore **share the same bound**. Choosing among them is
about solver behaviour and branching, never bound strength.

Polynomial special cases: Zangwill (1968),
[10.1287/mnsc.14.7.429](https://doi.org/10.1287/mnsc.14.7.429) — single-source/single-sink and
acyclic single-source multi-sink (the structure that makes Wagner–Whitin lot sizing polynomial);
Erickson, Monma & Veinott (1987),
[10.1287/moor.12.4.634](https://doi.org/10.1287/moor.12.4.634) — send-and-split DP, polynomial
for a fixed number of terminals.

### (f) Multi-period with carryover — preserved, and free

Build a node `(i,t)` per location and period. Transshipment arcs `(i,t) → (j,t+τ_ij)` carry
transit time as the offset. **Inventory carryover arcs `(i,t) → (i,t+1)`** carry holding cost and
storage capacity. Replenishment arcs enter from a supply node with the procurement lead time as
offset. Backorder arcs `(i,t+1) → (i,t)` if backlogging is allowed. Demand is a sink at `(i,t)`.

The time-expanded graph is still a directed graph, so its node–arc incidence matrix is still a
node–arc incidence matrix — **still TU**. Deterministic, linear-cost, multi-period reallocation
with carryover, lead times and backlogging remains a pure min-cost flow: strongly polynomial,
guaranteed integral. Ahuja, Magnanti & Orlin (1993) Ch. 19.

**This is the most valuable free extension in the set. Time is not a source of hardness.** The
price is graph size — `|V|·T` nodes, roughly `|A|·T` arcs — not complexity class.

What breaks it: per-period **setup/ordering** costs (economic lot sizing; single-item
uncapacitated stays polynomial via Wagner–Whitin, capacitated and multi-item are NP-hard —
Florian, Lenstra & Rinnooy Kan 1980,
[10.1287/mnsc.26.7.669](https://doi.org/10.1287/mnsc.26.7.669)); any of (a)–(e) applied per
period; and scenario-tree stochastic demand, where nonanticipativity constraints coupling
scenarios are not network rows. *(That last TU claim is inferred — no crisp published theorem
for scenario-coupled transshipment matrices was located.)*

## 5. Coupling replenishment with transshipment

**There is a clean structural result and no clean computational one beyond two locations.**

For periodic review, multi-location, single item, reactive complete pooling, linear costs and
**no fixed ordering cost**: an **order-up-to (base-stock) policy is optimal** for replenishment,
and given post-demand positions the transshipment subproblem is a **min-cost flow**. Proved
independently by Robinson (1990), Operations Research 38(2):278–295,
[10.1287/opre.38.2.278](https://doi.org/10.1287/opre.38.2.278), and Herer, Tzur & Yücesan (2006).

Robinson can find the base-stock level analytically **only for two locations, or N identical
locations**; otherwise Monte Carlo integration.

**Why it is theoretically joint.** The recourse cost is the optimal value of a min-cost flow LP
as a function of its right-hand side, hence **convex piecewise-linear** in the base-stock vector
by LP duality; expectation preserves convexity. The joint problem is a convex stochastic program
with an embedded network LP — exactly Van Mieghem & Rudi's newsvendor-network framework.

**Why it is not computationally clean.** Evaluating expected recourse means integrating a network
LP over the demand distribution, with no closed form beyond N=2 or identical locations. The
literature's actual method is **simulation-based sample-path optimisation** — Herer, Tzur &
Yücesan (2006) and Özdemir et al. (2006) use infinitesimal perturbation analysis for unbiased
gradients of expected cost with respect to S, then stochastic approximation.

### The throughline *(inferred, but it follows deductively)*

> TU ⇒ integral network polyhedron ⇒ LP value function convex in the RHS ⇒ expected recourse
> convex in S ⇒ base-stock optimal.

**Base-stock optimality is downstream of total unimodularity.** Break TU with (b) fixed transfer
costs, (c) minimum quantities, (d2) cardinality caps or (e) concave costs, and the recourse value
function stops being convex in the inventory position — so the standard argument for base-stock
optimality fails. This is exactly what Herer & Rashit (1999) observe: once fixed costs enter, the
optimal policy structure becomes materially more complex and they handle only two locations, one
period. (A different mechanism from the classical fixed-*ordering*-cost route to (s,S) via
K-convexity.)

**Most of the corpus is hierarchical, not joint.** Essentially the whole continuous-review branch
— Axsäter (1990, 2003), Evers, Minner et al., Olsson — **fixes** the ordering policy and optimises
only the transshipment rule. The review's own verdict: *"determining when it is best to perform
the redistribution when ordering decisions are also considered is still not fully understood."*

## 6. Consequences for this project

1. **The buildable core is (f) with linear costs, decomposed per SKU.** Multi-period, lead times,
   holding costs, backlogging, capacitated lanes — all polynomial, all provably optimal, all
   integral without branch-and-bound. That is a genuinely strong system and it is the *easy* case.
2. **Shared truck capacity across SKUs is the first cliff**, and it is NP-complete at two
   commodities. It is also named by the canonical review as an open research area. Treat any
   version of it as a research contribution, not a feature.
3. **Fixed transfer costs are the second cliff, and the most realistic one** — every real transfer
   has a per-shipment charge. Herer & Tzur place the frontier between two and three locations.
4. **Minimum shipment quantities should be avoided or heuristically handled**, and the write-up
   must not claim any approximation guarantee, because none can exist.
5. **Volume discounts are a trap for intuition** — they keep integrality and still make the
   problem NP-hard. Worth stating explicitly, because the natural assumption is the reverse.
6. **Breaking TU costs more than polynomial time — it costs base-stock optimality**, and with it
   the justification for the replenishment policy. The two halves of the system are coupled
   through the same theorem.
