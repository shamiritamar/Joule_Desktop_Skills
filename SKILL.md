---
name: rfp-ibp-fitgap
description: >-
  Maps RFP demands to SAP IBP (Integrated Business Planning) standard capabilities, assigns each demand to the correct IBP license tier (Foundation, Advanced Demand, Advanced Supply, Advanced Inventory Optimization), and generates a formatted Excel fit-gap report with extension/integration assessment. Use this skill when the user uploads an RFP file and wants to check SAP IBP coverage, identify gaps, or prepare a fit-gap analysis with custom extension specifications. Trigger phrases include: "analyze this RFP for IBP", "check RFP against SAP IBP", "map requirements to IBP capabilities", "fit-gap analysis IBP", "RFP IBP coverage", "check if SAP IBP covers these demands", "IBP fit-gap", "rfp-ibp-fitgap".
allowed-tools: web_search read_file write_file execute
metadata:
  version: 1.1.0
  tags: rfp sap ibp integrated-business-planning fit-gap excel supply-chain demand-planning supply-planning
---

# RFP to SAP IBP Fit-Gap Mapper

Maps every demand in an uploaded RFP to the equivalent SAP IBP standard capability, assigns the correct IBP license tier, and produces a formatted Excel fit-gap report with extension/integration assessment.

---

## IBP License Model Reference

SAP IBP uses a 4-tier licensing model. Every demand must be mapped to the license tier that covers it.

| License Tier | Key | Scope |
|---|---|---|
| **IBP Foundation** | `Foundation` | Mandatory base. S&OP cycle (demand review, supply review, reconciliation, exec review), basic statistical forecasting (Copy Past Periods, Simple Moving Average, Single Exponential Smoothing), basic supply heuristic (TS-Based infinite), analytics/dashboards, Excel add-in, planning views, scenarios/versions, alerts, exception handling, approval workflows, platform administration, security, data integration framework. |
| **Advanced Demand** | `Adv Demand` | Full statistical forecasting library (~20 algorithms incl. ARIMA, gradient boosting, XGBoost), preprocessing (outlier detection/correction), consensus planning, demand sensing (ML short-term), lifecycle planning (phase-in/phase-out), best-fit model selection, AI-assisted forecast explanations. |
| **Advanced Supply** | `Adv Supply` | Constrained supply planning (finite heuristic), supply optimizer (MILP cost-optimized), allocations and deployment, order scheduling, response management. |
| **Advanced Inventory Optimization** | `Adv Inventory` | Multi-stage inventory optimization, safety stock optimization, service level optimization, working capital optimization, what-if inventory analysis. |

Demands requiring capabilities outside IBP (e.g., non-SAP ERP integration, external data lakes, MDM, MLOps) should be tagged `BTP / Datasphere` or `BTP + [tier]` as appropriate.

### License Assignment Rules

1. If the demand is covered by Foundation capabilities alone → `Foundation`
2. If the demand requires advanced forecasting, demand sensing, consensus planning, or AI forecast features → `Adv Demand`
3. If the demand requires constrained/optimized supply planning, allocations, deployment → `Adv Supply`
4. If the demand requires multi-stage inventory optimization or safety stock optimization → `Adv Inventory`
5. If the demand requires integration with non-SAP systems, external data ingestion, data governance, or MLOps → `BTP / Datasphere`
6. If the demand spans IBP platform + BTP (e.g., financial planning via SAC, adoption telemetry) → `Foundation + BTP`
7. When a demand touches multiple tiers, assign the **most specific advanced tier** (not Foundation)

---

## Step 0 — Verify Local Environment (SAFEGUARD)

Before doing anything else, confirm the local environment is usable.

**Check 1 — Working Directory:** List files using `ls`. If the tool returns a sandbox error, STOP and tell the user:

> "The working folder is not accessible. Move your RFP file to a local folder (~/Documents/ or ~/Desktop/), set it as the working directory in Joule Desktop settings, and re-upload."

**Check 2 — File Accessibility:** Attempt `read_file` on the uploaded RFP. Confirm it returns readable text.

**Check 3 — Write Access:** Run `echo "ok"` via `execute`. If it fails, Excel cannot be saved.

Only proceed once all three checks pass.

---

## Step 1 — Receive the RFP File

If the user has not yet uploaded a file:

> "Please upload your RFP file (PDF or DOCX format) to get started."

Read the file using `read_file`. If the text is empty or binary, ask the user to re-upload in DOCX format.

---

## Step 1B — Validate Extraction Completeness (SAFEGUARD)

After reading, check for completeness:

**Check 1 — Section Detection:** A typical IBP/planning RFP should contain most of these:

