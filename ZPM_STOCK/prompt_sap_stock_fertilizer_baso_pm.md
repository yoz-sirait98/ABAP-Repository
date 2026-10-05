# Prompt — SAP ECC R/3 ABAP Classic: Data Stock SAP (Fertilizer + BASO PM)

## Role

Act as a senior SAP ABAP Developer specializing in **SAP ECC 6.0 / R/3 with classic ABAP syntax**.

Build the requested SAP report using **classic ABAP syntax compatible with SAP ECC R/3**.

## Important Technical Constraints

- Target system: **SAP ECC 6.0 / R/3**
- Use **classic ABAP**, not modern ABAP syntax.
- Do NOT use:
  - Inline declarations such as `DATA(...)`, `FIELD-SYMBOL(...)`
  - Constructor expressions such as `VALUE #( )`, `CORRESPONDING #( )`, `NEW #( )`
  - String templates `|...|`
  - Table expressions such as `itab[ ... ]`
  - `LOOP AT ... INTO DATA(...)`
  - Modern Open SQL syntax that may not be supported by the target ECC release
- Prefer traditional:
  - `REPORT`
  - `TABLES`
  - `TYPES`
  - `DATA`
  - `SELECT-OPTIONS`
  - `PARAMETERS`
  - `SELECT ... INTO ...`
  - `LOOP AT`
  - `READ TABLE`
  - `APPEND`
  - `CLEAR`
  - `MOVE-CORRESPONDING` where appropriate
  - `FORM ... ENDFORM` for modularization
- Keep the implementation compatible with the actual DDIC structures of the target SAP system.
- Do not invent custom tables or fields.

---

# Project Objective

Create an SAP report to display **SAP Stock Data for Fertilizer + BASO PM**.

The report reads stock information from:

1. `MARD` — Storage Location Stock
2. `MARA` — General Material Data
3. `MAKT` — Material Description
4. `MBEW` — Material Valuation / Moving Price

The final result is displayed in an **ALV Grid**.

---

# Source Data

## 1. MARD — Storage Location Stock

Relevant fields:

| Field | Description |
|---|---|
| `MATNR` | Material |
| `WERKS` | Plant |
| `LGORT` | Storage Location |
| `LABST` | Unrestricted Stock |

### MARD filtering rule

Retrieve MARD data for all relevant storage locations **except**:

- `HO`
- `PWK`
- `CTRN`
- `WSHP`

In other words:

```text
LGORT NOT IN ('HO', 'PWK', 'CTRN', 'WSHP')
```

Do not hard-code a list of allowed storage locations. Exclude only the four specified storage locations.

---

## 2. MARA — General Material Data

Relevant fields:

| Field | Description |
|---|---|
| `MATNR` | Material |
| `MEINS` | Base Unit of Measure |
| `MTART` | Material Type |
| `MATKL` | Material Group |

### MARA filtering rule

Exclude materials where:

```text
MATNR LIKE '3%'
OR MATNR LIKE '9%'
```

Therefore, only materials whose material number does **not** start with `3` or `9` should be included.

Important:

- Be careful with SAP material number formatting and leading zeros.
- Use the actual DDIC type of `MARA-MATNR`.
- Do not convert MATNR unnecessarily.

---

## 3. MAKT — Material Description

Use `MAKT` to retrieve:

| Field | Description |
|---|---|
| `MATNR` | Material |
| `MAKTX` | Material Description |

Use the user's logon language where appropriate:

```text
MAKT-SPRAS = SY-LANGU
```

If the implementation chooses a different fallback strategy, explain it clearly.

---

## 4. MBEW — Material Valuation

Relevant fields:

| Field | Description |
|---|---|
| `MATNR` | Material |
| `BWKEY` | Valuation Area |
| `VERPR` | Moving Price |

The report must determine the appropriate MBEW record based on the material and plant/valuation area relationship.

For the normal ECC scenario, consider:

```text
MBEW-MATNR = MARD-MATNR
MBEW-BWKEY = MARD-WERKS
```

Do not assume a different valuation-area configuration without checking the actual SAP configuration/semantics.

---

# Selection Screen

Create a selection screen with the following selection options:

```text
MATNR
WERKS
LGORT
MTART
MATKL
```

Recommended classic ABAP definitions:

```abap
SELECT-OPTIONS:
  S_MATNR FOR MARA-MATNR,
  S_WERKS FOR MARD-WERKS,
  S_LGORT FOR MARD-LGORT,
  S_MTART FOR MARA-MTART,
  S_MATKL FOR MARA-MATKL.
```

The selection screen must allow users to filter by:

- Material
- Plant
- Storage Location
- Material Type
- Material Group

The fixed business rules must still apply regardless of selection:

1. Exclude LGORT `HO`, `PWK`, `CTRN`, `WSHP`
2. Exclude MATNR starting with `3`
3. Exclude MATNR starting with `9`

---

# Expected ALV Output

Display the following columns:

