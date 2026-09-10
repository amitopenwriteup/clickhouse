# ClickHouse Window Functions: Complete Slide-by-Slide Guide

## Overview
This guide walks through the ClickHouse Window Functions presentation (12 slides) with detailed explanations, code examples, diagrams, and learning flow. Each section corresponds to one slide and builds upon previous concepts.

---

## **SLIDE 1: Title Slide**

### Content
- **Title**: ClickHouse Window Functions
- **Subtitle**: Hands-On Lab: Computing Over Partitions

### Purpose
Sets the stage for learning a powerful analytical tool. Window functions enable complex analytics without the data loss that GROUP BY causes.

### Learning Objective
Participants will understand that window functions are the bridge between simple aggregation (GROUP BY) and advanced analytics.

---

## **SLIDE 2: What are Window Functions?**

### Content
**Core Concept**: A window function computes a value ACROSS a set of related rows (the "window") but KEEPS every original row

### Key Principle
```
Window Functions ≠ GROUP BY

GROUP BY:      Collapses rows → loses detail
Window Func:   Adds context → keeps detail
```

### Example
```sql
SELECT 
    region, 
    salesperson, 
    amount,
    sum(amount) OVER (PARTITION BY region) AS region_total
FROM sales;
```

**What happens:**
- Input: 8 sales records
- Output: Still 8 records, but each now includes `region_total`
- Result keeps individual transaction detail while adding aggregate context

### Why This Matters
Real-world analytics often require both:
- Individual transaction data (amount)
- Context for comparison (region total, running total, rank)

GROUP BY gives you only the context (loses transactions).
Window functions give you both.

### Syntax Anatomy
```sql
<window_function>(...) OVER (
    PARTITION BY <column(s)>   -- optional: split into groups
    ORDER BY <column(s)>       -- optional: defines row order
    ROWS/RANGE BETWEEN ...     -- optional: defines the frame
)
```

**Each part serves a purpose:**
- `PARTITION BY`: Define which rows belong together
- `ORDER BY`: Define order within the partition (affects frame)
- `ROWS BETWEEN`: Explicitly control which rows are included

### Sample Data Used
```
region | salesperson | sale_date  | amount
East   | Alice       | 2026-01-01 | 100
East   | Alice       | 2026-01-02 | 150
East   | Bob         | 2026-01-01 | 200
East   | Bob         | 2026-01-03 | 50
West   | Carol       | 2026-01-01 | 300
West   | Carol       | 2026-01-02 | 120
West   | Dave        | 2026-01-01 | 90
West   | Dave        | 2026-01-02 | 60
```

### Flow Continuation
Now we understand the concept. How is this different from GROUP BY? → SLIDE 3 shows the comparison.

---

## **SLIDE 3: Window Functions vs GROUP BY**

### Content
**Direct Comparison** between the two approaches

### GROUP BY Approach

```sql
SELECT region, sum(amount) AS total
FROM sales
GROUP BY region;
```

**Result:**
```
region | total
East   | 400
West   | 570
```

**Characteristics:**
- ✓ Collapsed to 2 rows (one per group)
- ✗ Individual sales lost
- ✓ Fast aggregation
- ✗ Can't answer "what % of region is this sale?"

### Window Function Approach

```sql
SELECT 
    region, 
    amount,
    sum(amount) OVER (PARTITION BY region) AS region_total
FROM sales;
```

**Result:**
```
region | amount | region_total
East   | 100    | 400
East   | 150    | 400
East   | 200    | 400
East   | 50     | 400
West   | 300    | 570
West   | 120    | 570
West   | 90     | 570
West   | 60     | 570
```

**Characteristics:**
- ✓ All 8 rows preserved
- ✓ Individual sales visible
- ✓ Each sale has context
- ✓ Can answer "what % of region is this sale?"

### Comparison Table

| Aspect | GROUP BY | Window Function |
|--------|----------|-----------------|
| **Rows** | Collapsed (2) | Preserved (8) |
| **Detail** | Lost | Kept |
| **Row Context** | No | Yes |
| **Use Case** | Simple totals | Rank, compare, ratios, running totals |
| **Example Output** | 2 rows per region | 1 row per sale + context |

### When to Use Which

**Use GROUP BY when:**
- You only need summary data
- Detail is not needed
- Simple aggregates (total revenue per region)

**Use Window Functions when:**
- You need detail AND summary
- Calculating percentages or ratios
- Ranking or comparing rows
- Running totals or moving averages

