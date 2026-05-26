# Plaid API Response Schema — Reference

Key fields from each Plaid endpoint used as data sources for insights. Only insight-relevant fields are listed; auth/request fields omitted.

**Assumption:** All Plaid API responses are already imported into the datatables below. Insight logic queries these tables — never call Plaid APIs at runtime.

---

## Data Tables

Each endpoint maps to one or more tables. Nested response fields are flattened with underscores (e.g. `balances.current` → `balances_current`).

### Common columns (all tables)

| Column | Type | Notes |
|---|---|---|
| `user_id` | string | User scope — filter all queries by this |
| `item_id` | string | Plaid Item that produced this row |
| `synced_at` | timestamp | When this row was imported from Plaid |

For tables that accumulate history across syncs, keep one row per entity **per sync** (do not overwrite prior imports). Use `synced_at` to select current vs historical snapshots.

### Table index

| Endpoint | Table(s) |
|---|---|
| `/accounts/balance/get` | `plaid_accounts` |
| `/transactions/sync` | `plaid_transactions`, `plaid_transactions_removed` |
| `/investments/holdings/get` | `plaid_investment_holdings`, `plaid_investment_securities` |
| `/investments/transactions/get` | `plaid_investment_transactions` |
| `/liabilities/get` | `plaid_liabilities_credit`, `plaid_liabilities_student`, `plaid_liabilities_mortgage` |
| *(extension)* | `security_asset_allocation`, `plaid_items` |

---

## Extension tables

Tables maintained outside Plaid imports. Used when Plaid fields are insufficient for an insight.

### `plaid_items`

Source: Plaid Item metadata at link time (e.g. from `/link/token/create` callback or Items API). One row per Item per user.

| Column | Type | Notes |
|---|---|---|
| `item_id` | string | Join to `item_id` on all Plaid datatables |
| `institution_name` | string | Display name for the linked institution |
| `institution_id` | string \| null | Optional Plaid institution ID (e.g. `ins_3`) |

Used by [investment accounts by institution](examples/investment-account/investment-accounts-by-institution.md) to group accounts under institution labels. `item_id` alone is on every datatable row; institution names are not.

### `security_asset_allocation`

Source: external fund-composition data (not a Plaid endpoint). One or more rows per security.

| Column | Type | Notes |
|---|---|---|
| `security_id` | string | Join to `plaid_investment_securities.security_id` |
| `asset_class` | string | `equity`, `bonds`, `cash`, `crypto`, `international_equity` |
| `weight` | number | Share of security market value in this class (0–1) |
| `as_of` | date \| null | Optional composition freshness |

**Constraints:** For each `security_id`, weights must sum to `1.0` (±0.001). Used by [asset allocation](examples/investment-account/asset-allocation.md) to decompose ETFs and mutual funds.

---

### `plaid_accounts`

Source: `/accounts/balance/get` → `accounts[]`. One row per account per sync.

| Column | Type | Source field |
|---|---|---|
| `account_id` | string | `account_id` |
| `name` | string | `name` |
| `official_name` | string \| null | `official_name` |
| `mask` | string \| null | `mask` |
| `type` | string | `type` |
| `subtype` | string \| null | `subtype` |
| `balances_available` | number \| null | `balances.available` |
| `balances_current` | number \| null | `balances.current` |
| `balances_limit` | number \| null | `balances.limit` |
| `balances_iso_currency_code` | string \| null | `balances.iso_currency_code` |
| `balances_unofficial_currency_code` | string \| null | `balances.unofficial_currency_code` |
| `balances_last_updated_datetime` | string \| null | `balances.last_updated_datetime` |

**Query patterns:**
- Current state: `WHERE user_id = ? AND synced_at = (SELECT MAX(synced_at) FROM plaid_accounts WHERE user_id = ?)`
- Point-in-time: `WHERE user_id = ? AND synced_at <= ?` then take latest row per `account_id`

---

### `plaid_transactions`

Source: `/transactions/sync` → `added[]` / `modified[]`. One row per transaction (upsert on `transaction_id`; retain latest `synced_at`).