| Column | Source |
|---|---|
| Plant | `MARD-WERKS` |
| Stor. Location | `MARD-LGORT` |
| Material | `MARD-MATNR` |
| Description | `MAKT-MAKTX` |
| Material Type | `MARA-MTART` |
| Material Group | `MARA-MATKL` |
| Unrestricted Stock | `MARD-LABST` |
| UoM | `MARA-MEINS` |
| Moving Price | `MBEW-VERPR` |

Example output:

```text
Plant | Stor. Location | Material  | Description                 | Material Type | Material Group | Unrestricted Stock | UoM | Moving Price
DA20  | CENT           | 20017501  | PUPUK KIESERITE GRANULAR    | HIBE          | P2100          | 114.290            | KG  | 5.077
DA20  | DV01           | 20017501  | PUPUK KIESERITE GRANULAR    | HIBE          | P2100          | 8.088              | KG  | 5.077
DA20  | DV02           | 20017501  | PUPUK KIESERITE GRANULAR    | HIBE          | P2100          | 4.815              | KG  | 5.077
```

The exact decimal formatting should follow the SAP DDIC/domain definitions rather than being manually formatted as character strings.

---

# Functional Requirements

## 1. Data Relationship

The logical relationship is:

```text
MARD
  |
  | MATNR
  v
MARA
  |
  | MATNR
  v
MAKT

MARD-WERKS
  |
  | BWKEY
  v
MBEW
```

The result should contain one row per relevant MARD stock record.

For example, if one material exists in three storage locations:

```text
20017501 / DA20 / CENT
20017501 / DA20 / DV01
20017501 / DA20 / DV02
```

then the ALV should display three rows, each with the corresponding `LABST`.

---

# Important Data Selection Rules

The implementation must apply all filters correctly.

Conceptually:

```text
MARD
WHERE MATNR IN S_MATNR
  AND WERKS IN S_WERKS
  AND LGORT IN S_LGORT
  AND LGORT NOT IN:
      HO
      PWK
      CTRN
      WSHP

AND corresponding MARA record exists
AND MARA-MATNR does not start with 3
AND MARA-MATNR does not start with 9
AND MARA-MTART IN S_MTART
AND MARA-MATKL IN S_MATKL
```

Then retrieve:

```text
MAKT-MAKTX
MBEW-VERPR
```

---

# Performance Requirements

The report may potentially process a large amount of SAP stock data.

Therefore:

## Avoid

- `SELECT SINGLE` inside a large `LOOP` whenever avoidable.
- Nested database selects.
- `SELECT *`.
- Repeated database access for the same MATNR.
- Loading unnecessary records into internal tables.

## Prefer

A performant database access strategy appropriate for the ECC release, for example:

- A joined `SELECT` where safe and compatible with the target ECC database/release.
- Or staged selects using internal tables and `FOR ALL ENTRIES` where appropriate.
- Proper use of selection-screen filters at database level.

Important:

If using `FOR ALL ENTRIES`:

```abap
IF NOT gt_mard[] IS INITIAL.
  SELECT ...
    FOR ALL ENTRIES IN gt_mard
    WHERE ...
ENDIF.
```

Never execute a `FOR ALL ENTRIES` SELECT with an empty driver table.

Also consider duplicate MATNR/WERKS combinations before using `FOR ALL ENTRIES`.

---

# Stock Quantity

Use:

```text
MARD-LABST
```

as the unrestricted stock quantity.

Do not aggregate stock unless the functional requirement explicitly asks for aggregation.

If multiple MARD records exist, preserve the storage-location-level detail.

---

# Moving Price

Use:

```text
MBEW-VERPR
```

as Moving Price.

The implementation must correctly associate:

```text
MBEW-MATNR = Material
MBEW-BWKEY = Plant / Valuation Area
```

If there can be multiple MBEW records due to valuation type (`BWTAR`) in the actual system, investigate the business/data model before selecting a single value. Do not silently choose an arbitrary record.

---

# ALV Requirements

Use a classic ALV implementation compatible with ECC R/3.

Preferred options:

- `REUSE_ALV_GRID_DISPLAY`
- Or another classic ALV function module supported by the target ECC system.

Do not require SALV if it creates compatibility concerns.

Create a dedicated output structure, for example:

```abap
TYPES: BEGIN OF TY_OUTPUT,
         WERKS TYPE MARD-WERKS,
         LGORT TYPE MARD-LGORT,
         MATNR TYPE MARA-MATNR,
         MAKTX TYPE MAKT-MAKTX,
         MTART TYPE MARA-MTART,
         MATKL TYPE MARA-MATKL,
         LABST TYPE MARD-LABST,
         MEINS TYPE MARA-MEINS,
         VERPR TYPE MBEW-VERPR,
       END OF TY_OUTPUT.
```

Adjust field definitions if needed based on actual SAP DDIC semantics.

The ALV should:

- Have readable column headings.
- Use proper field lengths.
- Use proper quantity/unit relationship for `LABST` and `MEINS`.
- Use proper decimal formatting for `VERPR`.
- Allow normal ALV sorting/filtering/export functionality.

---

# Suggested Program Structure

Use classic modular ABAP.

Suggested flow:

