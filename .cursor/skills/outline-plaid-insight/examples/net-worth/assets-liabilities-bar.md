# Assets / liabilities bar

### Description

Latest snapshot horizontal stacked bar comparing total assets and total liabilities as shares of gross exposure (`total_assets + total_liabilities`). Legend shows dollar values per segment. Pair with [net worth balance chart](net-worth-balance-chart.md) for the net worth headline — bar segments are not sized by net worth.

Uses [net worth core](net-worth-core.md) with latest-only data access; `include_groups = false`.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Unique account identifier |
| `type` | Account type (`depository`, `credit`, `investment`, `loan`, `other`) |
| `subtype` | Account subtype |
| `balances_current` | Current balance — used for net worth |
| `synced_at` | Per-account sync timestamp |

**Input:** `user_id = ?`. Latest snapshot (`cutoff = MAX(synced_at)`). One row per `account_id` via [net worth core](net-worth-core.md) Layer 2. Exclude null `balances_current`.

### Calculation / analysis

1. **`cutoff`** — `MAX(synced_at)` for user
2. **Resolve accounts** — [net worth core](net-worth-core.md) Layer 2: `resolve_accounts_as_of(user_id, cutoff)`
3. **Compute totals** — [net worth core](net-worth-core.md) Layer 3: `compute_net_worth(accounts, include_groups = false)` → `total_assets`, `total_liabilities`, `net_worth`, `as_of`
4. **Bar total** — `bar_total = total_assets + total_liabilities` (gross exposure; not `net_worth`)
5. **Build segments** — derive from totals only (do not re-sum accounts independently):
   - **Assets:** `value = total_assets`; `percent = value / bar_total` when `bar_total > 0`, else `0`
   - **Liabilities:** `value = total_liabilities`; `percent = value / bar_total` when `bar_total > 0`, else `0`
   - Omit a segment when its `value = 0` (single-segment bar when only assets or only liabilities)
6. **Validation** — assert `net_worth = total_assets − total_liabilities`; assert `segments[].percent` sums to `1.0` (±0.001) when both segments present

### Data output

| Field | Type | Description |
|---|---|---|
| `total_assets` | number | Sum of depository + investment balances |
| `total_liabilities` | number | Sum of credit + loan balances (positive dollars owed) |
| `net_worth` | number | `total_assets − total_liabilities` — for headline pairing / validation; not used for segment widths |
| `bar_total` | number | `total_assets + total_liabilities` — bar denominator |
| `segments` | array | Ordered `[assets, liabilities]` when both non-zero: `{ segment, label, value, percent }` |
| `segments[].segment` | string | `assets` or `liabilities` |
| `segments[].label` | string | Display label: `"Assets"` / `"Liabilities"` |
| `segments[].value` | number | Dollar amount for legend |
| `segments[].percent` | number | Share of `bar_total` as fraction (0–1); multiply by 100 for optional % display |
| `as_of` | timestamp | `MAX(synced_at)` across resolved accounts |

**Empty state:** when `bar_total = 0` (no linked accounts or all null balances), return `segments = []` and omit bar.

### UI output

**Pattern:** [Assets / liabilities bar — stacked bar](../../ui-output-options.md#assets--liabilities-bar--stacked-bar)