| Section | Hebrew Markers | English Markers |
|---|---|---|
| Demand | "ביקוש" OR "תחזית" | "Demand" OR "Forecast" |
| Supply | "היצע" OR "אספקה" | "Supply" OR "Replenishment" |
| S&OP | "תכנון עסקי" OR "S&OP" | "S&OP" OR "Sales & Operations" |
| Inventory | "מלאי" OR "אופטימיזציה" | "Inventory" OR "Stock Optimization" |
| Integration | "אינטגרציה" OR "ממשקים" | "Integration" OR "Interface" |

If 2+ major sections are missing, flag the issue.

**Check 2 — Repetition:** Compare first 50 lines against lines 300–350. If same block appears 3+ times, PDF is looping.

**Check 3 — Size ratio:** File > 500KB but < 300 unique extracted lines = content likely missing.

**On failure**, STOP. Tell the user what was found vs. missing. Offer: (1) Re-upload DOCX, (2) Upload missing sections separately, (3) Paste in chat, (4) Proceed partial. Wait for user decision.

---

## Step 1C — Language Detection and Translation

Detect the primary language of the RFP text:

- **English:** Set `source_lang = "en"`. Proceed to Step 2.
- **Hebrew (or other non-English):** Set `source_lang = "he"`. Tell the user:

> "I detected the RFP is in Hebrew. I'll translate all requirements to English for the analysis. The Excel output will include both the original Hebrew text and the English translation, side by side."

Translate the full RFP content to English in working memory. Use the English version for all subsequent steps. Preserve the original Hebrew in `demand_he` fields for the Excel output.

---

## Step 2 — Confirm Customer Name

Ask:

> "What is the customer or project name? (Used for the output filename, e.g. 'Strauss')"

Store as `CUSTOMER` (trim spaces, replace spaces with hyphens for filenames).

---

## Step 3 — Extract and Categorize Demands

Read the RFP (English version if translated). Extract every functional and technical demand. Organize by IBP planning area. See `references/ibp-taxonomy.md` for sub-areas, extension points, and integration types per planning area.

| Code | Module Name |
|------|-------------|
| SOP | Sales & Operations Planning |
| DP | Demand Planning |
| DS | Demand Sensing |
| SP | Supply Planning |
| IP | Inventory Optimization |
| RS | Response & Supply Planning |
| INT | Integration & Data Management |
| ANA | Analytics & Reporting |
| TECH | Technical / Administration |

Assign sequential IDs per planning area: `SOP-001`, `DP-001`, `SP-001`, etc.

For each demand populate:
- `id` — demand ID
- `module_code` — e.g. `DP`
- `module_name` — e.g. `Demand Planning`
- `sub_module` — e.g. `Statistical Forecasting`
- `demand_en` — concise English description, one sentence
- `demand_he` — original Hebrew text if `source_lang == "he"`; empty string otherwise
- `required_license` — IBP license tier per the License Assignment Rules above. One of: `Foundation`, `Adv Demand`, `Adv Supply`, `Adv Inventory`, `BTP / Datasphere`, `Foundation + BTP`
- `notes` — relevant RFP context; empty if none

### Post-Extraction Validation

1. Verify demands exist for at least 4 planning areas. Alert if fewer.
2. Alert if fewer than 40 demands from a file > 500KB.
3. Report planning-area-by-area count to the user. Wait for confirmation before Step 4.

Tell the user:

> "I've extracted **[N] demands** across [M] planning areas:
> [list each planning area with count]
>
> Does this look right? I'll now map each demand to SAP IBP capabilities — this may take a moment."

---

## Step 4 — Map Each Demand to SAP IBP Capabilities

For each demand, run a targeted `web_search`:

```
SAP IBP "[demand topic]" site:help.sap.com/docs/SAP_IBP
```

**Performance:** Fire up to 5 parallel `web_search` calls per message. Wait for results, then fire the next batch.

For each demand determine:

| Field | Rules |
|---|---|
| `ibp_capability` | IBP standard capability name (e.g. "Statistical Forecasting", "Supply Heuristic") |
| `ibp_app_function` | Planning app or function (e.g. "Demand Review", "Supply Heuristic Run"). Empty string if config-only. |
| `ibp_help_link` | URL from help.sap.com. Fallback: `https://help.sap.com/docs/SAP_IBP` |
| `required_license` | IBP license tier from Step 3. Validate and refine based on web search results. |
| `coverage` | One of: `Standard Config`, `Config + Effort`, `Extension`, `Custom Integration` |
| `explanation` | 3–5 sentences explaining how IBP covers or fails to cover this demand |
| `ext_summary` | **Extension/Custom Integration only:** `"[Type] ([Technical Object]): [one-liner]"`. Empty for Standard Config / Config + Effort. |
| `ext_spec` | **Extension/Custom Integration only:** `"Type: [type] | Object: [name] | [2–3 sentences: what to build, technical approach, IBP extension framework]"`. Empty for Standard Config / Config + Effort. |