```text
REPORT

TOP-OF-PAGE / TYPE DEFINITIONS
  |
  +-- Output structure
  +-- Internal tables
  +-- Work areas
  +-- ALV structures
  |
SELECTION-SCREEN
  |
START-OF-SELECTION
  |
  +-- PERFORM GET_DATA
  |
  +-- PERFORM DISPLAY_ALV
```

Suggested FORM routines:

```abap
FORM GET_DATA.
ENDFORM.

FORM DISPLAY_ALV.
ENDFORM.

FORM BUILD_FIELDCAT.
ENDFORM.
```

You may add additional `FORM`s if they make the implementation cleaner.

---

# Empty Result Handling

If no records are found:

Display a clear SAP message, for example:

```text
No stock data found for the selected criteria.
```

Do not display an empty ALV unnecessarily.

---

# Error / Data Quality Considerations

Handle these cases gracefully:

1. MARD record exists but MARA does not.
2. MARA exists but MAKT description is unavailable in `SY-LANGU`.
3. MBEW record is not found for a material/valuation area.
4. Stock is zero.
5. Multiple MBEW records exist due to valuation types.
6. Selection criteria return no data.

Do not terminate the program unexpectedly because a description or moving price is missing.

For missing description or moving price, preserve the stock row where appropriate and leave the missing field initial.

---

# Important: Do Not Make Unverified Assumptions

Before finalizing the code, inspect the actual SAP DDIC definitions if the development environment allows it.

Verify:

- `MARD-MATNR`
- `MARD-WERKS`
- `MARD-LGORT`
- `MARD-LABST`
- `MARA-MATNR`
- `MARA-MEINS`
- `MARA-MTART`
- `MARA-MATKL`
- `MAKT-MATNR`
- `MAKT-SPRAS`
- `MAKT-MAKTX`
- `MBEW-MATNR`
- `MBEW-BWKEY`
- `MBEW-VERPR`

Do not invent fields.

---

# Deliverables

Generate a complete ABAP report implementation that can be copied into an SAP ECC 6.0 / R/3 development environment.

The response/code should include:

1. Complete `REPORT` statement.
2. Type definitions.
3. Internal tables and work areas.
4. Selection screen.
5. Data retrieval logic.
6. Required business filters.
7. MAKT language handling.
8. MBEW moving price retrieval.
9. Empty-result handling.
10. Classic ALV Grid display.
11. ALV field catalog.
12. Clear comments explaining important logic.

---

# Coding Style

Follow clean classic ABAP conventions.

Example:

```abap
DATA:
  GT_OUTPUT TYPE STANDARD TABLE OF TY_OUTPUT,
  GS_OUTPUT TYPE TY_OUTPUT.
```

Use readable naming:

```text
GT_ = global internal table
GS_ = global structure/work area
GV_ = global variable
LT_ = local internal table
LS_ = local structure
LV_ = local variable
S_  = selection option
```

Keep database access centralized and easy to maintain.

Avoid unnecessarily complicated abstractions.

---

# Final Validation Checklist

Before presenting the solution, verify:

- [ ] Compatible with SAP ECC 6.0 / R/3.
- [ ] Uses classic ABAP syntax.
- [ ] No inline declarations.
- [ ] No constructor expressions.
- [ ] No table expressions.
- [ ] No string templates.
- [ ] Selection options exist for MATNR, WERKS, LGORT, MTART, MATKL.
- [ ] LGORT `HO` is excluded.
- [ ] LGORT `PWK` is excluded.
- [ ] LGORT `CTRN` is excluded.
- [ ] LGORT `WSHP` is excluded.
- [ ] MATNR beginning with `3` is excluded.
- [ ] MATNR beginning with `9` is excluded.
- [ ] MARD-LABST is used for unrestricted stock.
- [ ] MARA-MEINS is used for UoM.
- [ ] MAKT-MAKTX is used for description.
- [ ] MARA-MTART is displayed.
- [ ] MARA-MATKL is displayed.
- [ ] MBEW-VERPR is used for moving price.
- [ ] MBEW is correctly associated with the valuation area.
- [ ] No unnecessary `SELECT SINGLE` inside loops.
- [ ] `FOR ALL ENTRIES` is protected against empty driver tables if used.
- [ ] No `SELECT *`.
- [ ] Empty result is handled.
- [ ] ALV output columns match the requested layout.
- [ ] Quantity and price use SAP-native numeric formatting.
- [ ] Code is complete and directly usable.

## Expected Result

The final ALV should conceptually look like:

```text
Plant | Stor. Location | Material  | Description              | Material Type | Material Group | Unrestricted Stock | UoM | Moving Price
DA20  | CENT           | 20017501  | PUPUK KIESERITE GRANULAR | HIBE          | P2100          | 114.290            | KG  | 5.077
DA20  | DV01           | 20017501  | PUPUK KIESERITE GRANULAR | HIBE          | P2100          | 8.088              | KG  | 5.077
DA20  | DV02           | 20017501  | PUPUK KIESERITE GRANULAR | HIBE          | P2100          | 4.815              | KG  | 5.077
```

Produce the solution as **classic SAP ECC R/3 ABAP**, prioritizing correctness, database performance, maintainability, and compatibility over modern ABAP syntax.