### Key Decision Tree
```
Do you need individual row detail?
├─ YES → Window Function
└─ NO  → GROUP BY (simpler, faster)

Do you need aggregate + row together?
├─ YES → Window Function  
└─ NO  → GROUP BY
```

### Flow Continuation
Now we understand the difference. But window functions have a special PARTITION BY that's NOT like GROUP BY. → SLIDE 4 explains this critical distinction.

---

## **SLIDE 4: Understanding PARTITION BY in Window Functions**

### Content
**Critical Insight**: PARTITION BY in window functions ≠ GROUP BY

### What PARTITION BY Does

```
PARTITION BY = Logical grouping WITHOUT collapsing rows
```

It's like saying: "These rows are related, calculate context for them, but keep them all!"

### Visual Explanation

**Input Data:**
```
region | salesperson | amount
East   | Alice       | 100
East   | Alice       | 150
East   | Bob         | 200
West   | Carol       | 300
```

**Query:**
```sql
SELECT region, amount,
       sum(amount) OVER (PARTITION BY region) AS region_total
FROM sales;
```

**Mental Model:**
```
ClickHouse thinks:
"Create logical partitions by region"

[EAST Partition]
- Alice, 100
- Alice, 150  
- Bob, 200
- Total: 450 (shown for each row)

[WEST Partition]
- Carol, 300
- Total: 300 (shown for each row)

"Now calculate region_total for each partition"
"But keep all rows!"
```

**Result:**
```
region | amount | region_total
East   | 100    | 450  ← Same total for all East rows
East   | 150    | 450  ← But kept all individual rows
East   | 200    | 450  ← Not collapsed!
West   | 300    | 300
```

### The Key Difference

```
╔═══════════════════════════════════════════════════════╗
║ PARTITION BY (Window Function)                       ║
├───────────────────────────────────────────────────────┤
║ • Groups rows logically                              ║
║ • Calculates aggregate per group                     ║
║ • KEEPS ALL ROWS (no collapse)                       ║
║ • Each row shows its group's aggregate               ║
╚═══════════════════════════════════════════════════════╝

╔═══════════════════════════════════════════════════════╗
║ GROUP BY                                              ║
├───────────────────────────────────────────────────────┤
║ • Groups rows                                         ║
║ • Calculates aggregate per group                     ║
║ • COLLAPSES to one row per group                     ║
║ • Detail is completely lost                          ║
╚═══════════════════════════════════════════════════════╝
```

### Why This Matters

With PARTITION BY, you can:
- ✓ See individual sale ($100) AND region total ($450)
- ✓ Calculate percentage: $100 / $450 = 22%
- ✓ Compare each sale to context
- ✓ Keep transaction-level audit trail

With GROUP BY, you can:
- ✓ See only region total ($450)
- ✗ Can't calculate $100's percentage
- ✗ Can't compare individual sales

### Flow Continuation
Now we understand PARTITION BY. But there's another important "ORDER BY" in ClickHouse: the MergeTree ORDER BY. How do they differ? → SLIDE 5 clarifies this confusion.

---

## **SLIDE 5: MergeTree ORDER BY vs Window Function ORDER BY**

### Content
**Common Confusion**: These are TWO DIFFERENT things!

### MergeTree ORDER BY (Physical Storage)

```sql
CREATE TABLE sales (
    region String,
    salesperson String,
    sale_date Date,
    amount UInt32
)
ENGINE = MergeTree
ORDER BY (region, sale_date);  ← Physical storage sort
```

**What it does:**
- Physically sorts data on disk by (region, then sale_date)
- Optimizes storage layout
- Speeds up queries that filter by these columns
- One-time cost at table creation

**Example:**
```
Disk layout (already sorted):
East | Alice | 2026-01-01 | 100
East | Alice | 2026-01-02 | 150
East | Bob   | 2026-01-01 | 200
East | Bob   | 2026-01-03 | 50
West | Carol | 2026-01-01 | 300
...
```

**Use for:** Query performance optimization

### Window Function ORDER BY (Logical Ordering)

```sql
sum(amount) OVER (
    PARTITION BY region
    ORDER BY sale_date  ← Logical row order within window
)
```

**What it does:**
- Defines logical order of rows within each partition
- Determines the default window frame
- Enables running totals, lag/lead, etc.
- Does NOT change how data is stored

**Example Effect:**
```
For East partition:
PARTITION BY region (East group)
ORDER BY sale_date (logical order)

Order matters for:
- Running totals (need sequence)
- lag()/lead() (need neighbors)
- Window frame (depends on order)
```

