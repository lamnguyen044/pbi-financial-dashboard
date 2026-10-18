# Financial Dashboard (Power BI)

**[Open the live dashboard]([https://app.powerbi.com/groups/me/reports/10b0d6fa-b7ab-4053-8677-20c7f64f050e/f0fe1a3a1d1b767dafec?experience=power-bi](https://app.powerbi.com/view?r=eyJrIjoiZjk2N2Q4MmQtNTFlMi00NTYxLTk1MjYtYjhkOTlhZGEyNjFlIiwidCI6IjM3MGZiM2I4LTMzMDYtNDg5MC05MDYzLWNjMDhiZTc4ODI1NyIsImMiOjEwfQ%3D%3D))**

A four-page report for a CFO built on a general ledger: Income Statement, Financial Details, Balance Sheet and Cash Flow, with Actual vs Budget vs prior year. This README covers only the data source, how the data was processed, and the model.

---

## 1. Data source

| Source | Content | Used for |
|---|---|---|
| `findata.xlsx` sheet **GL** | 27,909 ledger lines, 2018–2020, 7 territories | `Fact_GL` (Actual) |
| `findata.xlsx` sheet **COA** | 54 accounts (Report, Class, SubClass, Account, SubAccount) | `Dim_Account` |
| `findata.xlsx` sheet **Territory** | 7 countries: Australia, Canada, France, Germany, New Zealand, UK, USA | `Dim_Territory` |
| `findata.xlsx` sheet **Calendar** | 1,096 days | Reference only; `Dim_Date` is rebuilt in DAX |
| `findata.xlsx` sheets **CashFlow_St**, **SoCE_St** | Sample cash flow and equity statements | `Dim_CashFlowLine` and reconciliation only |
| `Fact_Budget.csv` | 2,716 rows, 2019-01 to 2020-12 | `Fact_Budget` (Budget) |

> **The Budget is synthetic.** The source has no budget, so `Fact_Budget.csv` was generated with `generate_budget.py` (fixed seed 2026) to demonstrate variance analysis. It is not a real company budget.

### How the synthetic Budget is built

```
Budget = Actual(prior year, same month, territory, account) × (1 + growth) × (1 + scenario) × (1 + noise)
```

- **2019 growth**: company-wide growth by account SubClass. **2020 growth**: each territory's 2018→2019 trend × 0.9, capped between −10% and +80%.
- **Scenarios** (deliberate deviations): Germany 2020 operating expenses −15%; Australia Q2-2020 sales +20%; USA Q4-2019 sales −8%.
- **Noise**: normal, standard deviation 3%.
- **Not budgeted**: Gain/Loss on asset sale, Exchange, Dividend Income.

To use a real budget, replace `Fact_Budget` with the same columns: `Date, Territory_key, Account_key, Scenario, Version, Amount`. No measure changes.

---

## 2. Data processing

**Profiling** (done before modelling): key integrity between GL, COA and Territory; date coverage; sign convention; duplicates and nulls in key columns; and control totals (Assets = Liabilities + Equity at every year-end).

**Decisions and reasons**

| Decision | Reason |
|---|---|
| Keep the GL sign convention (income +, expenses −) | Actual − Budget > 0 is always favourable, on revenue and cost lines alike. On cost lines, YoY % is negative when costs rise |
| Build `Dim_Date` in DAX instead of using the Calendar sheet | Full control of Year, Quarter, Month and sort order; continuous dates for time intelligence |
| Keep COA as its own dimension | One place defines Class / SubClass grouping |
| Keep the sample cash flow and equity sheets out of the star schema | They are results, not inputs; used only to verify what the model computes |
| Do not budget non-recurring accounts | They cannot be forecast |
| Single-year Year slicer (default 2020) | Budget exists only for 2019–2020, so "All" mixes periods and breaks prior-year comparisons |

**Refresh**: Power Query loads the workbook and CSV. Update the file paths (or switch to a database or SharePoint source), then Refresh. Measures do not change.

---

## 3. Data model

Star schema: two fact tables sharing three dimensions.

```
 Dim_Date ───────┐
 Dim_Account ────┼──► Fact_GL     (Actual)
 Dim_Territory ──┘──► Fact_Budget (Budget)
```

**Relationships**: six, all one-to-many, single direction, active.

| From (one) | To (many) |
|---|---|
| `Dim_Date[Date]` | `Fact_GL[Date]`, `Fact_Budget[Date]` |
| `Dim_Account[Account_key]` | `Fact_GL[Account_key]`, `Fact_Budget[Account_key]` |
| `Dim_Territory[Territory_key]` | `Fact_GL[Territory_key]`, `Fact_Budget[Territory_key]` |

**Disconnected tables** (no relationships; read through measures)

| Table | Purpose |
|---|---|
| `PL_Layout`, `BS_Layout`, `CF_Layout` | One row per statement line: `Line`, `Order`, `Kind` (Header / Detail / Subtotal / Total) and an Account_key range (`KeyFrom`–`KeyTo`) or Rank range. A measure reads the line in the current matrix cell and filters the accounts in that range, so changing the report structure only means editing the layout table |
| `Dim_CashFlowLine` | Cash flow line catalogue with section and order |
| `View_Select` | Buttons for Actual, vs Budget, vs Last Year, % of Revenue on Financial Details |

**Key measures** (full list in `measures.dax`)

- **Base**: `Actual` and `Budget` use `KEEPFILTERS` on `Dim_Account[Report]`, so a visual's own filter is respected. `Variance = Actual − Budget`; `Variance % = DIVIDE(Variance, ABS(Budget))`.
- **Time**: `Actual PY` with `SAMEPERIODLASTYEAR`; `YoY %`; YTD versions.
- **P&L**: Revenue, COGS, Gross Profit, EBIT (Trading + Operating accounts only), Net Profit, each with a Budget version.
- **Balance Sheet**: `Closing Balance` is semi-additive (balance up to the last date of the selected period), then Working Capital, Current / Quick ratio, Debt ratio, Debt-to-Equity, DSO, DIO, DPO.
- **Cash Flow**: indirect method from GL, driven by `Dim_CashFlowLine` rules (opening, closing, positive, negative, change-sign) and `TREATAS`; Operating / Investing / Financing CF, Net Change in Cash, Cash Conversion % (Operating CF ÷ Net Profit).
- **Checks**: `Check BS Balances` and `Check Cash Flow` feed the colour labels on pages 3 and 4.

---

## 4. Reconciliation

Figures were recomputed independently from the source with Python and matched against the report:

- Assets − Liabilities − Equity = 0 at each year-end (2020: 12,320,001 − 1,971,645 − 10,348,356).
- Opening cash + net change = closing cash (2020: 5,111,014 − 600,282 = 4,510,732).
- Cumulative Net Profit 2018–2020 = Retained Earnings account (3,216,948).
- 2020 quarterly revenue sums to the year (7,835,369); revenue variance (−935,453) equals the sum of the 12 months and the 7 countries.
- Budget Net Profit cross-checked with Excel `SUMIFS` (sheet `Budget_Check`): 1,065,815 (2019) and 2,329,328 (2020).

---

## 5. Repository layout

| File | Content |
|---|---|
| `README.md` | This file |
| `measures.dax` | All DAX measures and layout-table definitions |
| `Fact_Budget.csv` | Synthetic Budget |
| `generate_budget.py` | Script that generates the Budget (seed 2026) |
| `CFO_Model_and_Budget.xlsx` | Design workbook: model tables, relationships, measures, profiling notes, budget check |
