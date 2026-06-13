# insights-data-doc-vibes

Cursor skill and documentation for outlining financial insights from Plaid data.

## Contents

- **`.cursor/skills/outline-plaid-insight/`** — Skill for mapping Plaid datatables to insight inputs, analysis, and UI outputs (see `SKILL.md`).
- **`plaid-api-schema.xlsx`** — Excel export of all Plaid datatables and enum lookup tables (one sheet per table). Regenerate after editing `plaid-api-schema.md`:

  ```bash
  pip install -r scripts/requirements.txt
  python scripts/export-plaid-schema-to-excel.py
  ```