**Use for:** Analytics calculations

### Key Comparison Table

| Aspect | MergeTree ORDER BY | Window ORDER BY |
|--------|-------------------|-----------------|
| **When Applied** | At table creation | In query |
| **What It Affects** | Physical disk layout | Logical row ordering |
| **Purpose** | Storage optimization | Analytics logic |
| **Scope** | Entire table | One window partition |
| **Changeable** | No (static) | Yes (per query) |
| **Example** | `ORDER BY (region, date)` | `ORDER BY sale_date` |

### Why You Need Both

**Scenario: Monthly Sales Report**

```sql
CREATE TABLE sales (...)
ENGINE = MergeTree
ORDER BY (region, sale_date);  -- Storage optimization

SELECT
    region, 
    salesperson, 
    sale_date,
    amount,
    sum(amount) OVER (
        PARTITION BY region
        ORDER BY sale_date  -- Analytics logic
    ) AS running_total
FROM sales;
```

**MergeTree ORDER BY:**
- Makes the query fast (data already sorted on disk)

**Window ORDER BY:**
- Makes running_total calculation correct (defines row sequence)

### Common Misconception

```
❌ WRONG: "MergeTree ORDER BY and Window ORDER BY are the same"
✓ RIGHT: MergeTree ORDER BY = storage, Window ORDER BY = logic
```

### Flow Continuation
Now we understand the foundation. Let's explore ranking functions, the first major window function type. → SLIDE 6.

---

## **SLIDE 6: Ranking Functions**

### Content
**Three Ranking Functions**: row_number(), rank(), dense_rank()

### When They Matter

**Scenario:** Within each region, rank salespeople by their highest single sale amount.

**Query:**
```sql
SELECT 
    region,
    salesperson,
    amount,
    row_number()  OVER (PARTITION BY region ORDER BY amount DESC) AS rn,
    rank()        OVER (PARTITION BY region ORDER BY amount DESC) AS rnk,
    dense_rank()  OVER (PARTITION BY region ORDER BY amount DESC) AS drnk
FROM sales
ORDER BY region, amount DESC;
```

### How They Differ

**Sample Results (East region, ordered by amount descending):**

```
salesperson | amount | row_number | rank | dense_rank
Bob         | 200    | 1          | 1    | 1
Alice       | 150    | 2          | 2    | 2
Alice       | 100    | 3          | 3    | 3
Bob         | 50     | 4          | 4    | 4
```

**No ties in this example, so they're identical.**

### When Ties Occur

If two salespersons have the SAME amount:

```
salesperson | amount | row_number | rank | dense_rank
Bob         | 200    | 1          | 1    | 1
Alice       | 200    | 2          | 1    | 1  ← Tie!
Alice       | 150    | 3          | 3    | 2
Bob         | 50     | 4          | 4    | 3
```

### The Three Differences Explained

#### row_number()
```
Behavior: Always gives unique sequential numbers (1, 2, 3...)
Even for ties: Numbers continue (no gaps)

rule 1: No row gets the same number
rule 2: Continue sequentially regardless of value

Example: 1, 2, 3, 4 (even if rows 1-2 were tied)
```

#### rank()
```
Behavior: Gives same rank to ties, then SKIPS numbers
For ties: Next rank number skips

rule 1: Tied rows get same rank
rule 2: Numbers after tie are skipped

Example: 1, 1, 3, 4 (skips 2 because rows 1-2 were tied)
```

#### dense_rank()
```
Behavior: Gives same rank to ties, but NO gaps
For ties: Next rank continues without skip

rule 1: Tied rows get same rank
rule 2: Numbers continue without gaps

Example: 1, 1, 2, 3 (no skips, but tied rows still get same number)
```

### Which to Use

**Use row_number() when:**
- You need unique positions (get top-1 per group)
- Order among ties doesn't matter
- Example: `WHERE row_number() = 1` to get first

**Use rank() when:**
- Gaps matter semantically (think sports: 1st place, 1st place, 3rd place)
- You want to show that positions were skipped due to ties

**Use dense_rank() when:**
- Ties should be same rank, but no gaps
- Counting rank positions (medal counts)

### Practical Example: Get Top Salesperson Per Region

```sql
SELECT * FROM (
    SELECT 
        region, 
        salesperson, 
        amount,
        row_number() OVER (PARTITION BY region ORDER BY amount DESC) AS rn
    FROM sales
)
WHERE rn = 1;
```

**Result:**
```
region | salesperson | amount | rn
East   | Bob         | 200    | 1
West   | Carol       | 300    | 1
```

