# Survey: existing open-source inventory systems

Verified against the GitHub API and the projects' own source and `LICENSE` files on
2026-09-16. Licences were read from the repositories rather than taken from GitHub's
inferred label, which is wrong or `NOASSERTION` for several of the most relevant projects.

The question this survey answers: is there a permissively licensed, maintained,
multi-location inventory system worth forking as a baseline, so that effort goes into the
optimization layer instead of into CRUD?

## Answer: no — and the reason matters more than the answer

The blunt version holds. The mature systems are Python and PHP monoliths; the JavaScript
ones are e-commerce engines that model stock as a mutable quantity column, or tutorial
builds. The field is thinner than it looks:

| GitHub search | Repos returned |
|---|---|
| `inventory management nextjs stars:>150` | 0 |
| `warehouse management language:TypeScript stars:>150` | 0 |
| `inventory supabase stars:>60` | 0 |
| `inventory nextjs stars:>25` | 10, of which the top is AGPL and dead since 2024 |

But the decisive reason not to fork is not licensing or quality. It is that **the baseline
and the differentiator are coupled.** An optimizer needs movement history, per-location
cost basis, and lead-time data as inputs. The permissively licensed JavaScript candidates
record none of those. Forking one and adding optimization on top is impossible, because the
data the optimizer reads would not exist.

## Licence findings

Three findings contradict what is widely believed, two of them recently enough that most
secondary sources are stale.

