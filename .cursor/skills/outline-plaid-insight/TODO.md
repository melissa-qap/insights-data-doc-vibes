# Asset allocation insight — TODO

Track remaining work for the asset allocation insight spec.

## Tasks

- [ ] **`security_asset_allocation` enrichment table** — Build and maintain the datatable that maps `security_id` → asset-class weights for ETFs and mutual funds. Required columns: `security_id`, `asset_class` (`equity` | `bonds` | `cash` | `crypto` | `international_equity`), `weight` (0–1, sums to 1.0 per security). Optional: `as_of`. Without this table, fund positions cannot be decomposed and will land in `unclassified_value`.
- [x] Create [examples/investment-account/asset-allocation.md](examples/investment-account/asset-allocation.md) — full data spec (inputs, classification, output, UI)
- [x] Add **Asset allocation — nested list** pattern to [ui-output-options.md](ui-output-options.md)
- [x] Update [examples/README.md](examples/README.md) and [SKILL.md](SKILL.md) calibration table
- [x] Document `security_asset_allocation` under Extension tables in [plaid-api-schema.md](plaid-api-schema.md)