| Column | Type | Source field |
|---|---|---|
| `transaction_id` | string | `transaction_id` |
| `account_id` | string | `account_id` |
| `amount` | number | `amount` |
| `date` | date | `date` |
| `authorized_date` | date \| null | `authorized_date` |
| `name` | string | `name` |
| `merchant_name` | string \| null | `merchant_name` |
| `pending` | boolean | `pending` |
| `payment_channel` | string | `payment_channel` |
| `personal_finance_category_primary` | string | `personal_finance_category.primary` |
| `personal_finance_category_detailed` | string | `personal_finance_category.detailed` |
| `personal_finance_category_confidence_level` | string | `personal_finance_category.confidence_level` |
| `location_city` | string \| null | `location.city` |
| `location_region` | string \| null | `location.region` |
| `iso_currency_code` | string \| null | `iso_currency_code` |
| `unofficial_currency_code` | string \| null | `unofficial_currency_code` |

---

### `plaid_transactions_removed`

Source: `/transactions/sync` → `removed[]`. One row per removed transaction per sync.

| Column | Type | Source field |
|---|---|---|
| `transaction_id` | string | `transaction_id` |

Join or exclude against `plaid_transactions` when building the active transaction set.

---

### `plaid_investment_holdings`

Source: `/investments/holdings/get` → `holdings[]`. One row per holding per sync.

| Column | Type | Source field |
|---|---|---|
| `account_id` | string | `account_id` |
| `security_id` | string | `security_id` |
| `quantity` | number | `quantity` |
| `cost_basis` | number \| null | `cost_basis` |
| `institution_value` | number \| null | `institution_value` |
| `institution_price` | number \| null | `institution_price` |
| `institution_price_as_of` | date \| null | `institution_price_as_of` |
| `iso_currency_code` | string \| null | `iso_currency_code` |
| `unofficial_currency_code` | string \| null | `unofficial_currency_code` |

---

### `plaid_investment_securities`

Source: `/investments/holdings/get` → `securities[]`. One row per security per sync.

| Column | Type | Source field |
|---|---|---|
| `security_id` | string | `security_id` |
| `name` | string \| null | `name` |
| `ticker_symbol` | string \| null | `ticker_symbol` |
| `type` | string | `type` |
| `isin` | string \| null | `isin` |
| `close_price` | number \| null | `close_price` |
| `close_price_as_of` | date \| null | `close_price_as_of` |
| `iso_currency_code` | string \| null | `iso_currency_code` |

Join to `plaid_investment_holdings` on `security_id` + matching `synced_at`.

---

### `plaid_investment_transactions`

Source: `/investments/transactions/get` → `investment_transactions[]`. One row per transaction (upsert on `investment_transaction_id`).

| Column | Type | Source field |
|---|---|---|
| `investment_transaction_id` | string | `investment_transaction_id` |
| `account_id` | string | `account_id` |
| `security_id` | string \| null | `security_id` |
| `date` | date | `date` |
| `name` | string | `name` |
| `quantity` | number | `quantity` |
| `amount` | number | `amount` |
| `price` | number | `price` |
| `fees` | number \| null | `fees` |
| `type` | string | `type` |
| `subtype` | string | `subtype` |
| `iso_currency_code` | string \| null | `iso_currency_code` |
| `unofficial_currency_code` | string \| null | `unofficial_currency_code` |

---

### `plaid_liabilities_credit`

Source: `/liabilities/get` → `liabilities.credit[]`. One row per credit account per sync.

| Column | Type | Source field |
|---|---|---|
| `account_id` | string | `account_id` |
| `aprs` | JSON | `aprs[]` |
| `is_overdue` | boolean \| null | `is_overdue` |
| `last_payment_amount` | number \| null | `last_payment_amount` |
| `last_payment_date` | date \| null | `last_payment_date` |
| `last_statement_issue_date` | date \| null | `last_statement_issue_date` |
| `last_statement_balance` | number \| null | `last_statement_balance` |
| `minimum_payment_amount` | number \| null | `minimum_payment_amount` |
| `next_payment_due_date` | date \| null | `next_payment_due_date` |

---

### `plaid_liabilities_student`

Source: `/liabilities/get` → `liabilities.student[]`. One row per student loan per sync.

