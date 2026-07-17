---
name: sap-application-log
description: Use this skill whenever the user asks about SAP Application Log (BAL / transaction SLG1) entries — reading them ("Einträge aus dem BAL", "Application Log von heute", "welche Fehler/Warnungen gab es", "Log-Nachrichten zu Objekt/Unterobjekt X") or writing/creating a new one ("leg ein Demoprotokoll an", "schreibe eine Log-Nachricht", "BAL_LOG_CREATE"). Covers reading log headers via query_table (BALHDR) and the correct, working sequence to load and read the actual message texts (BAL_DB_LOAD with I_T_LOG_HANDLE → BAL_LOG_MSG_READ) — several other plausible-looking function modules for this (BAL_LOG_MSG_TREE_GET, BAL_LOG_MSG_LIST_GET, BAL_DB_READ_MESSAGES) do NOT exist / don't work cleanly on this system. Also covers the write path (BAL_LOG_CREATE, then BAL_LOG_MSG_ADD, then BAL_DB_SAVE).
---

# SAP Application Log (BAL / SLG1) — Reading Entries

## 0. Proactive logging (standing instruction)

Whenever Claude does something interesting or noteworthy on this SAP system — solves a
multi-step problem, uncovers a notable finding, completes a meaningful analysis — Claude
should proactively write a short entry to the Application Log using the write path below
(object `ZCLAUDE`, subobject `LOG`), without being asked each time. Keep the message brief
and factual (what was done/found), using the `ZWORKFLOW`/`001` free-text pattern (`& & & &`)
unless a more specific message class/number is warranted. This is a standing background
habit, not something to narrate to the user unless they ask — just do it and move on.

## 1. List log headers (which logs exist, when, by whom, how many errors)

Table `BALHDR` via `query_table`. Useful fields: `LOGNUMBER`, `LOG_HANDLE` (read this
directly from BALHDR — it saves a lookup step later), `OBJECT`, `SUBOBJECT`, `EXTNUMBER`,
`ALDATE`, `ALTIME`, `ALUSER`, `PROBCLASS` (1=very important/red, 2=important, 3=medium,
4=additional info), `MSG_CNT_E`/`MSG_CNT_W`/`MSG_CNT_A`/`MSG_CNT_I`/`MSG_CNT_S` (message
counts by type: Error/Warning/Abort/Info/Success).

query_table parser quirk (same one seen with HRS1000 in the workflow-investigation
skill): combining two AND-conditions in one call can throw a parser error like
"ANDSUBOBJECT" is not valid here. Workaround: filter on the single most selective
condition (e.g. just SUBOBJECT or just ALDATE) and narrow down the result set by eye,
rather than stacking multiple conditions.

## 2. Get the actual message texts — the working sequence

This is the part that's easy to get wrong; several standard-sounding function modules for
this do NOT exist / don't behave as expected on this system:
- BAL_LOG_MSG_TREE_GET — function not found
- BAL_LOG_MSG_LIST_GET — function not found
- BAL_DB_READ_MESSAGES — exists but has confusing mandatory parameters
  (IT_R_MSGNUM, then I_LOG_HANDLE) that don't fit a simple "give me messages for these
  lognumbers" use case
- Constructing BAL_S_MSGH manually — this type doesn't exist under that name in this
  system's DDIC

What actually works: call BAL_DB_LOAD with I_T_LOG_HANDLE (not lognumbers!) — it
returns E_T_MSG_HANDLE directly as part of the same call, no separate message-list FM
needed. Then loop over that table calling BAL_LOG_MSG_READ per handle.

Example ABAP:

DATA: lt_log_handle TYPE bal_t_logh,
      lv_handle     TYPE balloghndl,
      lt_msg_handle TYPE bal_t_msgh.

lv_handle = '<log_handle from BALHDR-LOG_HANDLE>'.
APPEND lv_handle TO lt_log_handle.
" ... append more handles as needed ...

CALL FUNCTION 'BAL_DB_LOAD'
  EXPORTING
    i_t_log_handle = lt_log_handle
  IMPORTING
    e_t_msg_handle = lt_msg_handle
  EXCEPTIONS
    OTHERS = 1.

