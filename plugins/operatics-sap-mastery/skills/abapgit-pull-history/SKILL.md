---
name: abapgit-pull-history
description: Use this skill whenever the user asks who pulled, updated, or synced an abapGit repository on an SAP system, or wants to know the recent pull/change history of abapGit-managed objects (e.g. "wer hat abapGit Repositories gepullt", "wann wurde Repo X zuletzt aktualisiert", "abapGit pull history", "welche Objekte wurden zuletzt über abapGit geändert"). Also use it as background knowledge whenever inspecting the ZABAPGIT persistence table or abapGit repository configuration on an SAP system via ABAP code execution (e.g. run_abap_code / query_table tools). abapGit itself does NOT log pull events with user+timestamp anywhere — this skill documents the reliable proxy method (TADIR + REPOSRC last-change metadata) instead of re-deriving it from scratch each time.
---

# abapGit Pull History (via SAP System Introspection)

## Core fact to remember

**abapGit does not persist a "who pulled when" log.** The `ZABAPGIT` table (or the
system-specific persistence table found via `DD02L` search, see below) only stores:
- `TYPE = 'REPO'` — one row per repository, `DATA_STR` = serialized XML with URL, PACKAGE, BRANCH_NAME etc.
- `TYPE = 'REPO_CS'` — checksums of the last-known file state (no user/date)
- `TYPE = 'USER'` — per-user favorites/settings
- `TYPE = 'BACKGROUND'`, `'SETTINGS'`, `'PACKAGES'` — misc config

None of these rows carry a pull timestamp or the executing user. So "who pulled recently"
cannot be answered by querying `ZABAPGIT` directly — you have to use a proxy.

## The proxy method that works

When abapGit pulls a repo, it overwrites the ABAP objects (programs, classes, etc.) that
belong to the repo's package. Standard SAP source-metadata tables record who last saved
those objects and when — and for abapGit-managed packages, that "last save" **is** the last
pull. This gives an accurate, reconstructable pull history without any custom logging.

### Step 1 — Find the persistence table name (only if not already known)

Table is usually literally `ZABAPGIT`, but confirm via:

```abap
SELECT tabname FROM dd02l INTO TABLE @DATA(lt_tabs)
  WHERE tabname LIKE '%ABAPGIT%' OR tabname LIKE '%AGIT%'.
```

### Step 2 — List repositories and their packages

```abap
SELECT type, value, data_str FROM zabapgit
  INTO TABLE @DATA(lt_rows) WHERE type = 'REPO'.
```

`DATA_STR` is a UTF-16 XML blob. `WRITE` truncates it at ~255 chars, so don't dump it raw —
extract fields with regex instead (ABAP regex does **not** support lazy quantifiers `.*?`,
use a negated character class instead):

```abap
FIND REGEX '<URL>([^<]*)</URL>'         IN lv_str SUBMATCHES lv_url.
FIND REGEX '<NAME>([^<]*)</NAME>'       IN lv_str SUBMATCHES lv_name.
FIND REGEX '<PACKAGE>([^<]*)</PACKAGE>' IN lv_str SUBMATCHES lv_pack.
FIND REGEX '<BRANCH_NAME>([^<]*)</BRANCH_NAME>' IN lv_str SUBMATCHES lv_branch.
```

This yields the mapping: repo ID → URL → package (devclass) → branch.

### Step 3 — For each package, list its objects via TADIR

```abap
SELECT * FROM tadir INTO TABLE @DATA(lt_tadir)
  FOR ALL ENTRIES IN lt_pack
  WHERE devclass = lt_pack-table_line.
```

Group by `object` type (PROG, CLAS, DDLS, TABL, ...). PROG and CLAS are the most useful for
last-change lookup because their source lives in `REPOSRC`.

### Step 4 — Get last-change user/date from REPOSRC

`REPOSRC` fields are `PROGNAME`, `UNAM` (⚠ not `UNAME`), `UDAT` (type `d`/`sy-datum`, ⚠ not
string — declare target variables with the right type or the RFC call fails), `R3STATE`
(filter `= 'A'` for the active version).

**For PROG objects**, `progname = obj_name` directly:

```abap
SELECT SINGLE unam udat FROM reposrc
  INTO (lv_unam, lv_udat)
  WHERE progname = ls_tadir-obj_name AND r3state = 'A'.
```

**For CLAS objects**, the source is split across multiple generated "class pool" programs
per class (method includes, definitions, etc.), not a single `progname = classname` row.
Don't try to guess the exact padding of the class-pool program name (it varies and isn't
worth hand-computing) — instead search with `LIKE`:

```abap
SELECT progname, unam, udat FROM reposrc
  INTO TABLE @DATA(lt_rows)
  WHERE progname LIKE 'ZCL_MY_CLASSNAME%' AND r3state = 'A'.
```

Every row for the same class will normally share the same `UNAM`/`UDAT` if the whole class
was pulled together — take the max `UDAT` per class if you want one row per object.

### Step 5 — Present as a table: date, repo/package, objects touched, user

That's the whole answer: no repo-level pull log exists, but object-level last-change
metadata reconstructs it accurately, because SAP objects don't change on their own — only a
pull, an in-system edit, or a transport import touches `REPOSRC`/`UDAT`. If you want to be
extra careful to rule out "in-system edit" as an explanation, cross-check against `E070`
(transport requests) for the same date — but in practice for abapGit-managed packages this
almost never happens outside of pulls.

## Gotchas encountered in practice (avoid re-discovering these)

- `WRITE` of multiple long fields on one line can throw `"X" is not allowed here` syntax
  errors — split into separate `WRITE:` statements per field, or keep total line length short.
- ABAP `FIND REGEX` does not support `.*?` (lazy match) — use `[^<]*` instead when extracting
  XML tag contents.
- `REPOSRC-UNAM`, not `UNAME`. `REPOSRC-UDAT` is a date type, not a string — mismatched
  target variable types cause an RFC-level type error, not a normal ABAP syntax error.
- Class-pool program names in `REPOSRC` are the class name padded with `=` to a fixed
  width followed by a 2-4 char suffix (`CP`, `CO`, `CU`, `CI`, `CT`, `CM001`, ...) — don't
  hand-construct the padding, just use `LIKE 'CLASSNAME%'`.
- A repo with only a `DEVC` (package) TADIR entry and nothing else means it was registered
  but never actually pulled yet (empty/new repo).

## Answering the user

Always caveat the answer: this is derived from object last-change metadata as a reliable
proxy, not from a dedicated abapGit audit log (abapGit doesn't have one). Present results as
a table: **Date | Repo (URL/package) | Objects | User**.