| Column | Type | Source field |
|---|---|---|
| `account_id` | string | `account_id` |
| `interest_rate_percentage` | number | `interest_rate_percentage` |
| `is_overdue` | boolean \| null | `is_overdue` |
| `last_payment_amount` | number \| null | `last_payment_amount` |
| `last_payment_date` | date \| null | `last_payment_date` |
| `loan_name` | string \| null | `loan_name` |
| `loan_status_type` | string | `loan_status.type` |
| `minimum_payment_amount` | number \| null | `minimum_payment_amount` |
| `next_payment_due_date` | date \| null | `next_payment_due_date` |
| `origination_date` | date \| null | `origination_date` |
| `origination_principal_amount` | number \| null | `origination_principal_amount` |
| `outstanding_interest_amount` | number \| null | `outstanding_interest_amount` |

---

### `plaid_liabilities_mortgage`

Source: `/liabilities/get` → `liabilities.mortgage[]`. One row per mortgage per sync.

| Column | Type | Source field |
|---|---|---|
| `account_id` | string | `account_id` |
| `interest_rate_percentage` | number \| null | `interest_rate.percentage` |
| `interest_rate_type` | string \| null | `interest_rate.type` |
| `last_payment_amount` | number \| null | `last_payment_amount` |
| `last_payment_date` | date \| null | `last_payment_date` |
| `loan_term` | string \| null | `loan_term` |
| `minimum_monthly_payment` | number \| null | `minimum_monthly_payment` |
| `next_payment_due_date` | date \| null | `next_payment_due_date` |
| `origination_date` | date \| null | `origination_date` |
| `origination_principal_amount` | number \| null | `origination_principal_amount` |
| `past_due_amount` | number \| null | `past_due_amount` |
| `ytd_interest_paid` | number \| null | `ytd_interest_paid` |
| `ytd_principal_paid` | number \| null | `ytd_principal_paid` |

---

## API Field Reference

Below is the original Plaid response field documentation. Column names in the tables above map directly to these fields.

---

## `/accounts/balance/get`

Returns real-time balance data for all accounts linked to an Item.

**`accounts[]`**

| Field | Type | Notes |
|---|---|---|
| `account_id` | string | Unique account identifier |
| `name` | string | Account name (user-assigned or institution-assigned) |
| `official_name` | string \| null | Official name from the institution |
| `mask` | string \| null | Last 2–4 digits of account number |
| `type` | string | `depository`, `credit`, `investment`, `loan`, `other` |
| `subtype` | string \| null | e.g. `checking`, `savings`, `credit card`, `mortgage`, `401k` |
| `balances.available` | number \| null | Funds available to withdraw. Null if institution doesn't provide it. |
| `balances.current` | number \| null | Total funds in or owed by account. For credit: amount owed. For loans: principal remaining. |
| `balances.limit` | number \| null | Credit limit (credit accounts) or overdraft limit (depository) |
| `balances.iso_currency_code` | string \| null | ISO-4217 currency code. Null if `unofficial_currency_code` is set. |
| `balances.unofficial_currency_code` | string \| null | For crypto or non-ISO currencies. Null if `iso_currency_code` is set. |
| `balances.last_updated_datetime` | string \| null | ISO 8601 timestamp. Only returned for Capital One (`ins_128026`). |

**Notes:**
- Use `available` for spending power; use `current` for net worth calculations
- `available` may be null — always check before using
- Balance data from this endpoint is real-time; other endpoints may return cached balances

---

## `/transactions/sync`

Returns incremental transaction updates using a cursor. Preferred over `/transactions/get` for ongoing sync.

**`added[]` / `modified[]`** (transactions)

| Field | Type | Notes |
|---|---|---|
| `transaction_id` | string | Unique transaction identifier |
| `account_id` | string | Account this transaction belongs to |
| `amount` | number | Positive = money out (expense/debit). Negative = money in (credit/refund). |
| `date` | string | Posted date (YYYY-MM-DD). Use `authorized_date` if more precision needed. |
| `authorized_date` | string \| null | Date transaction was authorized (may differ from posted date) |
| `name` | string | Raw transaction name from institution |
| `merchant_name` | string \| null | Cleaned merchant name (preferred over `name` for display) |
| `pending` | boolean | True if transaction has not yet posted. Exclude from most analysis. |
| `payment_channel` | string | `online`, `in store`, `other` |
| `personal_finance_category.primary` | string | Top-level category e.g. `FOOD_AND_DRINK`, `INCOME`, `TRANSFER_IN` |
| `personal_finance_category.detailed` | string | Sub-category e.g. `FOOD_AND_DRINK_RESTAURANTS` |
| `personal_finance_category.confidence_level` | string | `VERY_HIGH`, `HIGH`, `MEDIUM`, `LOW`, `UNKNOWN` |
| `location.city` | string \| null | City where transaction occurred |
| `location.region` | string \| null | State/region |
| `iso_currency_code` | string \| null | ISO-4217 currency code |
| `unofficial_currency_code` | string \| null | For non-ISO currencies |