### Coverage Model (4 Levels)

| Value | Meaning |
|---|---|
| `Standard Config` | Fully covered via IBP standard (planning areas, key figures, built-in heuristics, standard apps). Zero custom code. |
| `Config + Effort` | Standard IBP covers it but requires significant effort (complex planning book setup, multi-step heuristic chains, custom key figure formulas). No custom code. |
| `Extension` | Gap closed via IBP extension framework: custom planning operators (Groovy/Python), Excel Add-in macros, CIG/BTP integration operators. Targeted development within the IBP platform. |
| `Custom Integration` | No standard IBP coverage. Requires external custom integration: BTP Integration Suite iFlows, OData/REST custom services, or third-party tools. |

### Output JSON

Write to `mapping_data.json` in the scratch directory:

```json
{
  "customer": "CustomerName",
  "source_lang": "he",
  "demands": [
    {
      "id": "DP-001",
      "module_code": "DP",
      "module_name": "Demand Planning",
      "sub_module": "Statistical Forecasting",
      "demand_en": "Auto-select best forecast algorithm per product-location combination.",
      "demand_he": "בחירה אוטומטית של אלגוריתם תחזית מיטבי לכל שילוב מוצר-מיקום.",
      "required_license": "Adv Demand",
      "notes": "Requires weekly granularity."
    }
  ],
  "mappings": [
    {
      "id": "DP-001",
      "ibp_capability": "Statistical Forecasting – Auto Model Selection",
      "ibp_app_function": "Demand Planning App: Forecast Profile Configuration",
      "ibp_help_link": "https://help.sap.com/docs/SAP_IBP/...",
      "required_license": "Adv Demand",
      "coverage": "Standard Config",
      "explanation": "SAP IBP Demand Planning includes automatic best-fit model selection across multiple algorithms (Holt-Winters, Croston, MLR, etc.). The system evaluates historical data and selects the algorithm with the lowest forecast error per product-location. Weekly granularity is natively supported via time profile configuration. Requires Advanced Demand license for full algorithm library.",
      "ext_summary": "",
      "ext_spec": ""
    }
  ]
}
```

---

## Step 5 — Generate the Excel File

Install openpyxl first (separate terminal call):

```bash
pip3 install openpyxl
```

Then run the generation script. Replace `<skill-disk-path>` with this skill's directory path, `<scratch-dir>` with the scratch directory, `{CUSTOMER}` with the customer name:

```bash
python3 "<skill-disk-path>/scripts/generate_excel.py" \
  --data "<scratch-dir>/mapping_data.json" \
  --output "{CUSTOMER}-SAP-IBP-FitGap.xlsx"
```

Output file saved to the working directory:

**`{CUSTOMER}-SAP-IBP-FitGap.xlsx`**
- Sheet **"IBP Fit-Gap Analysis"**: title, planning area section banners, all columns per demand including Required License (adds Hebrew demand column if applicable)
- Sheet **"Summary by Planning Area"**: demand count per planning area with coverage breakdown
- Sheet **"License Requirements"**: demand distribution by IBP license tier with color coding

Coverage columns are color-coded: Green (Standard Config) · Light Green (Config+Effort) · Amber (Extension) · Red (Custom Integration).

License columns are color-coded: Blue (Foundation) · Green (Adv Demand) · Purple (Adv Supply) · Orange (Adv Inventory) · Teal (BTP / Datasphere) · Gray (Foundation + BTP).

---

## Step 6 — Present Results

Tell the user:

> "Done. File saved:
>
> **`{CUSTOMER}-SAP-IBP-FitGap.xlsx`** — SAP IBP fit-gap analysis ([N] demands, [M] planning areas)
>
> Coverage summary:
> - Standard Config: [N] demands
> - Config + Effort: [N] demands
> - Extension (Operator/Macro/CIG): [N] demands
> - Custom Integration (BTP/3rd party): [N] demands
> - **Weighted coverage**: [X]%
>
> License requirements:
> - Foundation (base): [N] demands
> - Advanced Demand: [N] demands
> - Advanced Supply: [N] demands
> - Advanced Inventory Optimization: [N] demands
> - BTP / Datasphere: [N] demands
>
> (Weighted: Std Config = 1.0, Config+Effort = 0.75, Extension = 0.5, Custom Integration = 0.0)
>
> Would you like me to highlight the critical integration gaps, drill into a specific planning area, or review the license recommendations?"