### Flow Continuation
Ranking is one use case. Another key use is running totals, which requires understanding how ORDER BY changes the default frame. → SLIDE 7.

---

## **SLIDE 7: Running/Cumulative Totals**

### Content
**The Most Critical Concept**: How ORDER BY changes the default window frame

### The Two Scenarios

#### WITHOUT ORDER BY

```sql
sum(amount) OVER (PARTITION BY region)
```

**Default frame behavior:**
```
Frame = ENTIRE PARTITION
```

**Execution:**
```
For East region partition:
  Sum ALL amounts in East
  = 100 + 150 + 200 + 50 = 450
  
Apply to EACH row:
  Row 1 (100): 450
  Row 2 (150): 450
  Row 3 (200): 450
  Row 4 (50): 450
```

**Characteristic:** Every row shows the SAME total

#### WITH ORDER BY

```sql
sum(amount) OVER (
    PARTITION BY region
    ORDER BY sale_date
)
```

**Default frame behavior:**
```
Frame = UNBOUNDED PRECEDING AND CURRENT ROW
(Everything from partition start up to and including current row)
```

**Execution:**
```
For East region partition (ordered by sale_date):
  Row 1 (2026-01-01, 100):    100           ← Just this row
  Row 2 (2026-01-02, 150):    100+150=250   ← Rows 1-2
  Row 3 (2026-01-03, 200):    100+150+200=450 ← Rows 1-3
  Row 4 (later, 50):          100+150+200+50=500 ← All rows
```

**Characteristic:** Each row shows running total up to that point

### Visual Comparison

```
WITHOUT ORDER BY:
┌────────────────────────────────────┐
│ Entire East Partition (450)         │
├────────────────────────────────────┤
│ 100 | 450 (whole partition)         │
│ 150 | 450 (whole partition)         │
│ 200 | 450 (whole partition)         │
│ 50  | 450 (whole partition)         │
└────────────────────────────────────┘

WITH ORDER BY (date):
┌────────────────────────────────────┐
│ East Partition (start to current)   │
├────────────────────────────────────┤
│ 100 | 100 (just this row)           │
│ 150 | 250 (rows 1-2)                │
│ 200 | 450 (rows 1-3)                │
│ 50  | 500 (rows 1-4)                │
└────────────────────────────────────┘
```

### Complete Example

**Query:**
```sql
SELECT
    region,
    salesperson,
    sale_date,
    amount,
    sum(amount) OVER (PARTITION BY region ORDER BY sale_date) AS running_total
FROM sales
ORDER BY region, sale_date;
```

**Output:**
```
region | salesperson | date       | amount | running_total
East   | Alice       | 2026-01-01 | 100    | 100
East   | Alice       | 2026-01-02 | 150    | 250
East   | Bob         | 2026-01-03 | 200    | 450
East   | Bob         | 2026-01-04 | 50     | 500
West   | Carol       | 2026-01-01 | 300    | 300 ← New partition
West   | Carol       | 2026-01-02 | 120    | 420
West   | Dave        | 2026-01-03 | 90     | 510
West   | Dave        | 2026-01-04 | 60     | 570
```

### Key Insight
```
╔═══════════════════════════════════════════════════════╗
║ This is the MOST IMPORTANT concept in window funcs:  ║
║                                                        ║
║ ORDER BY CHANGES the default frame!                   ║
║                                                        ║
║ No ORDER BY  → Whole partition aggregate              ║
║ With ORDER BY → Running/cumulative aggregate          ║
╚═══════════════════════════════════════════════════════╝
```

### Other Running Aggregates

```sql
-- Running average
avg(amount) OVER (PARTITION BY region ORDER BY sale_date)

-- Running count
count() OVER (PARTITION BY region ORDER BY sale_date)

-- Running maximum
max(amount) OVER (PARTITION BY region ORDER BY sale_date)

-- Running minimum
min(amount) OVER (PARTITION BY region ORDER BY sale_date)
```

### Flow Continuation
Now we can do running totals. What about comparing each row to its neighbors? → SLIDE 8 shows lag() and lead().

---

## **SLIDE 8: lag() and lead() - Accessing Neighbor Rows**

### Content
**Purpose**: Compare a row to its neighbors without a self-join

### lag() - Look Backward

```sql
lag(amount, 1) OVER (
    PARTITION BY salesperson
    ORDER BY sale_date
)
```

**Meaning:**
- Look back N rows (N=1 means previous row)
- Within the same partition (same salesperson)
- In order (by date)

