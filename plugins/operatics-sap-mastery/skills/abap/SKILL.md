---
name: abap
description: Working with ABAP code and SAP systems via the SAP MCP tools (sap_list_systems, query_table, check_abap_code, run_abap_code). Use whenever reviewing, writing, or fixing ABAP code that calls a function module/BAPI, or when a live SAP MCP connection is available and you need to confirm real table/structure fields or an interface before committing.
---

# ABAP / SAP-System Skill

## Never guess a BAPI/function module interface from memory — verify it live

SAP interface field names vary by release/support pack, and remembered
signatures are frequently wrong. Concrete case that bit us: assumed
`BAPI_MATERIAL_AVAILABILITY` took a `REQ_QTY` import parameter and returned
results via `AVAILABILITY`/`RETURN` **tables**. The real interface has
neither — `REQ_QTY` doesn't exist, and the actual parameters are
`PLANT`/`MATERIAL` (or `MATERIAL_LONG` for 40-char material numbers)/`UNIT`
on import, and a **scalar** `AV_QTY_PLT` + `RETURN` (structure `BAPIRETURN`,
not `BAPIRET2`) on export — no TABLES at all. The first version compiled
fine to the eye but would have dumped at runtime. When a live SAP MCP
connection is available, verify before committing:

1. `sap_list_systems` — get the short system name to target.
2. Introspect the *real* interface via `FUNCTION_IMPORT_INTERFACE`, executed
   with `run_abap_code` (this FM is on the sandbox's safety whitelist —
   confirmed via `check_abap_code` returning `safety_ok: true`):
   ```abap
   REPORT z_introspect.
   DATA: it_import TYPE STANDARD TABLE OF rsimp, ls_import TYPE rsimp,
         it_export TYPE STANDARD TABLE OF rsexp, ls_export TYPE rsexp,
         it_tables TYPE STANDARD TABLE OF rstbl, ls_tables TYPE rstbl,
         it_changing  TYPE STANDARD TABLE OF rscha,
         it_exception TYPE STANDARD TABLE OF rsexc.

   CALL FUNCTION 'FUNCTION_IMPORT_INTERFACE'
     EXPORTING
       funcname           = 'BAPI_XYZ'          " <- target FM
     TABLES
       import_parameter   = it_import
       export_parameter   = it_export
       tables_parameter   = it_tables
       changing_parameter = it_changing
       exception_list     = it_exception
     EXCEPTIONS
       error_message      = 1
       function_not_found = 2
       invalid_data       = 3
       OTHERS             = 4.

   LOOP AT it_import INTO ls_import.
     WRITE: / 'IMPORT', ls_import-parameter, ls_import-dbfield, ls_import-typ.
   ENDLOOP.
   LOOP AT it_export INTO ls_export.
     WRITE: / 'EXPORT', ls_export-parameter, ls_export-dbfield, ls_export-typ.
   ENDLOOP.
   LOOP AT it_tables INTO ls_tables.
     WRITE: / 'TABLES', ls_tables-parameter, ls_tables-dbstruct, ls_tables-typ.
   ENDLOOP.
   ```
   - `PARAMETER` = parameter name; `DBFIELD`/`DBSTRUCT` = the structure-field
     it's typed `LIKE`, when applicable; `TYP` = elementary/data-element type
     name (populated instead of `DBFIELD`/`DBSTRUCT` when the parameter is
     typed directly to a data element rather than `LIKE structure-field`).
   - Don't also select `-OPTIONAL` in the same loop/WRITE — on this system it
     reproducibly caused a 500/dump. If you need optionality, query it
     separately (see step 3) instead of adding it to the introspection report.
