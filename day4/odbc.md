# ClickHouse ODBC Driver — Presentation Guide

This document walks through each slide of `ClickHouse_ODBC_Driver.pptx` and explains the reasoning and source material behind it.

---

## Slide 1: Title
**ClickHouse ODBC Driver** — A standards-compliant interface for connecting ODBC applications to ClickHouse.

Sets the scope of the deck: this is about the official ClickHouse ODBC driver, not a general ODBC tutorial.

---

## Slide 2: What the ODBC Driver Does
Four core facts about the driver:

- **Standards-based access** — implements the ODBC API so any ODBC-aware tool (BI tools, scripts, apps) can talk to ClickHouse.
- **HTTP under the hood** — the driver doesn't use ClickHouse's native TCP protocol; it talks over HTTP, which is the most universally supported protocol across ClickHouse deployments.
- **Works everywhere** — because it's HTTP-based, it works the same way against local installs, self-managed clusters, and ClickHouse Cloud.
- **Open source** — actively developed at `github.com/ClickHouse/clickhouse-odbc`; features are still being added.

**Why this matters:** unlike some proprietary drivers, this one has one code path regardless of where ClickHouse lives, which simplifies support and debugging.

---

## Slide 3: How It Connects
A simple three-box flow diagram:

```
ODBC-Compatible Application  →  ClickHouse ODBC Driver  →  ClickHouse Server (via HTTP)
      (e.g. Power BI)
```

The driver sits in the middle: it accepts standard ODBC calls from the application and translates them into HTTP requests that ClickHouse understands. The application never needs to know ClickHouse-specific protocol details.

---

## Slide 4: Installation & Testing (Windows)
- **Installation:** download the MSI installer from the [clickhouse-odbc releases page](https://github.com/ClickHouse/clickhouse-odbc/releases/latest) and run it.
- **Recommended:** run ClickHouse server 24.11+ for best driver compatibility.
- **Testing:** a PowerShell snippet opens an `OdbcConnection`, runs `select version()`, and prints the result — the fastest way to confirm the driver and connection string both work before wiring up a BI tool.

```powershell
$url = "http://127.0.0.1:8123/"
$username = "default"
$password = ""
$conn = New-Object System.Data.Odbc.OdbcConnection(
  "Driver={ClickHouse ODBC Driver (Unicode)};" +
  "Url=$url;Username=$username;Password=$password")
$conn.Open()
$cmd = $conn.CreateCommand()
$cmd.CommandText = "select version()"
$reader = $cmd.ExecuteReader()
$reader.Read()
$reader.GetValue(0)
$reader.Close()
$conn.Close()
```

---

## Slide 5: Configuration Parameters
A reference table of the most common connection parameters:

| Parameter | Purpose |
|---|---|
| Url | Full HTTP(S) endpoint of the ClickHouse server |
| Username / Password | Authentication credentials |
| Database | Default database for the connection |
| Timeout | Max seconds to wait for a server response |
| ClientName | Custom identifier sent in the User-Agent header (useful for tracing) |
| Compression | Enables HTTP compression to reduce bandwidth on large result sets |
| SqlCompatibilitySettings | Makes ClickHouse behave more like a traditional RDBMS for tools like Power BI |

The full parameter list lives in the driver's GitHub repo — this table only covers the ones most people need day to day.

---

## Slide 6: Example Connection Strings
Two real examples, side by side, to show that only the `Url` and credentials change between environments:

**Local (WSL) install:**
```
Driver={ClickHouse ODBC Driver (Unicode)};Url=http://localhost:8123/;Username=default
```

**ClickHouse Cloud:**
```
Driver={ClickHouse ODBC Driver (Unicode)};Url=https://your-instance.gcp.clickhouse.cloud:8443/;Username=default;Password=your-password
```

---

## Slide 7: Microsoft Power BI Integration
Power BI ships **two** ODBC-based connectors, and picking the right one matters:

| | ClickHouse Connector (Recommended) | Generic ODBC Connector |
|---|---|---|
| Mode | Supports **DirectQuery** | **Import** only |
| Behavior | Power BI auto-generates SQL, pulls only what's needed per visual/filter | Runs the user's query (or whole table) and imports the entire result set |
| Refresh | Live, incremental | Every refresh re-imports everything |
| Best for | Interactive dashboards over large datasets | When you need a full local copy of the data |

**Takeaway:** default to the ClickHouse Connector unless you specifically need a static, fully-imported dataset.

---

## Slide 8: SQL Compatibility Settings
ClickHouse's SQL dialect differs from the standard in a few places that trip up BI-generated queries. `SqlCompatibilitySettings` turns on two settings to smooth this over:

- **`cast_keep_nullable`** — By default, ClickHouse refuses to `CAST` a `Nullable` column to a non-`Nullable` type, which many BI-generated queries do without thinking. This setting makes `CAST` preserve nullability instead of erroring.
- **`prefer_column_name_to_alias`** — ClickHouse normally resolves a repeated name to its *alias* within the same `SELECT` list (e.g. `SELECT sum(value) AS value, avg(value)` tries to average the alias, not the column) — which can silently turn into an illegal nested aggregate. This setting makes ClickHouse prefer the underlying column instead, matching how most other databases resolve subquery aliases.

**Caveat flagged on this slide:** users with `readonly = 1` can't change *any* setting, even for `SELECT` queries — so enabling `SqlCompatibilitySettings` will error out for them. That's addressed on the next slide.

---

## Slide 9: Making It Work for Read-Only Users
Two fixes for the `readonly` conflict:

1. **Recommended — set `readonly = 2`**, which permits changing settings while keeping the user read-only for data:
   ```sql
   ALTER USER your_odbc_user MODIFY SETTING readonly = 2
   ```
2. **Alternative — pre-set the same values the driver tries to apply**, so the driver's attempt becomes a no-op:
   ```sql
   ALTER USER your_odbc_user MODIFY SETTING
       cast_keep_nullable = 1,
       prefer_column_name_to_alias = 1
   ```
   Trade-off: this needs upkeep, since future driver versions may add more settings you'd have to mirror manually.

---

## Slide 10: Summary
Closing recap of the five things to remember:

- Standards-compliant ODBC access to ClickHouse over HTTP — works locally and in the cloud.
- Configure via `Url`, `Username`, `Password`, `Database`, `Timeout`, `ClientName`, `Compression`.
- Turn on `SqlCompatibilitySettings` when BI tools generate non-ClickHouse-flavored SQL.
- Prefer the ClickHouse Connector (DirectQuery) over the generic ODBC connector in Power BI where possible.
- For read-only users, set `readonly = 2` to avoid compatibility-setting errors.

**Sources:**
- GitHub: `github.com/ClickHouse/clickhouse-odbc`
- Docs: `clickhouse.com/docs/concepts/features/interfaces/odbc`

---

## Design Notes (for whoever edits the deck later)
- Palette: navy (`#21295C`) for headers, deep blue (`#065A82`) / teal (`#1C7293`) as accents, all on a **pure white background** with no decorative icons — per the request that generated this deck.
- Fonts: Cambria for headers, Calibri for body text, Courier New for code/config blocks — all in PowerPoint's safe font list, so text-fit is reliable across machines.
- Layout motif: light-gray-bordered boxes (`#F5F6F8` fill) for code and card content, used consistently across slides 4, 6, 7, 8, and 9.
