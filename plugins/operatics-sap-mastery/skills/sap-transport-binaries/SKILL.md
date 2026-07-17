---
name: sap-transport-binaries
description: This skill should be used when the user asks to fetch, download, or locate the binaries/data files/cofiles of an SAP transport request (Transportauftrag), asks about transport export status, or needs to SSH into a transport host (/usr/sap/trans) to pull the R and K files for a transport number and system ID. Also applies to diagnosing why a transport has no exported files ("Transportwesen einrichten").
version: 1.0.0
---

# SAP Transport Binaries via SSH

## Naming convention

For a transport request `<SID>K<number>` (e.g. `S4SK900009`), the exported files
on the transport host live under the shared transport directory:

- `/usr/sap/trans/data/R<number>.<SID>` — data file
- `/usr/sap/trans/cofiles/K<number>.<SID>` — cofile

Example: `S4SK900009` → `/usr/sap/trans/data/R900009.S4S` and
`/usr/sap/trans/cofiles/K900009.S4S`.

**Do not assume these files exist just because a request number exists.**
A request/task being *created* is not the same as being *released*, and being
*released* is not the same as being *exported*. Only a completed export
produces the data/cofile pair.

## Verifying status before assuming files exist

The action log at `/usr/sap/trans/actlog/<SID>Z<number>.<SID>` (note: `Z`, not
`K`, in the filename regardless of request vs. task) is the ground truth for
what actually happened to a request/task:

```
cat /usr/sap/trans/actlog/<SID>Z<number>.<SID>
```

Key markers to look for, in order:
- `ETK185` "has created the new request/task" — just created, nothing exported.
- `ETK193` — linked as a task to a parent request (tasks roll up into a parent
  request; the parent, not the task, is usually what gets released/exported).
- `ETK190` "started the release of the request/task" — release was *initiated*.
  This alone does **not** guarantee data/cofiles exist — the export can fail
  or the log can lack a completion entry.
- No release/export entry at all — nothing to fetch yet.

If a task is linked to a parent request (`ETK193 ... linked by ... to "<parent>"`),
check the **parent's** actlog too — that's usually the one that actually gets
released and exported, not the individual task.

## If data/cofiles are empty across the board

If `/usr/sap/trans/data`, `/usr/sap/trans/cofiles`, and `/usr/sap/trans/buffer`
are empty not just for one request but for *all* requests on the system, that's
a strong signal the transport management system (TMS) itself is newly set up
or not fully working yet, not that you're looking in the wrong place. Check:

```
ls -la /usr/sap/trans/bin/DOMAIN.CFG /usr/sap/trans/bin/TP_DOMAIN_<SID>.PFL
```

A recent mtime on these files means STMS configuration was just distributed
(Transaction STMS → System Overview → Extras → Distribute TMS Configuration).
If so, no export has ever succeeded yet — the fix is to finish setting up
Transportwesen (TMS domain, transport routes, RFC destinations) in STMS, then
re-release the request so an actual export runs, before there's anything to
fetch.

## Missing TPPARAM / TPSETTINGS after a fresh STMS setup

After STMS generates `/usr/sap/trans/bin/TP_DOMAIN_<SID>.PFL` and `DOMAIN.CFG`,
the classic `tp` tool (as `<sid>adm`) still needs two more files that STMS does
**not** create automatically in every setup:

1. `/usr/sap/trans/bin/TPPARAM` — the file `tp` reads by default when no `pf=`
   is given. Symptom: `tp connect <SID>` (or `pf=.../TPPARAM` explicitly)
   fails with `Error in .../TPPARAM: transdir not set.` (rc 208). Fix:
   ```
   ln -s /usr/sap/trans/bin/TP_DOMAIN_<SID>.PFL /usr/sap/trans/bin/TPPARAM
   chown -h <sid>adm:sapsys /usr/sap/trans/bin/TPPARAM
   ```

2. `TPSETTINGS` — on newer `tp` versions (seen: 381.580.22 / kernel 793) `tp`
   additionally opens a file literally named `TPSETTINGS` via a **relative**
   path (`openat(AT_FDCWD, "TPSETTINGS", ...)` — confirmed via `strace -f -e
   trace=openat,open`), i.e. relative to the **current working directory** of
   the shell that invokes `tp`, not `/usr/sap/trans/bin`. Symptom: `E-TPSETTINGS
   could not be opened` / `TPSETTINGS: transdir not set.` even after fixing
   TPPARAM, and even with a `TPSETTINGS` symlink correctly placed in
   `/usr/sap/trans/bin`. Since `su - <sid>adm` lands in the adm user's home
   directory (not `/usr/sap/trans/bin`), the relative lookup misses it there.
   Fix: create the symlink in `/usr/sap/trans/bin` (same pattern as TPPARAM)
   **and invoke `tp` from that directory**:
   ```
   ln -s /usr/sap/trans/bin/TP_DOMAIN_<SID>.PFL /usr/sap/trans/bin/TPSETTINGS
   chown -h <sid>adm:sapsys /usr/sap/trans/bin/TPSETTINGS
   su - <sid>adm -c 'cd /usr/sap/trans/bin && tp connect <SID>'
   ```
   A clean `tp connect <SID>` returns `tp finished with return code: 0` /
   `Connection to Database of <SID> was successful.`

Note: the `<sid>adm` user's login shell may be `csh`/`tcsh` (check with
`getent passwd <sid>adm`), where `2>&1` inside a `su - <sid>adm -c '...'`
string is invalid syntax ("Ambiguous output redirect") — don't rely on shell
redirects inside that string when diagnosing further; capture stderr from the
outer `ssh`/`su` invocation instead.

Once `tp connect` succeeds, the actual release/export must still be triggered
from the SAP GUI (SE01/SE09/SE10 → Release) — a working `tp` alone does not
retroactively export requests that were released while it was broken.

## Fetching once files exist

```bash
scp -i <key.pem> root@<host>:/usr/sap/trans/data/R<number>.<SID> ~/Downloads/
scp -i <key.pem> root@<host>:/usr/sap/trans/cofiles/K<number>.<SID> ~/Downloads/
```

## Handling the private key

Never paste a private key permanently into shared/long-lived files without
protecting it — write it to a restricted-permission file (`chmod 600`) in a
session-scoped scratch location, use it, and treat it as sensitive material
for the rest of the session.