**Behavior:**
- First row in partition: NULL (no previous row)
- All other rows: value from N rows back

### lead() - Look Forward

```sql
lead(amount, 1) OVER (
    PARTITION BY salesperson
    ORDER BY sale_date
)
```

**Meaning:**
- Look forward N rows (N=1 means next row)
- Within the same partition
- In order

**Behavior:**
- Last row in partition: NULL (no next row)
- All other rows: value from N rows ahead

### Example: Alice's Sales

**Data:**
```
salesperson | date       | amount
Alice       | 2026-01-01 | 100
Alice       | 2026-01-02 | 150
(end of Alice's sales)
```

**Query:**
```sql
SELECT
    salesperson,
    date,
    amount,
    lag(amount, 1) OVER (PARTITION BY salesperson ORDER BY date) AS prev_amount,
    lead(amount, 1) OVER (PARTITION BY salesperson ORDER BY date) AS next_amount,
    amount - lag(amount, 1) OVER (PARTITION BY salesperson ORDER BY date) AS change_from_prev
FROM sales;
```

**Result:**
```
salesperson | date       | amount | prev | next | change
Alice       | 2026-01-01 | 100    | NULL | 150  | NULL
Alice       | 2026-01-02 | 150    | 100  | NULL | 50
```

### Practical Use Cases

**1. Detect Trends**
```sql
SELECT
    salesperson,
    sale_date,
    amount,
    amount - lag(amount) OVER (PARTITION BY salesperson ORDER BY sale_date) AS change
FROM sales
WHERE amount - lag(amount) OVER (...) < 0;  -- Decreasing sales
```

**2. Compare to Yesterday**
```sql
SELECT
    date,
    revenue,
    lag(revenue) OVER (ORDER BY date) AS prev_day_revenue,
    (revenue - lag(revenue) OVER (ORDER BY date)) * 100.0 / lag(revenue) OVER (ORDER BY date) AS pct_change
FROM daily_revenue;
```

**3. Detect Gaps**
```sql
SELECT
    customer_id,
    purchase_date,
    lead(purchase_date) OVER (PARTITION BY customer_id ORDER BY purchase_date) AS next_purchase,
    datediff(lead(purchase_date) OVER (...), purchase_date) AS days_until_next
FROM purchases;
```

### Handling NULLs

By default, first/last rows get NULL. You can provide defaults:

```sql
lag(amount, 1, 0) OVER (...)  -- Default to 0 if no previous row
lead(amount, 1, amount) OVER (...)  -- Default to current amount if no next row
```

### Flow Continuation
We can now do ranking and comparisons. What about more complex frames like moving averages? → SLIDE 9 shows explicit frame control.

---

## **SLIDE 9: Moving Average with Explicit Frame**

### Content
**Advanced Framing**: Fine-grained control over which rows are included

### When You Need It

**Scenario:** Smooth out daily sales noise with a 2-day moving average (current day + previous day)

### Frame Syntax

```sql
ROWS BETWEEN ... AND ...
```

### Frame Examples

#### 2-Row Moving Average (Current + 1 Previous)
```sql
avg(amount) OVER (
    PARTITION BY region
    ORDER BY sale_date
    ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
)
```

**What it includes:**
- 1 row before current
- Current row
- Total: 2 rows

#### 3-Row Moving Average (Current + 2 Previous)
```sql
avg(amount) OVER (
    PARTITION BY region
    ORDER BY sale_date
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
)
```

**What it includes:**
- 2 rows before current
- Current row
- Total: 3 rows

#### Running Total (Default with ORDER BY)
```sql
sum(amount) OVER (
    PARTITION BY region
    ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
```

**What it includes:**
- All rows from start of partition
- Through current row

#### Reverse Running Total
```sql
sum(amount) OVER (
    PARTITION BY region
    ORDER BY sale_date DESC
    ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
)
```

**What it includes:**
- Current row
- All rows to end of partition

#### Whole Partition (Default without ORDER BY)
```sql
sum(amount) OVER (
    PARTITION BY region
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

**What it includes:**
- All rows in partition

### Frame Reference Table

| Frame Specification | Includes | Use Case |
|-------------------|----------|----------|
| `ROWS BETWEEN 1 PRECEDING AND CURRENT ROW` | Current + 1 before | 2-row moving avg |
| `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` | Current + 2 before | 3-row moving avg |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | Start to current | Running total |
| `ROWS BETWEEN CURRENT ROW AND 1 FOLLOWING` | Current + 1 after | Centered window |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | Whole partition | Partition aggregate |

### Practical Example: 2-Day Moving Average

**Query:**
```sql
SELECT
    region,
    salesperson,
    sale_date,
    amount,
    avg(amount) OVER (
        PARTITION BY region
        ORDER BY sale_date
        ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
    ) AS moving_avg_2
