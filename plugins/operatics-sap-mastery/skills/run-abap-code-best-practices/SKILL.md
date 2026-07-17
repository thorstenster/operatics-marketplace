---
name: run-abap-code-best-practices
description: Use this skill whenever calling the RemoteMCP run_abap_code (or query_table / get_business_partner(s)) tools against Thorsten's SAP system, or whenever writing/debugging any ad-hoc ABAP snippet meant for one-off execution via that tool. Covers known syntax pitfalls, field/type gotchas, and workflow strategy that reduce failed attempts. This skill is a living document — Claude should add newly discovered gotchas to it whenever it solves a new multi-step or error-prone problem via run_abap_code, without being asked, so the list keeps growing across sessions.
---

# run_abap_code: Best Practices & Known Gotchas

Living document. Append new lessons under "Gotchas" whenever a new one is discovered —
don't wait to be asked. Keep entries terse (symptom → cause → fix), most-recently-added
learnings can go at the top of their section or at the end; ordering isn't critical, just
keep it appended and don't delete older entries unless they're proven wrong.

## Pflicht-Workflow vor jeder Ausführung

Diesen Ablauf **immer** einhalten — er vermeidet Kurzdumps, Safety-Rejections und
Whitelist-Überraschungen in einem Rutsch:

```
1. get_whitelists()                            ← erlaubte Tabellen/FMs/Klassen prüfen
2. check_abap_code(abap_code = <dein Code>)   ← Syntax & Safety prüfen
3. Nur wenn keine ERROR-Meldungen → run_abap_code(abap_code = <dein Code>)
```

**Schritt 1 — Whitelist abrufen (`get_whitelists`)**
Bevor Code geschrieben wird: prüfen, welche Tabellen, Funktionsbausteine, Klassen und
Transaktionen auf der Whitelist stehen. So werden Safety-Rejections durch nicht erlaubte
Objekte von vornherein vermieden, statt erst nach dem Check- oder Run-Aufruf zu scheitern.
`get_whitelists` liefert alle Listen in einem einzigen Aufruf.

**Schritt 2 — Code-Check (`check_abap_code`)**
Fängt Syntax- und Safety-Fehler ab, bevor sie zu einem Kurzdump führen. Spart mindestens
einen Roundtrip.

Bei WARNING-Meldungen im Check kann trotzdem ausgeführt werden — nur ERROR blockiert.

## REPORT-Statement — häufigster erster Fehler

**Jedes ABAP-Snippet für `run_abap_code` muss mit `REPORT ztemp.` beginnen.**

Fehlt diese Zeile, kommt sofort: `"The REPORT/PROGRAM statement is missing"`.
Diesen Fehler beim ersten Versuch vermeiden: `REPORT ztemp.` ist immer die allererste Zeile.

```abap
REPORT ztemp.   " <-- IMMER, ohne Ausnahme

DATA: lv_text TYPE string.
...
```

## Strategy (do this before writing complex code)

1. **Explore structure before querying.** Don't guess table/field names. Use `DD02L` to
   find tables (`WHERE tabname LIKE '%PATTERN%'`) and `DD03L` to list a table's fields
   (`WHERE tabname = 'X' ORDER BY position`) before writing SELECTs against it.
2. **Don't hand-construct generated/padded names.** E.g. ABAP class-pool program names in
   `REPOSRC` are the class name padded with `=` to a fixed width plus a suffix (`CP`, `CO`,
   `CU`, `CI`, `CT`, `CM001`, ...). Don't compute the padding — query with `LIKE 'NAME%'`
   instead.
2b. **Prefer set-based building blocks over hand-rolled REPORT logic when equivalent.**
   `query_table` / `get_business_partner(s)` already exist for their respective use cases —
   only fall back to `run_abap_code` when those don't cover the need.
3. **Extract, don't dump, large string fields.** Fields like `DATA_STR` (serialized
   XML/JSON blobs) get truncated by `WRITE` at ~255 chars. Use `FIND REGEX ... SUBMATCHES`
   to pull out just the piece you need.
