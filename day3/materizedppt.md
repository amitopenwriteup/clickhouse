# ClickHouse Materialized Views — Slide-by-Slide Notes

A written companion to the deck `ClickHouse_Materialized_Views.pptx`. Each section below matches one slide.

---

## Slide 1 — Title

**ClickHouse Materialized Views: What they do, and the three types you'll actually use**

Sets up the scope of the deck: three MV types — **Incremental**, **Refreshable**, and **Cascading** — all illustrated using one running example (an e-commerce `orders` stream).

---

## Slide 2 — What is a Materialized View?

**Definition:** A materialized view (MV) is a query whose results are *pre-computed and stored* in a real table, instead of being re-run every time you ask for them.

**Key properties:**
- It sits between a **source table** and a **target table**.
- It runs **automatically** — you never trigger it by hand.
- You query the **target table** directly, and it's already up to date.

**Diagram:** `Source table (orders) → Materialized View → Target table`

This is the shape every MV type shares. What differs between the three types is *when* the view runs and *how much* of the query it re-executes.

---

## Slide 3 — Part A: Incremental Materialized View

**What it does:** Fires on every insert into the source table. No polling, no full re-scan — only the newly inserted block is processed.

**Diagram:** `orders (new row inserted) → daily_revenue_mv (GROUP BY day, category) → daily_revenue (SummingMergeTree)`

**Key points:**
- The target table's engine (`SummingMergeTree`) merges partial sums itself.
- The `GROUP BY` columns in the MV must match the target table's `ORDER BY`.
- Best for: real-time, single-table rollups.

**Gotcha:** The MV reacts to *when* a row is inserted, not its timestamp. Rows already sitting in `orders` before the MV was created are never picked up — unless the MV was created with `POPULATE`.

---

## Slide 4 — Part B: Refreshable Materialized View

**What it does:** Re-runs the *entire* query on a fixed schedule (`REFRESH EVERY ...`), rather than reacting to individual inserts. This makes it a good fit for JOINs, where a single row's contribution can depend on the whole dataset.

**Diagram:** `orders + customers → top_customers_mv (LEFT JOIN + GROUP BY, on a timer) → top_customers (overwritten each run)`

**Key points:**
- Does **not** react to individual inserts — freshness depends entirely on the schedule.
- The schedule is adjustable: `ALTER TABLE ... MODIFY REFRESH EVERY 5 MINUTE`.
- `APPEND` mode keeps a growing history of snapshots instead of overwriting the latest result.
- Best for: complex JOINs and periodic, batch-style reports.

**Check status anytime:** `SELECT ... FROM system.view_refreshes`

---

## Slide 5 — Part C: Cascading Materialized Views

**What it does:** Chains MVs together so one MV's *target* table becomes the next MV's *source* table — enabling multi-level rollups without ever re-touching the raw data.

**Diagram:** `orders → daily_revenue (SummingMergeTree) —sumState()→ monthly_revenue (AggregatingMergeTree) —sumMerge()→ yearly_revenue (SummingMergeTree)`

**Key points:**
- A single insert into `orders` flows through all three MVs automatically.
- `daily_revenue_mv` writes to `daily_revenue`, which triggers `monthly_revenue_mv`, which in turn triggers `yearly_revenue_mv`.
- Each stage forwards only the newly computed block — not the fully merged table.

**Gotcha:** An `AggregatingMergeTree` stage stores partial aggregate *states*, not final numbers. Downstream views must merge those states with `sumMerge()` before re-aggregating.

---

## Slide 6 — Three Types, Side by Side

| | Incremental | Refreshable | Cascading |
|---|---|---|---|
| **Trigger** | Every insert into source | Fixed timer (`REFRESH EVERY`) | Insert into an upstream MV's target |
| **Query re-run** | Only the new block | Entire query, every cycle | Only the new block, per stage |
| **Good for** | Single-table rollups | JOINs, batch-style reports | Multi-level rollups (day→month→year) |
| **Freshness** | Real-time | As fresh as the schedule | Real-time, cascades downstream |
| **Watch out for** | `POPULATE` needed for old rows | Left-most table only triggers a JOIN | Needs `xState`/`xMerge` across stages |

---

## Slide 7 — Key Takeaways

- A materialized view is **stored, automatic query output** — not a live view re-evaluated on read.
- **Incremental MVs** are cheap and real-time, but only ever see rows inserted *after* they were created.
- **Refreshable MVs** trade real-time freshness for full JOIN support on a predictable schedule.
- **Cascading MVs** let you build daily → monthly → yearly rollups without ever re-touching raw data.