FROM sales
ORDER BY region, sale_date;
```

**Output:**
```
region | salesperson | date       | amount | moving_avg_2
East   | Alice       | 2026-01-01 | 100    | 100.0 (only 1 row so far)
East   | Alice       | 2026-01-02 | 150    | 125.0 (avg of 100,150)
East   | Bob         | 2026-01-03 | 200    | 175.0 (avg of 150,200)
East   | Bob         | 2026-01-04 | 50     | 125.0 (avg of 200,50)
```

### Why Explicit Frames Matter

**Without explicit frame:**
```sql
avg(amount) OVER (PARTITION BY region ORDER BY sale_date)
-- Same as: ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
-- Gives running average, not moving average
```

**With explicit frame:**
```sql
avg(amount) OVER (
    PARTITION BY region
    ORDER BY sale_date
    ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
)
-- Gives true 2-row moving average
```

### Flow Continuation
Now we understand all the pieces: partitioning, ordering, ranking, neighbors, and frames. Let's combine them in a real-world example. → SLIDE 10.

---

## **SLIDE 10: Real-World Example - Sales Analysis**

### Content
**Scenario**: Analyze sales comprehensively with multiple window functions

### Business Questions
1. What's each sale as a percentage of the region total?
2. Is it increasing or decreasing compared to previous sale?

### The Query

```sql
SELECT
    region,
    salesperson,
    sale_date,
    amount,
    round(
        100.0 * amount / sum(amount) OVER (PARTITION BY region),
        1
    ) AS pct_of_region_total,
    amount - lag(amount, 1, amount) OVER (
        PARTITION BY salesperson 
        ORDER BY sale_date
    ) AS change_vs_previous
FROM sales
ORDER BY region, salesperson, sale_date;
```

### Breaking Down the Query

**Part 1: Partition by Region (Whole Partition Aggregate)**
```sql
sum(amount) OVER (PARTITION BY region)
```
- Sum all sales in each region
- Return to each row (no ORDER BY, so whole partition)

**Part 2: Calculate Percentage**
```sql
100.0 * amount / sum(amount) OVER (PARTITION BY region)
```
- Individual sale as percentage of region total
- Example: 100 / 400 = 0.25 = 25%

**Part 3: Compare to Previous Sale**
```sql
lag(amount, 1, amount) OVER (PARTITION BY salesperson ORDER BY sale_date)
```
- Look at salesperson's previous sale
- If no previous, use current amount (change = 0)

**Part 4: Calculate Difference**
```sql
amount - lag(amount, 1, amount) OVER (...)
```
- Change in amount from previous sale
- Positive = increase, Negative = decrease

### Expected Output

```
region | salesperson | date       | amount | pct_of_region | change
East   | Alice       | 2026-01-01 | 100    | 25.0%         | 0 (first)
East   | Alice       | 2026-01-02 | 150    | 37.5%         | 50
East   | Bob         | 2026-01-01 | 200    | 50.0%         | 0 (first)
East   | Bob         | 2026-01-03 | 50     | 12.5%         | -150
West   | Carol       | 2026-01-01 | 300    | 52.6%         | 0 (first)
West   | Carol       | 2026-01-02 | 120    | 21.1%         | -180
West   | Dave        | 2026-01-01 | 90     | 15.8%         | 0 (first)
West   | Dave        | 2026-01-02 | 60     | 10.5%         | -30
```

### Insights from Output

1. **Alice's 150 sale** is 37.5% of East region
2. **Bob's sales decreased by 150** from first to third sale
3. **Carol had biggest drop** (-180 from previous)
4. **Dave's sales are declining** consistently

### Why Window Functions Shine Here

Without window functions, you'd need:
- 1 subquery to get region totals (for percentage)
- 1 self-join to get previous sales (for change)
- Multiple passes through data

With window functions:
- Single query, single pass
- Clear logic
- Fast execution

### Flow Continuation
Now we've mastered the practical application. Let's test knowledge with a challenge. → SLIDE 11.

---

## **SLIDE 11: Challenge Question**

### Content
**Scenario**: For each region, show the top salesperson and their metrics

### Challenge Prompt

"For each region, show:
1. Top salesperson (highest single sale)
2. Their ranking within that region
3. Their running total of sales up to that point"

### Hint
Use row_number() to find top salesperson, then add running total

### Solution Approach

**Step 1: Find the top salesperson per region**
```sql
row_number() OVER (PARTITION BY region ORDER BY amount DESC) AS rank
-- rank = 1 is the top
```

**Step 2: Calculate running total for that salesperson**
```sql
sum(amount) OVER (PARTITION BY salesperson ORDER BY sale_date) AS running_total
```

**Step 3: Filter for rank = 1**
```sql
WHERE rank = 1
```

### Complete Solution

```sql
SELECT * FROM (
    SELECT 
        region,
        salesperson,
        amount,
        sale_date,
        row_number() OVER (PARTITION BY region ORDER BY amount DESC) AS rank,
        sum(amount) OVER (PARTITION BY salesperson ORDER BY sale_date) AS running_total
    FROM sales
) 
WHERE rank = 1
ORDER BY region;
```

### Expected Output

```
region | salesperson | amount | date       | rank | running_total
East   | Bob         | 200    | 2026-01-01 | 1    | 200 (his first sale)
West   | Carol       | 300    | 2026-01-01 | 1    | 300 (her first sale)
```

**Interpretation:**
- Bob has the highest single sale in East (200), which was his first/second sale
- Carol has the highest single sale in West (300), which was her first sale

### Why This is Hard

It combines:
- ✓ Window function for ranking (row_number)
- ✓ Different PARTITION BY (by region vs by salesperson)
- ✓ Filtering on window function result
- ✓ Understanding frame defaults

### Flow Continuation
We've covered all concepts and tested them. Now let's recap everything. → SLIDE 12.

---

## **SLIDE 12: Key Takeaways**

### The Seven Core Principles

#### 1. **Window Functions Keep Every Row (Unlike GROUP BY)**

```
GROUP BY:          Window Function:
4 sales → 2 rows   4 sales → 4 rows + context
```

#### 2. **PARTITION BY Defines Logical Groups**

```
PARTITION BY region
= "These rows are related, calculate aggregate for them,
   but keep them all"