4. **Iterate in small steps** when the shape of the data is unknown — one query to discover
   object types/fields present, then a second, narrower query using what you learned. This
   costs an extra round-trip but avoids long dead-end scripts.


## Open SQL — neue Syntax verwenden (ab ABAP 7.4)

**Immer die neue Open SQL Syntax mit `@` und `,` verwenden**, nicht die alte Syntax ohne diese Zeichen.
Die alte Syntax kann in bestimmten Kontexten (z. B. `run_abap_code`) Fehler wie
`"This ABAP SQL statement uses additions that can only be used when the fixed point
arithmetic flag is activated"` auslösen.

```abap
" FALSCH (alte Syntax) — kann Fehler verursachen:
SELECT unam udat FROM reposrc INTO CORRESPONDING FIELDS OF TABLE lt_rows
  WHERE udat >= lv_from AND r3state = 'A'.

" RICHTIG (neue Syntax) — immer so:
SELECT unam, udat FROM reposrc INTO CORRESPONDING FIELDS OF TABLE @lt_rows
  WHERE udat >= @lv_from AND r3state = 'A'.
```

Konkret:
- Host-Variablen in WHERE-Klauseln mit `@` prefixen: `WHERE datum >= @lv_datum`
- Feldlisten mit Komma trennen: `SELECT feld1, feld2, feld3`
- Inline-Deklaration bevorzugen: `INTO TABLE @DATA(lt_result)`

## Syntax pitfalls

- `WRITE` with several long/concatenated fields on one statement can throw cryptic errors
  like `"AS4TIME" is not allowed here. "." is expected.` — split into one `WRITE:` per
  field, or keep the line short.
- ABAP regex (`FIND REGEX`) does **not** support lazy quantifiers (`.*?`). Using one throws
  `Regular expression '...' is invalid`. Use a negated character class instead, e.g.
  `<TAG>([^<]*)</TAG>` instead of `<TAG>(.*?)</TAG>`.
- A genuine compile/syntax error doesn't come back as clean JSON — it comes back as a full
  SAP ICF 500-error HTML page mentioning a generated report name like `Z$$$XRFC`. This looks
  alarming but just means: simplify the code and retry, it's a normal compile error, not a
  system fault.

## Field / type pitfalls

- Verify exact field names via `DD03L` rather than guessing from convention — e.g. it's
  `REPOSRC-UNAM`, not `UNAME`.
- Match ABAP variable types to the DB field type exactly, especially dates: `REPOSRC-UDAT`
  is type `d` / `sy-datum`, not `string`. A mismatch causes an RFC-level type error
  ("The data type of LV_X is not compatible...") rather than a normal ABAP syntax error —
  confusing because it doesn't look like a type-mismatch message at first glance.
- Inline declarations (`DATA(...)`) inside `LOOP` bodies have occasionally coincided with
  compile failures in this dynamic-execution context. Not confirmed as the root cause every
  time, but when a script with inline-in-loop declarations fails mysteriously, try
  pre-declaring all variables with `DATA:` before the loop as a first debugging step.

## Domain knowledge picked up along the way

- **abapGit has no pull-history log.** See the dedicated `abapgit-pull-history` skill for
  the full method (TADIR + REPOSRC last-change metadata as a proxy). Cross-reference it
  rather than duplicating here.
- System has custom Z-packages including `$MCP` (this Claude/MCP integration itself —
  `ZCL_CLAUDE_API_CLIENT`, `ZCL_CLAUDE_MCP_HANDLER`, `ZCL_CLAUDE_TOOLS`, `Z_I_BUSINESSPARTNER`
  CDS view) and `ZCA_ROLE_SOD` (segregation-of-duties classes).

## Meta

This skill should be updated (not just consulted) whenever a new run_abap_code problem is
solved that involved more than one attempt or revealed a non-obvious fact about this SAP
system. Thorsten does not need to ask for the update each time.