**`removed[]`**

| Field | Type | Notes |
|---|---|---|
| `transaction_id` | string | ID of transaction to remove from local store |

**Pagination fields**

| Field | Type | Notes |
|---|---|---|
| `next_cursor` | string | Pass in next request to get only new updates |
| `has_more` | boolean | True if more updates are available — keep paginating |

**Notes:**
- `amount` sign convention: positive = money out (buy/expense), negative = money in (sell/dividend). Same direction as `/transactions/sync`.
- Always paginate until `has_more = false`
- `personal_finance_category` is the v2 taxonomy field; replaces legacy `category` array

---

## `/investments/holdings/get`

Returns current investment positions (holdings) for investment-type accounts.

**`holdings[]`**

| Field | Type | Notes |
|---|---|---|
| `account_id` | string | Account holding this position |
| `security_id` | string | Links to `securities[]` for instrument details |
| `quantity` | number | Number of shares/units held |
| `cost_basis` | number \| null | Average cost per share. Null if institution doesn't provide. |
| `institution_value` | number \| null | Current market value as reported by institution |
| `institution_price` | number \| null | Price per share as reported by institution |
| `institution_price_as_of` | string \| null | Date of the price (YYYY-MM-DD) |
| `iso_currency_code` | string \| null | Currency of the holding |
| `unofficial_currency_code` | string \| null | For non-ISO currencies |

**`securities[]`**

| Field | Type | Notes |
|---|---|---|
| `security_id` | string | Unique identifier — join to `holdings[].security_id` |
| `name` | string \| null | Full security name e.g. "Apple Inc." |
| `ticker_symbol` | string \| null | Exchange ticker e.g. "AAPL" |
| `type` | string | `equity`, `etf`, `mutual fund`, `fixed income`, `cash`, `derivative`, `other` |
| `isin` | string \| null | International Securities Identification Number |
| `close_price` | number \| null | Latest closing price |
| `close_price_as_of` | string \| null | Date of close price |
| `iso_currency_code` | string \| null | Currency |

**Notes:**
- Join `holdings` to `securities` on `security_id` to get the full picture of a position
- `cost_basis` is often null — don't rely on it for gain/loss calculations without a fallback
- `institution_value` is the most reliable current value field

---

## `/investments/transactions/get`

Returns historical investment transactions (buys, sells, dividends, fees, etc.).

**`investment_transactions[]`**

| Field | Type | Notes |
|---|---|---|
| `investment_transaction_id` | string | Unique transaction identifier |
| `account_id` | string | Account this transaction belongs to |
| `security_id` | string \| null | Links to `securities[]`. Null for cash transactions. |
| `date` | string | Transaction date (YYYY-MM-DD) |
| `name` | string | Description of the transaction |
| `quantity` | number | Shares/units transacted |
| `amount` | number | Total dollar value. Positive = cash out of account (buy). Negative = cash into account (sell/dividend). |
| `price` | number | Price per share at time of transaction |
| `fees` | number \| null | Fees/commissions charged |
| `type` | string | `buy`, `sell`, `cash`, `transfer`, `fee`, `dividend` (high-level) |
| `subtype` | string | More specific e.g. `dividend`, `capital gain long`, `qualified dividend`, `reinvested dividend` |
| `iso_currency_code` | string \| null | Currency |
| `unofficial_currency_code` | string \| null | For non-ISO currencies |

**Pagination fields**

| Field | Type | Notes |
|---|---|---|
| `total_investment_transactions` | integer | Total count across all pages |

**Notes:**
- Use `type` + `subtype` together to classify transactions (e.g. `cash` + `dividend` vs `buy` + `buy`)
- `amount` sign convention: positive = money left account (purchase), negative = money entered account (sale, dividend)
- Pair with `securities[]` (also returned in response) for instrument details