```

Not the same as GROUP BY (which also collapses).

#### 3. **Without ORDER BY: Whole Partition Aggregate**

```
sum(amount) OVER (PARTITION BY region)
= Same total for every row in region
```

Each row sees entire partition.

#### 4. **With ORDER BY: Running/Cumulative Aggregate**

```
sum(amount) OVER (PARTITION BY region ORDER BY date)
= Cumulative total up to current row
```

Each row sees start-of-partition to current-row.

#### 5. **MergeTree ORDER BY ≠ Window ORDER BY**

```
MergeTree ORDER BY = Physical storage (optimization)
Window ORDER BY = Logical ordering (analytics)

Both needed, different purposes!
```

#### 6. **Ranking Functions Handle Ties Differently**

```
row_number()  → Always unique (1,2,3,4)
rank()        → Gaps on ties (1,1,3,4)
dense_rank()  → No gaps (1,1,2,3)
```

#### 7. **ROWS BETWEEN Gives Fine-Grained Frame Control**

```
ROWS BETWEEN 1 PRECEDING AND CURRENT ROW  → Moving averages
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW → Running totals
ROWS BETWEEN ... AND ... → Any custom frame
```

### Quick Decision Tree

```
What do you need?

├─ Individual row + region total?
│  └─ sum(amount) OVER (PARTITION BY region)
│
├─ Rank within region by amount?
│  └─ row_number() OVER (PARTITION BY region ORDER BY amount DESC)
│
├─ Running total by region?
│  └─ sum(amount) OVER (PARTITION BY region ORDER BY date)
│
├─ Compare to previous sale?
│  └─ lag(amount) OVER (PARTITION BY salesperson ORDER BY date)
│
├─ Moving average?
│  └─ avg(amount) OVER (... ROWS BETWEEN 1 PRECEDING AND CURRENT ROW)
│
└─ Calculate percentage of group?
   └─ amount / sum(amount) OVER (PARTITION BY group)
