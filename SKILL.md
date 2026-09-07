# SKILL: Team Opportunity Filter — Current Period (CAL: Opportunity Deal Sheet)

## Description
Automates a three-part chained workflow for the Anaplan **Template Sales Forecasting** model:

1. **Part A — Get My Team**: Retrieves the members of the user's team, defined as the leaf employees in the list **"Employee to Sub Region T5"** (the dimension of the module **"INP: Input T5 Forecast"**).
2. **Part B — Get Current Period**: Retrieves the current planning period, defined as the value of the line item **"Current Quarter"** in the module **"SYS: Time Settings"** (e.g. `Q3 FY26`).
3. **Part C — Filter Opportunities (downstream)**: Filters the module **"CAL: Opportunity Deal Sheet"** to opportunities where:
   - the line item **"Opportunity Owner"** equals any member of the team from Part A, **AND**
   - the opportunity is **relevant to the current period** from Part B (i.e. **"Final Close Quarter"** = Current Quarter).

Use this skill whenever the user asks to "filter opportunities by my team", "show my team's deals", "opportunities for the current period/quarter", or any combination of team-ownership + current-period filtering on the opportunity deal sheet.

### Anaplan objects used
| Purpose | Object | Detail |
|---|---|---|
| Team roster | List: **Employee to Sub Region T5** | Leaf members = employees; parent = sub-region (e.g. "Sub Region 1 Northeast", code `S01-01`) |
| Team roster context | Module: **INP: Input T5 Forecast** | The module whose dimension is the team list (used only to confirm the list is the right one) |
| Current period | Module: **SYS: Time Settings**, line item **Current Quarter** | VARCHAR value, e.g. `Q3 FY26` |
| Opportunity data | Module: **CAL: Opportunity Deal Sheet** | Line items: `Opportunity Owner`, `Final Close Quarter`, `Opportunity Name`, `Stage`, `Forecast Category`, `NNACV`, `Close Date`, `In Forecast?` (verify names via `catalog_line_items` before first use — do NOT invent names) |

### "CAL: Opportunity Deal Sheet" dimensionality notes
- The module is dimensioned by **Opportunity** and a **sales hierarchy level dimension (T1–T5)**: each opportunity row repeats once per hierarchy level. **Always filter to the T5 (Rep) level** for the rep-level view, or deduplicate by opportunity in post-processing.
- No Versions dimension → never ask the user for a version on this module.
- Time relevance is carried by the **"Final Close Quarter"** line item (text), NOT by a native Time dimension slice.

## Prerequisites
- MCP session bound to the workspace/model above.
- Tools: `set_model_context`, `get_model_context`, `get_model_status`, `catalog_lists`, `catalog_line_items`, `sql_schema`, `sql_query`.

---

## Part A — Get My Team

### Step 1: Check and Set Model Context ⚠️ CRITICAL FIRST STEP

**ALWAYS start here before any other operations.**

Check get_model_context(). If not bound, follow the discovery protocol to connect to the Anaplan model.

```bash
get_model_context()
```

**If the response shows `bound: false` or context is not set:**

1. **Ask the user for Workspace ID and Model ID:**
   ```
   "To generate this pipeline report, I need to connect to your Anaplan model.
   
   Please provide:
   - Workspace ID
   - Model ID
   
   If you don't know these IDs, I can help you find them. Would you like me to:
   1. List your available workspaces
   2. Show models in a specific workspace
   ```

2. **If user doesn't know their IDs, help them discover:**
   
   **a) List available workspaces:**
   ```bash
   catalog_workspaces()
   ```
   Show the user their workspaces with IDs and ask which one to use.
   
   **b) Once they select a workspace, list its models:**
   ```bash
   catalog_workspaces_and_models(
     workspace_id: "user_selected_workspace_id"
   )
   ```
   Show the user the models and ask which one contains their pipeline data.

3. **Once you have both IDs, set the context:**
   ```bash
   set_model_context(
     workspace_id: "user_provided_workspace_id",
     model_id: "user_provided_model_id",
     workspace_name: "optional_workspace_name",
     model_name: "optional_model_name"
   )
   ```

**If context IS already set (`bound: true`):**

Verify it's the correct model by showing the user:
```
"I'm connected to [model_name] in [workspace_name]. 
Is this the correct model for the pipeline report?"
```

If the user confirms, proceed to Step 1.
If not, ask for the correct workspace and model IDs and set context accordingly.

---


### Step 2 — Verify model readiness (handle cold-start)
If any model-scoped call returns `CORE_TIMEOUT` or the model state is not `ready`:
1. Call `get_model_status`.
2. If not ready, wait per `retry.after_seconds` (typically 5–30 s) and poll again.
3. Proceed only when `state` = `ready`.

### Step 3 — Retrieve the team roster from the list
Query the list **Employee to Sub Region T5** (as a table) or use `catalog_lists` → list detail to enumerate leaf members:
```json
{
  "included_objects": { "Employee to Sub Region T5": [] },
  "query": "SELECT * FROM \"template sales forecasting.employee to sub region t5\" WHERE \"employee to sub region t5_is_leaf\" = TRUE"
}
```
(Adjust table/column names to what `sql_schema` returns for the list — run `sql_schema` with the same `included_objects` first.)

