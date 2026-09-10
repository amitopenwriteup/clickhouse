# ClickHouse Materialized Views Lab — Hinglish Explanation

## Overall Context

Acha, toh isko samajhte hain. Ek online store hai — jaha customer orders daal rahe hain. Ab hume daily revenue report chahiye, top customers dekne hain, aur monthly-yearly rollups bhi chahiye. Lekin har baar sab data re-scan karna CPU ka waste hai na?

**Materialized Views (MV)** woh magic tool hain jo real-time mein aggregated data maintain karte hain.

---

## Part A — Incremental Materialized View

### Kya kaam hai?

Online store ke liye **daily revenue per category** maintain karna — jaise electronics mein aaj kitna aaya, books mein kitna aaya, automatically calculate ho jaye.

### Kaise kaam karta hai?

**Step 1:** Phele ek target table banao — `daily_revenue` ka naam de do.

Isme columns hain:
- Day (date)
- Category (electronics, books, toys, etc.)
- Total_amount (kitna revenue)
- Order_count (kitne orders)

**Step 2:** Ab `daily_revenue_mv` naam ka materialized view banate hain.

Ye view har baar jab koi naya order `orders` table mein insert hota hai, toh uska data automatically `daily_revenue` table mein sum/aggregate karke daal deta hai.

```
Jab order insert hota hai:
orders → daily_revenue_mv (calculate karta hai) → daily_revenue (result store karta hai)
```

**Step 3:** Test karte hain — do electronics orders insert karo (120 + 45 = 165) aur ek books order (15.50).

Query maro toh dekhoge — **165 electronics aur 15.50 books** automatically sum ho gaya! 🎉

### Important Gotcha! ⚠️

Agar table mein *pehle se* data tha aur baad mein MV banaya, toh **purana data ignore ho jayega**. Sirf jo naye orders insert honge woh count honge.

Agar backfill karna hoga toh MV creation ke time `POPULATE` keyword add karna padega.

---

## Part B — Refreshable Materialized View

### Kya kaam hai?

**"Top 5 customers by spending"** — iska matalab customer ka full profile (naam, country) + total amount join karke dekna.

Ye simple incremental view se different hai kyunki **JOIN lag raha hai** customers table ka saath.

### Kaise kaam karta hai?

**Step 1:** Target table banao — `top_customers`.

**Step 2:** Materialized view banao — **lekin incremental nahi, REFRESHABLE**.

```sql
REFRESH EVERY 30 SECOND TO top_customers
```

Matlab: har 30 second mein poora query re-run hota hai. Incremental ke tarah per-insert nahi, timer-based.

**Step 3:** `SYSTEM REFRESH VIEW` command se manually refresh karo aur dekho — customer 1 (Asha) top pe hai 165.00 ke saath.

**Step 4:** Status check karo `system.view_refreshes` table se:
- Kab last successful refresh hua?
- Next refresh kab hoga?

**Step 5:** Agar refresh frequency change karna ho toh:

```sql
ALTER TABLE top_customers_mv MODIFY REFRESH EVERY 5 MINUTE;
```

### Optional: APPEND Mode

Agar historical snapshots chahiye (har 30 second mein ek snapshot save karna), toh `APPEND` mode use karo. Table grow hota jayega, replace nahi hota.

---

## Part C — Cascading Materialized Views

### Concept: "Nesting Dolls" Style Aggregation

Sochte hain ek scenario:

```
Raw orders table
    ↓
Daily revenue (Part A)
    ↓ (yeh become source for...)
Monthly revenue
    ↓ (yeh become source for...)
Yearly revenue
```

**Ek order** flow hota hai chhote se bade level tak — automatically!

### Step-by-Step

**Step 1: Monthly rollup banao**

`daily_revenue` ko source banake `monthly_revenue` table mein aggregate karo.

```
daily_revenue → monthly_revenue_mv → monthly_revenue
```

Har din jo new data `daily_revenue` mein aata hai, woh automatically month mein roll-up hota hai.

**Step 2: Yearly rollup banao**

`monthly_revenue` ko source banake `yearly_revenue` banao.

```
monthly_revenue → yearly_revenue_mv → yearly_revenue
```

**Step 3: Test — do orders insert karo:**

```
Order 1: June 15, 2024 — books, $20
Order 2: June 20, 2024 — books, $10
Order 3: January 5, 2023 — toys, $50
```

**Flow:**
1. Orders table mein insert → daily_revenue_mv trigger → daily_revenue update (2024 ka 2 entries, 2023 ka 1)
2. Daily_revenue update → monthly_revenue_mv trigger → monthly_revenue update
3. Monthly_revenue update → yearly_revenue_mv trigger → yearly_revenue update

**Result:** sab kuch automatically ho gaya!

**Step 4: Verify each level:**

```
Daily: June 15, 20, January 5 — dates dikhenge
Monthly: June 2024, January 2023 — months dikhenge
Yearly: 2024 (30 books), 2023 (50 toys) — years dikhenge
```

---

## Key Gotchas (Important!) ⚠️

1. **Incremental MV ka historical data problem:** Pehle se jo data tha, woh ignore hota hai.

2. **JOIN issue:** Incremental MV mein LEFT JOIN lete hain toh sirf left-most table (source) trigger karta hai.

3. **AggregatingMergeTree trick:** Cascading mein intermediate table mein `AggregateFunction` states store hote hain (poora sum nahi), toh `sumMerge()` use karna padta hai output mein.

4. **Real-time vs Batch trade-off:**
   - Incremental = real-time pero simple queries
   - Refreshable = delayed pero complex JOINs support karte hain

---

## Summary Table

| Part | Type | Mechanism | Best Use |
|------|------|-----------|----------|
| A | Incremental | Insert → Calculate → Store | Real-time summaries, single table |
| B | Refreshable | Timer-based full query re-run | Complex reports with JOINs |
| C | Cascading | Chain multiple rollups | Daily → Monthly → Yearly |

---

**Conclusion:** Ye lab mein samjh aata hai ki ClickHouse mein data ki aggregation kitni efficiently ho sakti hai — CPU aur storage dono bachta hai! 🚀
