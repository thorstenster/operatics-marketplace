---
name: sap-workflow-investigation
description: Use this skill whenever the user asks about SAP Business Workflow activity on Thorsten's system — e.g. "sind Workflows gelaufen", "welche Workflows/Workitems", "wer hat einen Workflow gestartet", "welche Funktion/welcher Zweck steckt hinter Workitem X", "workflow log/history". Covers which tables to query (SWWWIHEAD, SWWLOGHIST, SWW_CONT/SWWVCONTWI, HRS1000) and in what order, plus a query_table parser quirk on multi-condition lookups against HRS1000. Prefer query_table for all of this — run_abap_code is not needed for any step here.
---

# SAP Workflow Investigation

Reusable recipe for answering "what workflows ran / who ran them / what does a workitem do"
without falling back to `run_abap_code` — everything below works with `query_table`.

## 1. Which workflow instances ran, and who started them

Table `SWWWIHEAD`, filter `WI_TYPE = 'F'` for top-level workflow instances (not sub-steps).

Useful fields: `WI_ID`, `WI_CD` (date), `WI_CT` (time), `WI_STAT` (`STARTED`/`COMPLETED`/...),
`WI_CREATOR`, `WI_TEXT` (a generic instance label — usually not the real workflow name, see
step 3), `WI_RH_TASK` (technical workflow template ID, e.g. `WS90100001` — this is the *ID*,
not the name; don't show it to the user directly unless asked, resolve it to a name instead).

## 2. What actually happened during a workitem (execution log)

Table `SWWLOGHIST`, filter `WI_ID = '<12-digit padded id>'` (e.g. `'000000000005'` for
workitem 5 — pad to 12 digits with leading zeros). Fields: `METHOD`, `MESSAGE`, `METH_USER`,
`METH_EDATE`, `METH_ETIME`. Shows workflow-runtime method calls (`SWW_WI_CREATE...`,
`SWW_WI_START`, ...) — useful for confirming who/when triggered it, but these are generic
runtime plumbing, not the business function.

## 3. What a workitem/workflow template is actually *for* (the human-readable name/purpose)

This is the useful one for "welche Funktion steckt dahinter" type questions. Workflow
templates and standard tasks have short texts in table `HRS1000`, keyed by:
- `OTYPE` = `'WS'` for a workflow template, `'TS'` for a standard task (a single step)
- `OBJID` = the numeric part of the technical ID (e.g. `90100001` for `WS90100001`)
- `LANGU` = language key (`'D'` for German)
- `SHORT` = short technical name, `STEXT` = descriptive text

Query with just `OBJID` as the condition and read both the `TS` and `WS` rows that come
back — a single OBJID often has entries for both the workflow template and its associated
standard task, which is exactly what you want (workflow name + step name).

**⚠ Parser quirk:** combining `OTYPE` and `OBJID` as two AND-conditions in the same
`query_table` call throws a parser error (`"ANDOBJID" is not valid here"` / `"ANDOTYPE" is
not valid here"` — looks like a missing space between `AND` and the field name in the
generated WHERE clause). Workaround: filter on `OBJID` alone (it's numeric and usually
unique enough) and eyeball/filter the `OTYPE`/`LANGU` you need from the small result set,
rather than trying to add the second condition.

## 4. Container data (business object / parameters bound to the workitem)

Tables `SWW_CONT` (raw) / `SWWVCONTWI` (typed view) can hold container element values
(e.g. which object was passed in). In practice these have been empty for simple/demo
workflow instances tested so far — don't be surprised by an empty result; it doesn't mean
the query is wrong, it can just mean the workflow didn't bind any container elements.

## Putting it together

For "welche Funktion liegt hinter Workitem X":
1. `SWWWIHEAD` → get `WI_RH_TASK` (e.g. `WS90100001`) for the workitem.
2. Extract the numeric OBJID from it (`90100001`).
3. `HRS1000` filtered on that `OBJID` → read `STEXT` for the `WS` row (workflow name) and
   the `TS` row (step name/purpose).
4. Answer with the human-readable name(s) — see the `abapgit-pull-history` /
   `run-abap-code-best-practices` skills for the general rule: present this in Demo Mode
   (no table/field names in the visible answer) unless asked how you found it.

No `run_abap_code` is required anywhere in this flow — it's `query_table` end to end,
consistent with Rule 0 in `run-abap-code-best-practices`.