3. Cross-check unfamiliar structure/table names with `query_table`:
   - `DD02L`, `TABNAME LIKE '%FRAGMENT%'` to find the real table name when a
     remembered one (e.g. a hallucinated `BAPICAVAIL`) doesn't exist.
   - `DD03L`, `TABNAME = 'X'` to list a structure's fields with
     `DATATYPE`/`LENG`/`DECIMALS` — use this to confirm a target field is
     move-compatible with the quantity/char field you're assigning it to.
   - Gotcha: `query_table`'s condition parser breaks when two conditions key
     off the same field name (e.g. two `FIELDNAME` conditions) — it errors
     with `"ANDFIELDNAME" ist hier nicht erlaubt`. Use one condition per call
     (usually just `TABNAME =`) and filter further criteria client-side from
     the returned rows.
4. Before committing, run the *exact* snippet you intend to ship through
   `check_abap_code` (want `syntax_ok: true` / `safety_ok: true`) — it
   compiles against the real system's DDIC/interfaces and catches wrong
   field names before they reach the repo. Do this even for code you're
   fairly confident about; it's cheap and it's what caught the corrected
   `BAPI_MATERIAL_AVAILABILITY` call. **But this is not sufficient on its
   own** — see the next section: `check_abap_code` passing does not mean
   the call will run without dumping.
5. If SAP system access is denied or unavailable for a given step, don't
   fall back to guessing and presenting it as verified — say plainly that
   the interface/field names are unconfirmed and flag it to the user instead
   of shipping an unverifiable assumption as fact.

## Mandatory TABLES parameters — a syntax-clean call can still dump

Classic `TABLES` parameters (and `IMPORTING`/`EXPORTING` ones) can be marked
**mandatory** in the function module's interface. Omitting a mandatory
`TABLES` parameter compiles and passes `check_abap_code` fine (it's a
syntax/safety checker, not a runtime-obligation checker) but **dumps at
runtime** (`CALL_FUNCTION_PARM_MISSING`-style error) the moment the code
actually executes. This bit us for real: `BAPI_MATERIAL_AVAILABILITY` also
has `TABLES` parameters `WMDVSX` (`BAPIWMDVS`) and `WMDVEX` (`BAPIWMDVE`),
which looked like optional detail tables — but a call built only from
`PLANT`/`MATERIAL_LONG`/`UNIT`/`AV_QTY_PLT`/`RETURN` compiled clean and then
produced a genuine 500/dump via `run_abap_code`. Only after supplying empty
internal tables for `WMDVSX`/`WMDVEX` did the exact same call execute and
return a normal business message. **So: always check whether TABLES (and
other) parameters are mandatory, and don't rely on `check_abap_code` alone
to tell you.**

- The metadata *does* carry this (`RSIMP-OPTIONAL`, `RSTBL-OPTIONAL` from the
  `FUNCTION_IMPORT_INTERFACE` introspection above), but reading `-OPTIONAL`
  reproducibly dumped (500) on this system, in isolation and inside `IF`
  comparisons alike — it wasn't usable here.
- The reliable, ground-truth test instead: actually **execute** a minimal
  call via `run_abap_code` with placeholder values (e.g. `plant = 'TEST'`,
  a bogus material) — once with a parameter omitted, once with it supplied.
  A dump on omission = mandatory; a clean run (even with a business
  warning/error in `RETURN`, e.g. "material not maintained") = it was fine
  to omit, or you've now supplied the right value either way.
- Do this test for every TABLES parameter you're tempted to leave out, not
  just the ones that look load-bearing — "looks like an optional detail
  table" is exactly the assumption that broke here.

## Reading ABAP source correctly — comment column rule

A `*` only starts a full-line comment when it is the very first character of
the line (column 1, no leading whitespace). An **indented** line beginning
with `*name` is regular executable ABAP — most commonly a reference to the
SAP-generated shadow/"before image" structure for a classic `TABLES` work
area (`TABLES: vbap.` implicitly also gives you `*vbap`), not a comment.
Before concluding a line is dead/commented-out code, check its exact
indentation with `cat -A <file>` — don't rely on how the Read tool renders
whitespace.