Capture:
- `team_members` = list of leaf employee display names (e.g. "Frazier, Tom", "Waterbury, Reggie", …).
- The parent node (sub-region) for context in the final report.

**Fallback**: if the list is not directly queryable via SQL, use `catalog_lists(name_contains: "Employee to Sub Region T5")` to get the list id, then read its members via list detail / `catalog_properties`.

## Part B — Get Current Period

### Step 4 — Schema + query for Current Quarter
```json
{ "included_objects": { "SYS: Time Settings": ["Current Quarter"] } }
```
Then:
```json
{
  "included_objects": { "SYS: Time Settings": ["Current Quarter"] },
  "query": "SELECT \"current quarter\" FROM \"template sales forecasting.sys: time settings\""
}
```
Capture the value as `:qtr` (e.g. `Q3 FY26`).

## Part C — Filter the Opportunity Deal Sheet

### Step 5 — Fetch the deal sheet schema
```json
{
  "included_objects": { "CAL: Opportunity Deal Sheet": ["Opportunity Owner", "Final Close Quarter", "Stage", "Forecast Category", "NNACV", "Close Date", "In Forecast?"] }
}
```
Note the exact dimension column names returned (e.g. the opportunity dimension and the hierarchy-level dimension) — **every dimension column must be sliced with `=` or constrained with `<dim>_is_leaf = TRUE`**.

### Step 6 — Query: team-owned opportunities in the current period
Run **one query per team member** (the SQL engine rejects `IN (...)` on some columns; if `owner IN (...)` fails, loop per member), or a single query with OR conditions on the measure column when supported:
```json
{
  "included_objects": { "CAL: Opportunity Deal Sheet": ["Opportunity Owner", "Final Close Quarter", "Stage", "Forecast Category", "NNACV", "Close Date", "In Forecast?"] },
  "query": "SELECT <opportunity_dim_col>, \"opportunity owner\", \"stage\", \"forecast category\", \"nnacv\", \"close date\", \"in forecast?\" FROM \"template sales forecasting.cal: opportunity deal sheet\" WHERE <opportunity_dim_col>_is_leaf = TRUE AND <hierarchy_dim_col> = :t5_member_or_leaf_constraint AND \"opportunity owner\" = :owner AND \"in forecast?\" = :in AND \"final close quarter\" = :qtr",
  "parameters": { "owner": "Frazier, Tom", "in": "✔️", "qtr": "Q3 FY26" }
}
```
Rules:
- `Opportunity Owner` and `Final Close Quarter` are **line items (measures)**, so they may be filtered directly by value in WHERE; if a bare comparison is rejected, wrap with `COALESCE(...)`.
- Prefer slicing the hierarchy dimension to the **T5 level** to avoid duplicate rows per opportunity; otherwise deduplicate by opportunity name in post-processing.
- Do NOT add a native `time`/`versions` filter unless `sql_schema` listed those columns.

### Step 7 — Aggregate and present
Produce:
1. **Team roster** (Part A) with count.
2. **Summary by owner**: total opps, Won (count + NNACV), Open = Commit/Upside/Pipeline (count + NNACV), Lost/Omitted.
3. **Open-deal detail table** (Commit first, then Upside/Pipeline by NNACV desc): Owner, Opportunity, Stage, Forecast Category, NNACV ($k), Close Date, In Forecast?.
4. **Key takeaways**: which deals could bridge the current-quarter gap to quota (chain with the "Get Current Period & Forecast-vs-Quota Gap Analysis" skill if the user also asks about the gap), data-hygiene flags (inconsistent category/close date across hierarchy levels), and team members owning zero opportunities.
5. Offer to export the filtered list (CSV/XLSX) via the sandbox tools.

---

## Notes & Gotchas
- **Never invent Anaplan object names** — if any name above fails, re-verify with `catalog_modules(name_contains: ...)` / `catalog_line_items(module_id: ...)` and use the exact returned names.
- `included_objects` is REQUIRED and must be identical on both `sql_schema` and `sql_query`.
- **SQL slicing rule**: every dimension column must be sliced (`=`) or leaf-constrained (`<dim>_is_leaf = TRUE`); `IN (...)` on dimension columns is rejected — loop one query per value instead.
- `Current Quarter` and `Final Close Quarter` are text values (e.g. `Q3 FY26`) — compare as strings, not as native Time periods.
- The deal sheet repeats opportunities across hierarchy levels T1–T5 — always control for this (slice T5 or deduplicate) or counts/sums will be inflated ~5×.
- Some opportunities show inconsistent Forecast Category / Close Date across hierarchy levels; when detected, flag them as data-hygiene issues rather than silently picking one value.
- Related chained skill: **RPM Sales Coach / Get Current Period & Forecast-vs-Quota Gap Analysis** — reuse its Part A/B outputs when the user asks both questions in one session.
