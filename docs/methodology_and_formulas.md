# Methodology & Google Sheets Formula Architecture

This document details the analytical engineering, spreadsheet formulas, pivot configurations, and data-quality safeguards implemented across the Google Sheets workbook pipeline.

---

## 1. Relational Schema & Data Typing

| Table / Sheet | Granularity | Key Columns & Data Types |
| :--- | :--- | :--- |
| `games` | Game title (`3` titles) | `game_name` (*String*), `Ad spend` (*Float*), `Total users count` (*Integer*) |
| `games_payments` | Transaction (`3,029` rows) | `user_id` (*String* — identifier, non-numeric arithmetic), `game_name` (*String*), `payment_date` (*Date*), `revenue_amount_usd` (*Float*) |
| `games_paid_users` | Paying player (`999` rows) | `user_id` (*String*), `game_name` (*String*), `language` (*String*), `has_older_device_model` (*Boolean*), `age` (*Integer*) |
| `active users` | Active player (`216` rows) | `user_id` (*String*), `game_name` (*String*), `language` (*String*), `has_older_device_model` (*Boolean*), `age` (*Integer*) |
| `activity` | Daily gameplay event (`44,012` rows) | `user_id` (*String*), `activity_date` (*Date*), `game_activity_name` (*String*), `total_seconds` (*Integer*) + engineered columns |
| `activity_types` | Lookup table (`8` mappings → `5` categories) | `activity_name` (*String*), `activity_type` (*String*) |

---

## 2. Core Formula Engineering by Analytical Layer

### Layer A — Monetization KPIs & Payer Demographics
Calculated dynamically on the `games` summary table:
- **Total Revenue:** `=SUMIF(games_payments!B:B, A2, games_payments!D:D)`
- **Paid Users Count:** `=COUNTIF(games_paid_users!B:B, A2)`
- **Conversion Rate to Paid (%):** `=E2 / C2` *(Paid Users / Total Users)*
- **ARPPU:** `=D2 / E2` *(Total Revenue / Paid Users)*
- **Paid User Age Stats:**
  - Mean: `=AVERAGEIF(games_paid_users!B:B, A2, games_paid_users!E:E)`
  - Median (filtered per game): `=MEDIAN(FILTER(games_paid_users!E:E, games_paid_users!B:B = A2))`
  - Bounds: `=MINIFS(...)` and `=MAXIFS(...)`

> **Aggregation Guardrail:** When building portfolio `Total` rows, ratio KPIs (`CR %`, `ARPPU`) and averages/medians must be recomputed from aggregated totals or raw ranges rather than summed across rows.

### Layer B — Age Variability & Daily/Weekly Engagement (`DAU`, `WAU`, `Stickiness`)
- **Variability Summary (`active users`):**
  - `Mean`: `=AVERAGE(E:E)` → `28.01`
  - `Std Dev`: `=STDEV(E:E)` → `11.28`
  - `Median`: `=MEDIAN(E:E)` → `25`
  - `IQR`: `=PERCENTILE(E:E, 0.75) - PERCENTILE(E:E, 0.25)` → `15`
  - `P10 / Max`: `=PERCENTILE(E:E, 0.10)` (`17`) and `=PERCENTILE(E:E, 1)` (`83`)
- **Monday-Aligned Week Start (`activity` & `dau`):**
  - `=B2 - WEEKDAY(B2, 3)` *(Type `3` indexes Monday as `0`, ensuring exact ISO-week alignment across calendar month boundaries).*
- **Daily Active Users (`dau`, `306` days):**
  - `=COUNTUNIQUEIFS(activity!A:A, activity!B:B, A2)`
- **Weekly Active Users & Stickiness (`wau`, `44` actual weeks):**
  - `WAU`: `=COUNTUNIQUEIFS(activity!A:A, activity!J:J, A2)`
  - `Average DAU`: `=SUMIF(dau!C:C, A2, dau!B:B) / COUNTIF(dau!C:C, A2)`
  - `Stickiness (DAU/WAU)`: `=C2 / B2` *(formatted as `%`)*

### Layer C — 20-Week Linear Regression Forecasting (`Jan–May 2023`)
Extended the `wau` sheet by 20 weekly intervals using absolute range locking to prevent sliding-window drift:
- **Projected WAU:** `=ROUND(FORECAST(A46, $B$2:$B$45, $A$2:$A$45), 0)`
- **Projected Avg DAU:** `=ROUND(FORECAST(A46, $C$2:$C$45, $A$2:$A$45), 0)`
- **Projected Stickiness:** `=ROUND(C46 / B46, 2)`

### Layer D — Feature Engineering & Cohort Retention Pipeline (`44,012` Rows)
1. **Text Parsing:** Split composite `game_activity_name` (`"game 3: Main campaign"`) into `game_name` and `activity_name` using `=SPLIT(F2, ":")`.
2. **Dimensional Enrichment via `VLOOKUP`:**
   - Normalized 8 granular activity labels into 5 categories (`Battle pass`, `Campaign`, `Clan multiplayer`, `Mini-games`, `Special events`) via `=VLOOKUP(H2, ' activity_types'!$A$2:$B$9, 2, FALSE)`.
   - Joined player locale from `active users` via `=VLOOKUP(A2, 'active users'!$A$2:$E$217, 3, FALSE)`.
3. **Cohort Indexing:**
   - `activity month`: `=MONTH(B2)`
   - `first activity month`: `=MONTH(MINIFS($B$2:$B$44013, $A$2:$A$44013, A2))`
   - `activity month number`: `=C2 - D2` *(guarantees zero-indexed non-negative tenure `0..9` within 2022).*
4. **Pivot Table & Dynamic Retention Matrix:**
   - **Pivot Table:** Rows = `first activity month`, Columns = `activity month number`, Values = `COUNTUNIQUE(user_id)`.
   - **Companion Retention Rate (%) Matrix:** Built outside the pivot range to avoid circular dependencies, using absolute column-locked denominator referencing and empty-cell `#DIV/0!` guards:
     `=IF(C3 > 0, C3 / $B3, "")`
   - **Visualization & Interactive Controls:** Dual min-to-max gradient color scales (rose → yellow → green) and **3 connected Slicers** (`game_name`, `activity_type`, `activity_language`) targeting the `44,012`-row `activity` range.
