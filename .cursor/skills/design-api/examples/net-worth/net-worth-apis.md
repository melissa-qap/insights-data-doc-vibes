# Net worth APIs

### Description

API surface for the net worth overview screen: balance sync, account discovery, performance line chart, account balance list, and assets/liabilities stacked bar. Generic read routes (`/performance-history`, `/account-balance`, `/assets-liabilities`) are not net-worth-prefixed — reusable across domains.

**Formatting:** Apply [output formatting](../../../outline-plaid-insight/SKILL.md#output-formatting) — dollar fields 2 dp; fraction percent fields 3 dp.

### Shared calculation

All read endpoints use [net worth core](../../../outline-plaid-insight/examples/net-worth/net-worth-core.md) — `resolve_accounts_as_of` + `compute_net_worth`.

---

### Sync APIs

#### POST /v1/plaid/sync/balances

- **Plaid source:** `/accounts/balance/get` (primary) or `/accounts/get` (alternate) — see [plaid-api-schema.md](../../../outline-plaid-insight/plaid-api-schema.md#accounts--balance)
- **Datatable writes:** `plaid_accounts` — one row per account per sync; retain history (`synced_at` per import). Each row is a direct 1:1 map of one Plaid `accounts[]` element (no derived or renamed fields) — see [plaid_accounts](../../../outline-plaid-insight/plaid-api-schema.md#plaid_accounts).
- **Request:**
  - `item_id` (optional) — scope sync to one Plaid Item; omit for all Items for user
- **Response:**

| Field | Type | Description |
|---|---|---|
| `synced_at` | timestamp | Import timestamp for this sync batch |
| `accounts_synced` | number | Count of accounts written |
| `accounts[]` | array | Rows written in this sync — same shape as `plaid_accounts` (omit `user_id`). See `accounts[]` fields below |

**`accounts[]` fields** — echo of `plaid_accounts` for this `synced_at`:

| Field | Type | Description |
|---|---|---|
| `account_id` | string | Plaid account ID |
| `name` | string | Display name |
| `official_name` | string \| null | Institution official name |
| `mask` | string \| null | Last 2–4 digits |
| `type` | string | Account type (e.g. `investment`, `depository`) |
| `subtype` | string \| null | Account subtype (e.g. `401k`, `brokerage`) |
| `holder_category` | string \| null | `personal`, `business`, `unrecognized` (from Plaid; often null) |
| `balances_available` | number \| null | `balances.available` |
| `balances_current` | number \| null | `balances.current` — total balance; investment = total asset value |
| `balances_limit` | number \| null | `balances.limit` |
| `balances_iso_currency_code` | string \| null | ISO-4217 currency |
| `balances_unofficial_currency_code` | string \| null | Non-ISO currency |
| `balances_last_updated_datetime` | string \| null | Balance freshness timestamp (institution-specific) |
| `balances_margin_loan_amount` | number \| null | Always `null` on balance sync — not returned by `/accounts/balance/get`; populated only via optional enrichment from `/investments/holdings/get` on holdings sync |
| `item_id` | string | Plaid Item ID (from response `item.item_id`) |
| `synced_at` | timestamp | Same as top-level `synced_at` for this row |

- **Powers:** Background refresh before read APIs; populates balance history for charts; account discovery and list without a separate read call (e.g. investment page filters `type = investment` client-side)

**Daily schedule:** Run once per calendar day per user (or per Item) via a background job — not on read-path. For each active Plaid Item, call `/accounts/balance/get` with that Item's `access_token`, map `accounts[]` into `plaid_accounts`, and set `synced_at` to the import timestamp. Omit `item_id` in the scheduled run to refresh all Items for the user in one job pass. Each daily run appends a new snapshot row per account so `GET /v1/performance-history` can build a daily series; read APIs always use the latest `synced_at` unless `as_of` is passed.

| Concern | Rule |
|---|---|
| **Frequency** | Once daily per user (all Items); do not call Plaid on every read |
| **Plaid endpoint** | Prefer `/accounts/balance/get` for real-time balances; `/accounts/get` only if cost or institution latency is a constraint |
| **Scope** | One Plaid call per Item; job iterates `plaid_items` where Item is active |
| **Idempotency** | Multiple runs same day are allowed — each writes a distinct `synced_at`; chart uses end-of-day resolution |
| **Failure** | Per-Item errors do not block other Items; retry failed Items with backoff; stale balances served from last successful sync |
| **On-demand** | Same endpoint may be invoked manually (e.g. user pull-to-refresh); rate-limit to avoid duplicate Plaid calls within a short window |

**Investment holdings (cross-ref):** For users with investment Items, run `POST /v1/plaid/sync/investment-holdings` after balance sync in the same post-market daily job — see [investment-account-apis.md § investment-holdings](../investment-account/investment-account-apis.md#post-v1plaidsyncinvestment-holdings). Populates `plaid_investment_holdings`, `plaid_investment_securities`, and `plaid_accounts.balances_margin_loan_amount`.

---

### Read APIs

#### GET /v1/accounts

- **Parameters:** none (scoped by auth `user_id`)
- **Calculation:** Latest metadata row per `account_id` from `plaid_accounts` at `MAX(synced_at)` — ids and display fields only; **no balance fields**
- **Response:**

| Field | Type | Description |
|---|---|---|
| `accounts[]` | array | `{ account_id, name, mask, official_name, type, subtype, item_id }` |

- **Powers:** Account pickers, lightweight discovery — **no balance fields**. Use `GET /v1/account-balance` when balances or `a_l`/`group` classification are needed.

| Endpoint | Returns | Use when |
|---|---|---|
| `GET /v1/accounts` | ids + metadata, no balance | Pickers, routing, ID discovery before balance load |
| `GET /v1/account-balance` | accounts with balance + `a_l`/`group` | Any screen showing balances; filterable via `account_ids[]` |

#### GET /v1/performance-history

- **Parameters:**

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `timeframe` | enum | `trailing_1y` | `trailing_1m`, `trailing_3m`, `ytd`, `trailing_1y`, `all_time` |
| `account_ids` | `string[]` | omitted | Omitted or empty → net worth mode (all accounts). Non-empty → signed net of selected accounts only |

**Timeframe aliases:** `1M` → `trailing_1m`; `3M` → `trailing_3m`; `YTD` → `ytd`; `1Y` → `trailing_1y`; `All` → `all_time`.

- **Calculation:** [net worth core](../../../outline-plaid-insight/examples/net-worth/net-worth-core.md) — historical `plaid_accounts` rows; daily `resolve_accounts_as_of(user_id, end_of_day(D))` + `compute_net_worth(...)` for each calendar day `D` in window:

  1. **Validate `account_ids`** *(when non-empty)* — resolve at `MAX(synced_at)`; reject if any ID not in set (do not silently drop)
  2. **`window_end`** — calendar date of `MAX(synced_at)` for user
  3. **`window_start` from `timeframe`** — `trailing_1m`: 30 days before `window_end`; `trailing_3m`: 90 days; `ytd`: Jan 1 of `window_end` year; `trailing_1y`: 12 months; `all_time`: earliest day any in-scope account has a row on or before end of day
  4. **Daily series** — for each `D` from `window_start` through `window_end`: resolve accounts; filter to `account_ids` when non-empty; `value = net_worth`; emit `{ date, value }` when at least one in-scope account has data; omit empty days
  5. **Period return** — always for the selected `timeframe`: `period_return_amount = points[last].value − points[0].value`; `period_return_pct = period_return_amount / points[0].value` when `points[0].value > 0`, else omit
  6. **`as_of`** — same as last point date (`points[last].date`)

- **Response:** *(no `window_start` / `window_end` — infer series range from `points[0].date` and `points[last].date`)*

| Field | Type | Description |
|---|---|---|
| `account_ids` | `string[]` \| null | Echo requested IDs when non-empty; else `null` |
| `timeframe` | string | Requested window |
| `points` | array | `{ date, value }`, sorted ascending |
| `period_return_amount` | number | Dollar change over selected `timeframe` |
| `period_return_pct` | number | Fraction (0–1); omit if start value ≤ 0 |
| `as_of` | date | Last point date |

- **Powers:** Net worth line chart + period change label; account-detail performance (pass `account_ids`); investment portfolio chart (pass investment `account_ids` — see [investment account APIs § client composition](../investment-account/investment-account-apis.md#client-composition) and [investment performance chart](../../../outline-plaid-insight/examples/investment-account/investment-performance-chart.md))
- **Invariants:** Net worth mode — `points[last].value` must equal `GET /v1/assets-liabilities` `net_worth` at same cutoff. Accounts mode (all investment IDs) — `points[last].value` must equal investment performance `end_value`.
- **Notes:** Carry-forward between syncs; holding-period return (not TWR). **Not available:** benchmark overlay, TWR/MWR, holdings-based historical value.

#### GET /v1/account-balance

- **Parameters:**

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `account_ids` | `string[]` | omitted | Omitted or empty → all accounts. Non-empty → filter to matching IDs; reject unknown IDs |
| `as_of` | date | latest sync | Optional cutoff; defaults to `MAX(synced_at)` |

- **Calculation:** [net worth core](../../../outline-plaid-insight/examples/net-worth/net-worth-core.md) at `cutoff = end_of_day(as_of)` — `resolve_accounts_as_of` + `compute_net_worth(...)` → per-account list with `a_l` and `group`:

  1. **`cutoff`** — `end_of_day(as_of)` or `MAX(synced_at)` when omitted
  2. **Validate `account_ids`** *(when non-empty)* — resolve full set at cutoff; reject if any ID not in set (do not silently drop)
  3. **Resolve + classify** — core Layer 2–3; use `accounts[]` from output (ignore portfolio totals)
  4. **Filter** *(when `account_ids` non-empty)* — keep only matching rows
  5. **Enrich** — join `plaid_items` on `item_id` → `institution_name` (fallback `"Unknown institution"`); add `mask`, `official_name` from `plaid_accounts`; investment accounts with null `balance`: sum `plaid_investment_holdings.institution_value` at same `synced_at`

- **Response:**

| Field | Type | Description |
|---|---|---|
| `as_of` | timestamp | `MAX(synced_at)` across returned accounts |
| `accounts[]` | array | `{ account_id, name, mask, official_name, type, subtype, balance, a_l, group, institution_name, synced_at }` |

Server owns classification (`a_l`, `group`); client groups by `group` and sums `balance` for section headers. Do not re-derive `group` from `type`/`subtype` on the client.

- **Client rollup** *(when no `account_ids` filter)* — section total for a group = sum of `balance` where `group` matches; portfolio net worth from rows = `sum(balance where a_l=asset) − sum(balance where a_l=liability)` — must equal `performance-history.points[last].value` and `assets-liabilities.net_worth` at same cutoff.
- **Powers:** Account list (overview — client groups into Cash / Investment / Credit cards / Loans); account detail header/balance (`account_ids[]=<id>`); `account_ids` discovery for performance-history filter; investment overview flat account list (filter `group = investment`, sort by `balance` desc — see [investment account APIs § client composition](../investment-account/investment-account-apis.md#client-composition))

#### GET /v1/assets-liabilities

- **Parameters:**

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `as_of` | date | latest sync | Optional; same cutoff semantics as account-balance |

- **Calculation:** [net worth core](../../../outline-plaid-insight/examples/net-worth/net-worth-core.md) at cutoff — `compute_net_worth(...)` → totals only:

  1. **`cutoff`** — same as account-balance
  2. **Bar total** — `bar_total = total_assets + total_liabilities` (gross exposure; not net worth)
  3. **Segments** — derive from totals only:
     - **Assets:** `value = total_assets`; `percent = value / bar_total` when `bar_total > 0`, else `0`
     - **Liabilities:** `value = total_liabilities`; same percent rule
     - Omit segment when `value = 0`
  4. **Validation** — `net_worth = total_assets − total_liabilities`; segment percents sum to ~1.0 (±0.002) when both present

- **Response:**

| Field | Type | Description |
|---|---|---|
| `total_assets` | number | Depository + investment |
| `total_liabilities` | number | Credit + loan (positive dollars owed) |
| `net_worth` | number | For headline pairing; not used for segment widths |
| `bar_total` | number | `total_assets + total_liabilities` |
| `segments` | array | `[assets, liabilities]` when non-zero: `{ segment, label, value, percent }` |
| `as_of` | timestamp | Portfolio freshness |

- **Powers:** Horizontal stacked bar — assets vs liabilities as shares of gross exposure
- **Empty state:** `bar_total = 0` → `segments = []`

---

### Client composition

**Net worth page load** — three parallel read calls (one performance call):

| Call | Purpose |
|---|---|
| `GET /v1/performance-history?timeframe=…` | Line chart + headline `points[last].value` + period return |
| `GET /v1/account-balance` | Flat account list — client groups by `group`, sums `balance` for section headers |
| `GET /v1/assets-liabilities` | Stacked bar |

**Account detail page** — two calls:

| Call | Purpose |
|---|---|
| `GET /v1/account-balance?account_ids[]=<id>` | Header balance, `a_l`, `group`, institution metadata |
| `GET /v1/performance-history?account_ids[]=<id>&timeframe=…` | Performance chart |

**Invariants across responses at latest sync** *(overview, no `account_ids` filter on account-balance)*:

- `performance-history.points[last].value` = `assets-liabilities.net_worth`
- `sum(account-balance.accounts where a_l=asset) − sum(... where a_l=liability)` = `performance-history.points[last].value`
- `sum(account-balance.accounts where a_l=asset)` = `assets-liabilities.total_assets`
- `sum(account-balance.accounts where a_l=liability)` = `assets-liabilities.total_liabilities`

**YTD headline label:** second `performance-history` call with `timeframe=ytd` for `period_return_amount` / `period_return_pct` (or reuse if main chart timeframe is already `ytd`).

**Overview dashboard insight:** [overview-dashboard.md](../../../outline-plaid-insight/examples/net-worth/overview-dashboard.md) documents enrichment and composite validation — maps to these three endpoints, not a server BFF.

**Optimization:** Backend may share one historical `plaid_accounts` fetch across the three read handlers in a single page request; client still calls three routes.