LOOP AT lt_msg_handle INTO DATA(ls_msgh).
  DATA: ls_msg TYPE bal_s_msg.
  CALL FUNCTION 'BAL_LOG_MSG_READ'
    EXPORTING
      i_s_msg_handle = ls_msgh
    IMPORTING
      e_s_msg        = ls_msg
    EXCEPTIONS
      OTHERS = 1.
  WRITE: / ls_msg-msgty.
  WRITE: ls_msg-msgid.
  WRITE: ls_msg-msgno.
  WRITE: ls_msg-msgv1(30).
  WRITE: ls_msg-msgv2(20).
  WRITE: ls_msg-msgv3(20).
ENDLOOP.

Note: I_T_LOG_HANDLE is the key that makes this work in one shot — passing lognumbers
into BAL_DB_LOAD (I_T_LOGNUMBER) also succeeds and returns E_T_LOG_HANDLE, but does
NOT populate E_T_MSG_HANDLE directly; you'd then need a second FM to get from log
handle to message handles, and that's exactly where the nonexistent-FM dead ends above come
from. Going straight from BALHDR-LOG_HANDLE to BAL_DB_LOAD(I_T_LOG_HANDLE=...) skips that
whole problem.

ls_msg-msgty/msgid/msgno/msgvN gives you the raw message class/number/parameters — to get
the actual rendered German/English text (as it would appear in SLG1), you'd additionally
need to read the message text via T100 with msgid+msgno and substitute &1/&2/...
placeholders with msgv1/msgv2/.... This skill doesn't yet document a clean way to do that
last text-rendering step — add it here once solved (e.g. MESSAGE ID ls_msg-msgid TYPE
ls_msg-msgty NUMBER ls_msg-msgno WITH ls_msg-msgv1 ls_msg-msgv2 ls_msg-msgv3 ls_msg-msgv4
INTO lv_text. is the standard ABAP idiom for this and is worth trying first).

## 3. Writing a new log (create + add message + save)

Three-FM sequence, all straightforward — no gotchas hit here (unlike reading):

```abap
DATA: ls_log_handle TYPE balloghndl,
      ls_log        TYPE bal_s_log,
      ls_msg        TYPE bal_s_msg,
      lt_lognumbers TYPE bal_t_lgnm.

ls_log-object    = 'ZCLAUDE'.      " log object, e.g. a custom Z object
ls_log-subobject = 'LOG'.          " subobject
ls_log-extnumber = 'Some label'.   " free-text external number/label
ls_log-aluser    = sy-uname.
ls_log-alprog    = sy-repid.

CALL FUNCTION 'BAL_LOG_CREATE'
  EXPORTING
    i_s_log      = ls_log
  IMPORTING
    e_log_handle = ls_log_handle
  EXCEPTIONS
    OTHERS       = 1.

ls_msg-msgty = 'I'.                " I/W/E/S/A
ls_msg-msgid = 'ZWORKFLOW'.        " message class (T100)
ls_msg-msgno = '001'.              " message number, e.g. free-text "& & & &"
ls_msg-msgv1 = 'first param'.
ls_msg-msgv2 = 'second param'.
ls_msg-msgv3 = 'third param'.
ls_msg-msgv4 = 'fourth param'.

CALL FUNCTION 'BAL_LOG_MSG_ADD'
  EXPORTING
    i_log_handle = ls_log_handle
    i_s_msg      = ls_msg
  EXCEPTIONS
    OTHERS       = 1.

CALL FUNCTION 'BAL_DB_SAVE'
  EXPORTING
    i_t_log_handle    = VALUE bal_t_logh( ( ls_log_handle ) )
  IMPORTING
    e_new_lognumbers  = lt_lognumbers
  EXCEPTIONS
    OTHERS = 1.

COMMIT WORK.
```

`BAL_DB_SAVE` returns `E_NEW_LOGNUMBERS` (a table, not a simple field — don't `WRITE` it
directly expecting a scalar lognumber, its structure has multiple fields including
`EXTNUMBER`, which is what shows up first if you dump it naively). To confirm what was
actually saved, just re-query `BALHDR` filtered on the `OBJECT` you used — the new row's
`LOGNUMBER`/`LOG_HANDLE` are right there. Message class `ZWORKFLOW` / number `001` with a
generic `& & & &` text is a handy free-text placeholder pattern when you don't want to
register a dedicated message number for a one-off log entry.

## 4. Presentation

Per Demo Mode (see run-abap-code-best-practices): summarize findings in plain language
(what failed, when, roughly why) without naming tables/FMs/fields in the visible answer,
unless asked how it was determined.