**Vendure is GPL-3.0, not MIT.** It relicensed on 2024-07-17 (PR #2946), MIT to GPL-3.0
plus a commercial licence. A plugin exception added 2024-12-30 permits distributing
*plugins* under other terms, but the core is copyleft.

**Carbon (`crbnos/carbon`) is AGPL-3.0 with an open-core carve-out**, and its LICENSE goes
beyond stock AGPL: `packages/ee` and any `*.ee.*` file require a paid licence, and internal
production use is prohibited unless modifications are open-sourced. This is the painful one,
because architecturally it is precisely the target — TypeScript, Supabase, SQL migrations,
RLS, multi-location, both `warehouseTransfer` and `stockTransfer`, an append-only
`itemLedger`, and a separate `costLedger`. **Design reference; not a fork candidate.**

**Medusa is genuinely MIT** for everything relevant. Its `ENTERPRISE-LICENSE.md` reserves
rights over "Enterprise Materials", but no file currently matches `packages/ee` or `*.ee.*`,
and the notice is explicitly non-retroactive.

## The field

"Ledger" means append-only movement rows rather than a mutable quantity column. "Cost" means
real cost layers or valuation, not a single price field.

| Project | ★ | Licence | Stack | Multi-loc | Transfers | Ledger | Cost |
|---|---|---|---|---|---|---|---|
| Odoo CE | 54,380 | LGPL-3.0 | Python | Yes (tree) | Yes + transit | Yes | FIFO/AVCO/std |
| ERPNext | 39,254 | GPL-3.0 | Python/Frappe | Yes (tree) | Yes | Yes | FIFO + moving avg |
| Medusa | 36,320 | MIT | TS/Node | Yes | No | No | No |
| Saleor | 23,334 | BSD-3 | Python | Yes | No | No | No |
| Spree | 15,697 | BSD-3 | Ruby | Yes | Yes | Yes | Partial |
| Vendure | 8,440 | GPL-3.0 | TS/NestJS | Yes | No | Yes | No |
| Dolibarr | 7,615 | GPL-3.0 | PHP | Yes | Yes | Yes | Weighted avg |
| InvenTree | 7,574 | MIT | Python/Django | Yes (MPTT) | Yes | Yes | Specific-ID only |
| GreaterWMS | 4,377 | Apache-2.0 | Django + Vue | Yes | Yes | Yes | Weak |
| Carbon | 2,407 | AGPL + EE | TS + **Supabase** | Yes | Yes (2 kinds) | Yes | Yes |
| Open Mercato | 1,735 | **MIT** | **Next.js/TS** | Yes | Partial | Yes | **No** |
| OpenBoxes | 896 | EPL-1.0 | Groovy/Grails | Yes | Yes | Yes | Partial |

Dead or unusable: `ed-roh/inventory-management` (442★) is the top TypeScript result for
"inventory-management" and has **no licence file at all**, making it legally unusable;
`medusajs/nextjs-starter-medusa` (2,794★) is archived; `vercel/commerce` (14,259★) is a
Shopify storefront holding no inventory state.

**Open Mercato** is the only literal answer to the question — MIT, Next.js, TypeScript,
actively committed, with `wms_inventory_movements` carrying an idempotency-key unique index.
Its disqualifier is structural rather than legal: it has no cost accounting of any kind, and
its reorder points carry a unique index per product/variant, making them **global per SKU
rather than per location** — the wrong shape for a multi-store problem, and the exact field
this project must vary.

## The canonical data model

The design converges across ERPNext, Odoo, Dynamics Business Central and Carbon. Carbon's
`itemLedgerType` enum reproduces Business Central's item ledger almost verbatim.

- **`item` / `product_variant`** — the stockable unit. Medusa's split between the sellable
  `product_variant` and the stockable `inventory_item` is the cleaner separation: the thing
  sold and the thing counted are not the same object.
- **`location`** — self-referencing tree, not a flat list. Odoo's tree includes *virtual*
  locations (Vendor, Customer, Inventory Loss, Transit), which is how it models stock that
  belongs to no real place.
- **`stock_movement`** — append-only. **The system of record.**
- **`stock_level`** — the *derived cache* of on-hand per (item, location). ERPNext calls it
  `Bin`, Odoo `stock.quant`. Not the truth; a materialized aggregate.
- **`reservation`** — soft holds, so `available = on_hand − reserved`.
- **`transfer_order`** — the one most systems lack. The mature shape carries **separate
  shipped and received quantities and an in-transit state**, because stock in a truck belongs
  to neither location. Odoo models this with a virtual transit location, so one transfer is
  two moves and in-transit stock stays on the balance sheet.
- **`cost_layer`** — see below.

### Why append-only, and not a quantity column

Auditability is the least of it.

1. **Concurrency.** `UPDATE stock SET qty = qty - 1` serializes on a row and invites lost
   updates and deadlocks. Inserting a signed delta does not contend.
2. **Backdating.** Real operations post yesterday's receipt tomorrow. A mutable column
   cannot answer "what did we hold on the 31st?" ERPNext stores `qty_after_transaction` and
   `posting_datetime` on every entry precisely so it can recompute forward.
3. **Cost depends on sequence.** FIFO is meaningless without an ordered history, and cost
   layers cannot be retrofitted onto a quantity column later. This is the single most
   expensive mistake to correct.
4. **It is the optimizer's input.** Demand history, lead-time variance and service levels are
   all derived from movement rows. A system holding only current quantities has already
   destroyed the data this project depends on.

Mature systems enforce immutability: corrections are new compensating rows, never edits.
Spree and Solidus both literally override `def readonly?` on `StockMovement`.

## Cost accounting, and what a transfer must do

**An internal transfer must not create or destroy value.** The issue consumes source layers
at *their* cost; the receipt creates a layer at *exactly that* cost — not at the
destination's average, not at list price. Net change in company-wide inventory value is zero,
excluding explicitly capitalized freight. The destination's moving-average rate therefore
shifts although no money was spent, which is correct and routinely surprises people.

ERPNext, Odoo, Carbon and Dolibarr implement this. **Medusa, Vendure, Saleor, Solidus, OSPOS,
Part-DB, Snipe-IT, Open Mercato and Bagisto track quantity only** — every JavaScript and
TypeScript option surveyed except AGPL-licensed Carbon.

The trap: implementing transfers as "decrement A, increment B" on quantity columns and adding
costing later **silently creates or destroys inventory value on every transfer** — on exactly
the operation this project optimizes. It is also what makes the objective function meaningful,
since a transshipment is only economically sound if the cost of the move can be priced against
the margin of the averted stockout.

## The gap

Searched for directly. `transshipment` in `odoo/odoo`: **0 hits**. Across GitHub the term
appears in maritime logistics, in academic MIP solvers, and in bibliographies — it exists in
the operations-research literature and in nobody's shipping product.

Replenishment is barely better. `reorder_point` in Saleor: 0. In Medusa: 0. `reorderPoint` in
Vendure: 0. InvenTree has `minimum_stock` and zero occurrences of `reorder`, `safety`,
`forecast` or `transship`. Open Mercato has a `reorder_point` column and no code that reads it.

Where replenishment does exist it is hand-entered min/max evaluated by a cron job:

- **ERPNext** — `Item Reorder` has five fields; the scheduled job compares projected quantity
  to a typed threshold and files a request. It *can* file an inter-warehouse transfer, but a
  human picks the source.
- **Odoo** — `stock.warehouse.orderpoint` is min/max, and inter-warehouse resupply is a
  **static route** (`resupply_wh_ids`) configured in advance. A fixed hierarchy, not a decision.
- **Carbon** — the most sophisticated, with four policies and demand-accumulation periods.
  Safety stock is still a number typed in, not computed from demand variance and a service level.
- **OpenBoxes** — carries the right vocabulary (`forecastQuantity`, `expectedLeadTimeDays`,
  `abcClass`). But `ForecastingService.groovy` does no forecasting: its methods are SQL
  aggregations of historical issues, with zero occurrences of `exponential`, `regression`,
  `seasonal` or `stddev`. Without a standard deviation it cannot compute safety stock from
  variability even in principle.

**No system surveyed — permissive or copyleft, in any language — computes which location
should ship how much to which other location, from current imbalance, demand forecast, lead
times, and holding-versus-stockout economics.** There is no solver, no LP or MIP formulation,
no newsvendor model, and no service-level-driven safety stock anywhere in the surveyed code.

## Consequences for this project

1. **Do not fork.** Build the schema fresh, using Carbon's `itemLedger`/`costLedger` and
   ERPNext's `Stock Ledger Entry` as design references. The CRUD avoided by forking is a small
   fraction of the work, and every candidate with a good schema is licensed in a way this
   project has ruled out.
2. **The append-only ledger with cost layers is day-one work**, not a later refinement. It is
   the optimizer's input and it cannot be retrofitted.
3. **`stock_level` is a cache**, maintained by trigger or `SECURITY DEFINER` RPC in the same
   transaction as the ledger insert. Summing the movement table on read is the first
   performance cliff; Carbon's migration names (`item-ledger-snapshot`,
   `ledger-performance-indexes`) confirm the path. The ledger should be insert-only under RLS.
4. **The gap is real, and it is an absence of optimization rather than an absence of software.**
   Claims about it should be phrased that way.
