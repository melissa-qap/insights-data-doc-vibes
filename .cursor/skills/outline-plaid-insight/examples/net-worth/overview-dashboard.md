# Overview dashboard

### Description

Composite overview screen: net worth headline and chart, assets/liabilities summary bar, and account list grouped by Cash, Investment, Credit cards, and Loans. Client composition uses three read APIs from [net worth APIs](../../design-api/examples/net-worth/net-worth-apis.md); shared calculation: [net worth core](net-worth-core.md).

### Required input data

#### `plaid_accounts`

Same columns as `GET /v1/account-balance` — historical rows for performance-history; latest snapshot for account-balance and assets-liabilities.

#### `plaid_items` *(extension)*

| Column | Description |
|---|---|
| `item_id` | Join key — matches `plaid_accounts.item_id` |
| `institution_name` | Institution display label per account row |

**Input:** `user_id = ?`. All historical account rows (same scope as performance-history). Enrichment applied server-side in account-balance response.

#### `plaid_investment_holdings`

Used only for investment account balance fallback when `balances_current` IS NULL — same rule as [investment accounts by institution](../investment-account/investment-accounts-by-institution.md).

**Parameters:**

| Parameter | Type | Default | Notes / options |
|---|---|---|---|
| `timeframe` | enum | `trailing_1y` | `trailing_1m`, `trailing_3m`, `ytd`, `trailing_1y`, `all_time`; passed to `GET /v1/performance-history`. Aliases: `1M`→`trailing_1m`, `3M`→`trailing_3m`, `YTD`→`ytd`, `1Y`→`trailing_1y`, `All`→`all_time` |

### Calculation / analysis

1. **Load accounts** — fetch all `plaid_accounts` rows for user (historical syncs); shared by all three read APIs via [net worth core](net-worth-core.md)
2. **Performance chart** — `GET /v1/performance-history?timeframe=…` (net worth mode, no `account_ids`)
3. **Account list** — `GET /v1/account-balance` (no filter); client groups `accounts[]` by `group` (`cash`, `investment`, `credit_cards`, `loans`) and sums `balance` per section for header subtotals
4. **Display labels** — per account row from account-balance: `{name} • {mask}` when `mask` present; else `name` (prefer over `official_name`); sort within each group by `balance` descending
5. **Assets / liabilities bar** — `GET /v1/assets-liabilities` at latest sync; client section subtotals must match `total_assets` / `total_liabilities`; headline from `performance-history.points[last].value` must equal `assets-liabilities.net_worth`
6. **Headline** — `performance-history.points[last].value`; YTD label uses second call with `timeframe=ytd` → `period_return_amount`, `period_return_pct` (or reuse if main timeframe is `ytd`)

### Data output

Client-side composition — not a single server BFF. See [client composition](../../design-api/examples/net-worth/net-worth-apis.md#client-composition).

| Source API | Fields used |
|---|---|
| `GET /v1/performance-history` | `points[]`, `period_return_amount`, `period_return_pct`, `timeframe`, `as_of` |
| `GET /v1/account-balance` | `accounts[]` — client groups by `group`, sums `balance` for section headers |
| `GET /v1/assets-liabilities` | `segments[]`, `total_assets`, `total_liabilities`, `net_worth`, `bar_total`, `as_of` |