```

### When to Use Window Functions

**Perfect for:**
- ✓ Ranking rows within groups
- ✓ Running totals/cumulative sums
- ✓ Moving averages/smoothing
- ✓ Comparing to neighbors (lag/lead)
- ✓ Calculating percentages of group
- ✓ Detecting trends/changes

**Not needed for:**
- ✗ Simple aggregates only (use GROUP BY)
- ✗ Don't need row detail

### Common Pitfalls to Avoid

| Pitfall | Solution |
|---------|----------|
| Forgot ORDER BY for running total | Add `ORDER BY date` inside OVER |
| Wrong PARTITION BY | Define which rows should be related |
| Used GROUP BY when needed window | Remember: GROUP BY collapses rows |
| Confused MergeTree ORDER BY | MergeTree = storage, Window = logic |
| Ties not handled right | Choose correct ranking function |

### Performance Considerations

**Window functions are fast because:**
- ✓ Single pass through data
- ✓ No expensive self-joins
- ✓ ClickHouse optimizes well
- ✓ Especially good for time-series data

**Best practices:**
- ✓ Always ORDER BY for running totals
- ✓ Use PARTITION BY to limit scope
- ✓ Filter results with WHERE (not inside OVER)
- ✓ Test with EXPLAIN to see execution

### Next Steps for Mastery

1. **Practice with Real Data:**
   - Analyze your own transaction log
   - Calculate monthly running totals
   - Rank items within categories

2. **Combine Multiple Functions:**
   - Use ranking + running totals together
   - Add lag/lead to detect anomalies
   - Create sophisticated analytics

3. **Optimize Performance:**
   - Study EXPLAIN output
   - Understand how ORDER BY affects frame
   - Profile queries with complex windows

4. **Build Analytical Tools:**
   - Create dashboards with rank/percentage calculations
   - Detect trends with lag/lead
   - Build alerts on moving averages

---

## **Quick Reference Card**

```
╔═════════════════════════════════════════════════════════════╗
║ CLICKHOUSE WINDOW FUNCTIONS - QUICK REFERENCE              ║
╠═════════════════════════════════════════════════════════════╣
║                                                              ║
║ BASIC SYNTAX:                                              ║
║   func(...) OVER (PARTITION BY x ORDER BY y ROWS ...)      ║
║                                                              ║
║ AGGREGATES (whole partition):                              ║
║   sum(col) OVER (PARTITION BY x)                           ║
║   avg(col) OVER (PARTITION BY x)                           ║
║   count() OVER (PARTITION BY x)                            ║
║                                                              ║
║ RUNNING TOTALS (with ORDER BY):                            ║
║   sum(col) OVER (PARTITION BY x ORDER BY y)                ║
║                                                              ║
║ RANKING:                                                   ║
║   row_number() OVER (PARTITION BY x ORDER BY y DESC)       ║
║   rank() OVER (PARTITION BY x ORDER BY y DESC)             ║
║   dense_rank() OVER (PARTITION BY x ORDER BY y DESC)       ║
║                                                              ║
║ NEIGHBORS:                                                 ║
║   lag(col, 1) OVER (PARTITION BY x ORDER BY y)             ║
║   lead(col, 1) OVER (PARTITION BY x ORDER BY y)            ║
║                                                              ║
║ MOVING AVERAGE:                                            ║
║   avg(col) OVER (...ORDER BY y ROWS BETWEEN 1 PRECEDING...) ║
║                                                              ║
║ KEY INSIGHT: ORDER BY CHANGES DEFAULT FRAME                ║
║   Without: whole partition    |  With: start to current    ║
║                                                              ║
╚═════════════════════════════════════════════════════════════╝
```

---

## **Learning Flow Summary**

### Progressive Complexity

**Beginner (Slides 1-4):**
- What are window functions?
- vs GROUP BY
- vs PARTITION BY concept

**Intermediate (Slides 5-9):**
- MergeTree vs Window ORDER BY
- Ranking functions
- Running totals (critical!)
- lag/lead functions
- Explicit frame control

**Advanced (Slides 10-12):**
- Real-world multi-function queries
- Challenge combining everything
- Performance and best practices

### Knowledge Building

```
Foundation:        Understand window functions keep rows
    ↓
Concepts:          Learn PARTITION BY and ORDER BY
    ↓
Critical:          ORDER BY changes default frame
    ↓
Functions:         Ranking, running, comparisons, frames
    ↓
Integration:       Combine multiple functions
    ↓
Application:       Real-world analytics queries
    ↓
Mastery:           Optimize and debug complex queries
```

---

**End of ClickHouse Window Functions: Complete Slide-by-Slide Guide**

This markdown guide is designed to be:
- **Comprehensive**: Covers all concepts in depth
- **Visual**: ASCII diagrams and tables for clarity
- **Practical**: Real SQL examples and use cases
- **Sequential**: Builds knowledge progressively
- **Reference**: Can be used as lookup guide
- **Educational**: Explains the "why" not just "how"
