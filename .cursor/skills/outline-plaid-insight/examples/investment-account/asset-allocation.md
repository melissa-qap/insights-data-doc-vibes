# Asset allocation across investment accounts

### Description

Shows portfolio-wide asset allocation (equity, bonds, cash, crypto, international equity) across all linked investment accounts as of the latest holdings snapshot. ETFs and mutual funds are decomposed using an enrichment table; direct holdings use Plaid security metadata. Each asset class includes a holdings drill-down with class-attributed values.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Account identifier — filter to investment accounts |
| `type` | Account type — must be `investment` |
| `synced_at` | Snapshot timestamp — align with holdings sync |

**Input:** `user_id = ?`. Latest snapshot (`synced_at = MAX` for user). `type = investment`. Collect `account_id` list for holdings filter.

#### `plaid_investment_holdings`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Investment account holding the position |
| `security_id` | Links to security metadata and enrichment |
| `institution_value` | Current market value — basis for allocation |
| `unofficial_currency_code` | Non-ISO currency — used for crypto detection |
| `synced_at` | Snapshot timestamp — must match securities join |

**Input:** `user_id = ?`. Latest snapshot (same `synced_at` as `plaid_accounts`). `account_id` in investment account list. Exclude rows where `institution_value` IS NULL.

#### `plaid_investment_securities`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `security_id` | Join key to holdings and enrichment |
| `name` | Security display name |
| `ticker_symbol` | Ticker for display |
| `type` | Security type (`equity`, `etf`, `mutual fund`, `fixed income`, `cash`, etc.) |
| `isin` | ISIN — country prefix used for domestic vs international equity |
| `synced_at` | Must match holdings `synced_at` on join |

**Input:** `user_id = ?`. Latest snapshot. Join to holdings on `security_id` + `synced_at`.

#### `security_asset_allocation` (extension — not Plaid)

| Column | Description |
|---|---|
| `security_id` | Join to `plaid_investment_securities.security_id` |
| `asset_class` | `equity`, `bonds`, `cash`, `crypto`, or `international_equity` |
| `weight` | Share of security market value in this class (0–1) |
| `as_of` | Optional composition freshness date |

**Input:** Rows for `security_id` values present in the user's holdings. For each `security_id`, weights must sum to `1.0` (±0.001). Required for `etf`, `mutual fund`, and any security where Plaid `type` alone is insufficient.

**Not available in current Plaid schema:** Fund-level composition. Plaid exposes instrument `type` only (e.g. `etf`), not underlying allocation. The enrichment table is mandatory for accurate ETF/MF allocation.

### Calculation / analysis

1. **Load snapshot timestamp**
   - Latest snapshot from `plaid_accounts` for the user
2. **Load investment accounts**
   - All `account_id` values where `type = investment`
3. **Load holdings and enrichment**
   - `plaid_investment_holdings` for those accounts at that `synced_at`
   - Join `plaid_investment_securities` on `security_id` + `synced_at`
   - Load `security_asset_allocation` for held `security_id` values
4. **Classify each holding**
   - `institution_value` = V
   - Into attribution slices `{ security_id, asset_class, attributed_value }`:
     - If `security_id` has enrichment rows: for each row, `attributed_value = V × weight`
     - Else if `holdings.unofficial_currency_code IS NOT NULL`: one slice, `asset_class = crypto`, `attributed_value = V`
     - Else if `type = fixed income`: `bonds`, V
     - Else if `type = cash`: `cash`, V
     - Else if `type = equity` AND (`isin` IS NULL OR left 2 chars of `isin` = `US`): `equity`, V
     - Else if `type = equity` AND `isin` country prefix ≠ `US`: `international_equity`, V
     - Else if `type` in (`etf`, `mutual fund`, `derivative`, `other`) with no enrichment: add V to `unclassified_value`; emit no slices
5. **Roll up by `asset_class`**
   - Portfolio-wide, classified only
   - `value` = sum of `attributed_value`
   - Derive `total_classified_value` from classified class rows (do not compute independently)
6. **Holdings drill-down per class**
   - Group slices by `security_id`
   - `value` = sum of attributed slices in that class
   - Use security `name`; fall back to `ticker_symbol` if null
   - Sort holdings within class by `value` descending
   - A single ETF may appear under multiple classes with partial values
7. **Unclassified**
   - Collect holdings with no classification path into `unclassified_holdings[]`
   - `unclassified_value` = sum of their `institution_value`
   - `total_investment_value` = `total_classified_value + unclassified_value`
8. **Other class**
   - If `unclassified_value > 0`, append `{ asset_class: other, value: unclassified_value, holdings: unclassified_holdings }` to `asset_classes`
   - Omit when zero
9. **Percents**
   - For each `asset_classes` entry (including `other`), `percent` = `value / total_investment_value` (0 if denominator 0)
   - Sort `asset_classes` by `value` descending
10. **`as_of`**
    - Snapshot `synced_at`

**Heuristic limitations:** `international_equity` for single stocks uses ISIN country prefix only. Crypto detection depends on `unofficial_currency_code` and institution labeling — not all crypto positions may be identified.

### Data output

| Field | Type | Description |
|---|---|---|
| `asset_classes` | array | One entry per class with non-zero value: `{ asset_class, value, percent, holdings }` — sorted by `value` descending; includes `other` when unclassified holdings exist |
| `asset_classes[].asset_class` | string | `equity`, `bonds`, `cash`, `crypto`, `international_equity`, or `other` (unclassified funds) |
| `asset_classes[].value` | number | Class total — sum of `holdings[].value` |
| `asset_classes[].percent` | number | Share of `total_investment_value` as a fraction (0–1); all class percents sum to 1.0 when `other` is present |
| `asset_classes[].holdings` | array | `{ security_id, name, ticker_symbol, value }` — class-attributed portion for classified classes; full `institution_value` for `other` |
| `total_classified_value` | number | Sum of classified class values (excludes `other`) |
| `unclassified_value` | number | Same as `other` entry `value` when present; 0 otherwise |
| `unclassified_holdings` | array | Same as `other` entry `holdings` when present; empty otherwise |
| `total_investment_value` | number | `total_classified_value + unclassified_value` |
| `as_of` | timestamp | `synced_at` of the snapshot used |

### UI output

**Pattern:** [Asset allocation — nested list](../../ui-output-options.md#asset-allocation--nested-list) — portfolio root with class groups and holdings nested below, including **Other** for unclassified positions (not a footnote). Each asset-class row supports a **% / $ toggle** for its displayed value (`percent` vs `value`); holding rows always show dollar `value`.
