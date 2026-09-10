Here's the same explanation, in English.

## Slide 1: Title
"Today we'll understand how ClickHouse optimizes **filtering, aggregation, and sorting** — and how the entire query pipeline works."

## Slide 2: The Query Pipeline
"Whenever a query runs, it always goes through the same 4 steps, in this order:

- **READ** → data is first read from disk
- **FILTER** → the WHERE clause narrows down to matching rows
- **AGGREGATE** → GROUP BY, SUM, AVG etc. are applied
- **SORT** → finally, ORDER BY sorts the result

The most important point: **the less data that survives the FILTER stage, the faster AGGREGATE and SORT will run.**"

## Slide 3: Reading EXPLAIN — Bottom to Top
"The `EXPLAIN` output is printed top to bottom, but it actually executes **bottom to top**. On the left, you can see the real output — read it from the bottom: `ReadFromMergeTree` runs first, then `Filter`, then `Aggregating`, and finally `Sorting` at the very end. On the right, the same thing is shown as simple numbered steps 1-2-3-4."

## Slide 4: Optimizing FILTER
"This slide shows the difference with and without a **primary key**.

- Without a primary key, any filter you apply forces ClickHouse to **scan the entire table**
- But when the table has a primary key like `ORDER BY (pickup_date, ...)`, ClickHouse's **sparse index** can find only the relevant granules

On the right, the `EXPLAIN` output shows **'Granules: 5/244'** — meaning only 5 out of 244 granules were read. That's the real optimization — irrelevant data gets skipped before it's even read."

## Slide 5: Optimizing AGGREGATE
"Aggregation itself doesn't get faster — it just gets **less data** to work with, which is why it feels faster.

The flow is: all 2 million rows exist first → after FILTER, only the matching rows remain → then GROUP BY runs only on those remaining rows, not the entire table."

## Slide 6: Optimizing SORT
"Sorting always happens at the **very end** of the pipeline — by that point, filtering and aggregation have already shrunk the data significantly.

There are two ways SORT becomes cheaper:
1. It only has to sort the small amount of data left over
2. If you add `LIMIT`, ClickHouse tracks only the top-N values instead of sorting the entire list — this is called a **partial sort**"

## Slide 7: Key Takeaways
"Finally, five key points to remember:

1. The pipeline always runs in this order — READ → FILTER → AGGREGATE → SORT
2. Always read the EXPLAIN output bottom to top
3. The more data FILTER removes, the faster everything else becomes
4. Choosing the right PRIMARY KEY makes FILTER itself faster
5. SORT runs last, so it's usually cheap — the real problem is often in FILTER or AGGREGATE, not SORT"
