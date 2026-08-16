# Open to Buy · the season in flight

A merchandise planning application for an apparel buyer. It answers the question
a planner asks in the first week of every month: where is the season against
plan, what did that cost in markdown, and what still has to be bought.

**Live site:** https://astrodataus.github.io/open-to-buy/

Built by [Astrodata](https://astrodata.us). The same document also runs as an
Omni app; this repository is the static build of it.

---

## The data is invented

Nothing here is a real company. It is a synthetic plan shaped like a children’s
apparel business: 36 million dollars, four divisions, twenty departments, three
channels, three fiscal years, with a February to July 2026 actuals window
against a plan that was reforecast mid-year. It behaves like a real plan,
including the parts that do not tie neatly, but no figure describes anyone.

The plan balances to the dollar on the standard retail identity:

```
BOM + Receipts − Sales − Markdown − Shrink = EOM
```

---

## How this repository works

`index.html` is the whole application: one document, no build step, no
framework, no server. On load it fetches the eight CSVs sitting beside it and
draws everything from them.

Every chart is hand-drawn SVG and both typefaces are base64 woff2 inlined into
the document, so **the page makes zero requests off its own origin**. That is
not minimalism for its own sake. The Omni app sandbox does not allowlist CDNs,
so anything fetched from one works locally and fails silently in production.

| File | What it is |
|---|---|
| `index.html` | the application, web build, reads the CSVs beside it |
| `open-to-buy-standalone.html` | the same application with every row inlined, one file, opens offline |
| `*.csv` | the eight query results, one file per view |
| `.nojekyll` | stops GitHub Pages running Jekyll, whose `{{` and `{%` delimiters collide with encoded data |

---

## Data dictionary

Column names are `<view>.<column>`, which is the field ID the underlying Omni
SQL query views emit. They are kept verbatim so a figure on the page can be
traced back to a query without a translation step.

Money is US dollars, rounded once at build time to two decimal places. Counts
are integers. **An empty cell is a missing value, never a zero:** a department
with no actual recorded yet is not a department that sold nothing.

Two grains recur throughout. `dept_key` is `<division> <department>`, for
example `Girls Dresses`. `month_key` is `FY<year>-<period>`, for example
`FY2026-06`, where period 1 is February.

### `plan-vs-actual-monthly.csv` · 108 rows

Plan against actual at month and channel, all three fiscal years.

| Column | Type | Notes |
|---|---|---|
| `month_key` | text | `FY2024-01` through `FY2026-12` |
| `fiscal_year` | integer | 2024, 2025, 2026 |
| `fiscal_month` | text | `Feb` through `Jan` |
| `fiscal_month_no` | integer | 1 to 12, February is 1 |
| `channel` | text | Ecommerce, Retail Stores, Wholesale |
| `plan_sales_dollars` | money | |
| `actual_sales_dollars` | money | empty after July 2026, the season is open |
| `plan_gross_margin_dollars` | money | |
| `actual_gross_margin_dollars` | money | empty after July 2026 |

### `department-variance.csv` · 2,160 rows

The main fact table. Twenty departments by three channels by twelve months by
three years. Everything on the variance and plan tabs is an aggregate of this.

| Column | Type | Notes |
|---|---|---|
| `dept_key`, `division`, `department` | text | |
| `channel` | text | |
| `fiscal_year`, `fiscal_month`, `fiscal_month_no` | integer / text | |
| `has_actual` | boolean | `true` where the month has closed |
| `plan_sales_dollars` | money | |
| `actual_sales_dollars` | money | empty where `has_actual` is false |
| `plan_markdown_dollars` | money | |
| `actual_markdown_dollars` | money | empty where `has_actual` is false |
| `actual_eom_retail` | money | ending inventory at retail |
| `plan_bom_retail` | money | beginning inventory at retail |
| `plan_eom_retail` | money | |
| `plan_receipts_retail` | money | |
| `plan_shrink_dollars` | money | |

### `plan-version-compare.csv` · 120 rows

Original plan against current plan, at department and year. This is what makes
the reforecast visible: a department can beat the plan it was cut to and still
have missed the plan it started with.

| Column | Type | Notes |
|---|---|---|
| `dept_key`, `division`, `department` | text | |
| `plan_version` | text | `Original Plan` or `Current Plan` |
| `fiscal_year` | integer | |
| `plan_sales_dollars`, `plan_receipts_cost`, `plan_gross_margin_dollars` | money | |

### `otb-by-month.csv` · 18 rows · `otb-by-department.csv` · 60 rows

Open to buy, the money not yet committed. `open_to_buy_cost` is
`plan_receipts_cost − on_order_cost`, at cost rather than retail, which is the
currency a buyer actually places orders in.

| Column | Type | Notes |
|---|---|---|
| `month_key`, `fiscal_month`, `fiscal_month_no` | text / integer | month file only |
| `dept_key`, `division`, `department` | text | department file only |
| `channel` | text | |
| `plan_receipts_cost` | money | what the plan says to receive |
| `on_order_cost` | money | already committed on purchase orders |
| `open_to_buy_cost` | money | the difference, what is left to spend |

### `po-watchlist.csv` · 360 rows

Receipts that have not been ordered yet, dated backwards from the receipt month
by the vendor lead time. `po_placement_by` is the last day an order can go out
and still land on time.

| Column | Type | Notes |
|---|---|---|
| `month_key`, `fiscal_month`, `channel` | text | |
| `dept_key`, `division`, `department` | text | |
| `lead_time_weeks` | integer | vendor lead time |
| `po_placement_by` | date, `YYYY-MM-DD` | receipt month start minus lead time |
| `po_status` | text | `Past due`, `Due now`, `Upcoming` |
| `days_past_placement` | integer | negative means not yet due |
| `status_rank` | integer | 1 to 3, sorts the statuses in urgency order |
| `plan_receipts_cost`, `on_order_cost`, `open_to_buy_cost` | money | |

### `style-detail.csv` · 599 rows

Style level, one row per style and channel for fiscal 2026. Carries `print_name`,
which is the join that lets a print be traced across departments. That question
is a merchandising one, and it is not normally answerable from a plan at all,
because prints and plans do not usually live in the same place.

| Column | Type | Notes |
|---|---|---|
| `style_key` | text | `ST-0085` |
| `style_name`, `print_name`, `class_name` | text | |
| `division`, `department`, `dept_key` | text | |
| `season` | text | Spring, Summer, Fall, Holiday |
| `lifecycle` | text | New, Carryover, Clearance |
| `channel` | text | |
| `fiscal_year` | integer | 2026 only |
| `units_sold`, `on_hand_units`, `on_order_units` | integer | |
| `sales_dollars` | money | |

### `year-totals.csv` · 3 rows

One row per fiscal year, for the masthead.

| Column | Type |
|---|---|
| `fiscal_year` | integer |
| `plan_sales_dollars`, `actual_sales_dollars`, `plan_gross_margin_dollars`, `plan_receipts_cost` | money |

---

## Rebuilding

`build.py` in the source directory produces all three targets from one template
and one dataset. It rounds every number once, before writing anything, so the
embedded build and the published CSVs cannot disagree about a figure.

```
python3 build.py         # writes dist/, the standalone file, and the Omni build
python3 verify_web.py    # serves dist/ over HTTP and checks it against the CSVs
```

`verify_web.py` also takes an origin, which is how the deployed site gets
checked against its own bytes rather than against the local build:

```
python3 verify_web.py https://astrodataus.github.io/open-to-buy/
```