---

## `/liabilities/get`

Returns detailed liability data for credit and loan accounts. Supported types: credit cards, student loans, mortgages.

**`liabilities.credit[]`** (credit cards)

| Field | Type | Notes |
|---|---|---|
| `account_id` | string | Links to `accounts[]` |
| `aprs[]` | array | APR details — each has `apr_percentage`, `apr_type`, `balance_subject_to_apr`, `interest_charge_amount` |
| `is_overdue` | boolean \| null | Whether payment is overdue |
| `last_payment_amount` | number \| null | Amount of last payment |
| `last_payment_date` | string \| null | Date of last payment (YYYY-MM-DD) |
| `last_statement_issue_date` | string \| null | Date of last statement |
| `last_statement_balance` | number \| null | Balance on last statement |
| `minimum_payment_amount` | number \| null | Minimum payment due |
| `next_payment_due_date` | string \| null | Date next payment is due |

**`liabilities.student[]`** (student loans)

| Field | Type | Notes |
|---|---|---|
| `account_id` | string | Links to `accounts[]` |
| `disbursement_dates[]` | string[] \| null | Dates loan was disbursed |
| `expected_payoff_date` | string \| null | Projected payoff date |
| `guarantor` | string \| null | Loan guarantor name |
| `interest_rate_percentage` | number | Annual interest rate |
| `is_overdue` | boolean \| null | Whether payment is overdue |
| `last_payment_amount` | number \| null | Amount of last payment |
| `last_payment_date` | string \| null | Date of last payment |
| `last_statement_issue_date` | string \| null | Date of last statement |
| `loan_name` | string \| null | Name/type of loan |
| `loan_status.type` | string | e.g. `repayment`, `deferment`, `forbearance`, `in_school` |
| `minimum_payment_amount` | number \| null | Minimum monthly payment |
| `next_payment_due_date` | string \| null | Next due date |
| `origination_date` | string \| null | When loan originated |
| `origination_principal_amount` | number \| null | Original loan amount |
| `outstanding_interest_amount` | number \| null | Accrued interest not yet capitalized |
| `payment_reference_number` | string \| null | Payment reference |
| `pslf_status.estimated_eligibility_date` | string \| null | PSLF estimated eligibility |
| `repayment_plan.type` | string \| null | e.g. `income_driven`, `standard`, `graduated` |
| `sequence_number` | string \| null | Loan sequence number (for servicers with multiple loans) |

**`liabilities.mortgage[]`** (mortgages)

| Field | Type | Notes |
|---|---|---|
| `account_id` | string | Links to `accounts[]` |
| `account_number` | string \| null | Loan account number |
| `current_late_fee` | number \| null | Current late fee owed |
| `escrow_balance` | number \| null | Escrow account balance |
| `has_pmi` | boolean \| null | Whether PMI is present |
| `has_prepayment_penalty` | boolean \| null | Whether prepayment penalty applies |
| `interest_rate.percentage` | number \| null | Interest rate |
| `interest_rate.type` | string \| null | `fixed` or `variable` |
| `last_payment_amount` | number \| null | Last payment amount |
| `last_payment_date` | string \| null | Last payment date |
| `loan_term` | string \| null | Term e.g. "30 year" |
| `loan_type_description` | string \| null | e.g. "Conventional" |
| `maturity_date` | string \| null | Loan maturity date |
| `minimum_monthly_payment` | number \| null | Minimum monthly payment |
| `next_monthly_payment` | number \| null | Next scheduled payment amount |
| `next_payment_due_date` | string \| null | Next payment due date |
| `origination_date` | string \| null | Origination date |
| `origination_principal_amount` | number \| null | Original loan amount |
| `past_due_amount` | number \| null | Amount past due |
| `property_address` | object \| null | Street, city, region, postal code of property |
| `ytd_interest_paid` | number \| null | Interest paid year-to-date |
| `ytd_principal_paid` | number \| null | Principal paid year-to-date |

**Notes:**
- `accounts[]` is also returned — join on `account_id` for balance and account type info
- Credit liability data is refreshed daily; mortgage/student loan data may lag
- US-only coverage; limited Canadian support
