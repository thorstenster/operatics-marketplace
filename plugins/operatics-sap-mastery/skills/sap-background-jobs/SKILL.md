---
name: sap-background-jobs
description: >
  Use this skill whenever the user asks about SAP background jobs — welche Jobs sind heute
  gelaufen, gab es Fehler, welche Programme wurden ausgeführt, wann hat ein Job gestartet
  oder geendet, oder wer hat einen Job eingeplant. Deckt die wichtigsten Tabellen (TBTCO,
  TBTCP), Statuskennzeichen, funktionierende ABAP-Muster und bekannte Fallstricke ab.
---

# SAP Hintergrundjobs auslesen

## Tabellen

| Tabelle | Inhalt |
|---------|--------|
| `TBTCO` | Job-Kopfdaten (eine Zeile pro Job-Lauf): Name, Status, Start-/Endzeit, Benutzer |
| `TBTCP` | Job-Schritte (eine Zeile pro Schritt): Programm, Returncode, Schritt-Status |

Beide Tabellen über `JOBNAME` + `JOBCOUNT` verknüpft.

## Wichtige Felder — TBTCO

| Feld | Bedeutung |
|------|-----------|
| `JOBNAME` | Jobname |
| `JOBCOUNT` | Eindeutige Job-Instanz-Nummer |
| `SDLSTRTDT` | Geplantes Startdatum (Typ `d`) |
| `SDLSTRTTM` | Geplante Startzeit |
| `ENDDATE` | Tatsächliches Enddatum |
| `ENDTIME` | Tatsächliche Endzeit |
| `STATUS` | Job-Status (siehe unten) |
| `SDLUNAME` | Benutzer, der den Job zuletzt verändert hat |
| `AUTHCKNAM` | Benutzer, unter dem der Job läuft (Autorisierungsprüfung) |

## Statuswerte in TBTCO und TBTCP

| Code | Bedeutung |
|------|-----------|
| `S` | Scheduled — eingeplant, noch nicht freigegeben |
| `R` | Released — freigegeben, wartet auf Ausführung |
| `Y` | Ready — bereit zur Ausführung |
| `A` | Active — läuft gerade |
| `F` | Finished — erfolgreich abgeschlossen |
| `Z` | Cancelled — abgebrochen (Fehler oder manuell) |
| `P` | Printed — Spool-Liste gedruckt (nach F) |

→ Fehlerhafte Jobs erkennt man an Status `Z`.

## Wichtige Felder — TBTCP

| Feld | Bedeutung |
|------|-----------|
| `JOBNAME` | Jobname (Join-Schlüssel zu TBTCO) |
| `JOBCOUNT` | Job-Instanz (Join-Schlüssel zu TBTCO) |
| `STEPCOUNT` | Schrittnummer |
| `PROGNAME` | ABAP-Programm des Schritts |
| `VARIANT` | Selektionsvariante |
| `STATUS` | Schritt-Status (gleiche Codes wie TBTCO) |
| `EXITCODE` | Returncode des Schritts (leer = 0 = OK) |
| `SDLDATE` | Datum, zu dem der Schritt eingeplant wurde |
| `SDLUNAME` | Benutzer des Schritts |

## Vorgehen: Jobs eines Tages abfragen

### Schritt 1 — Job-Köpfe über TBTCO

Entweder per `query_table`:

```abap
" query_table: Bedingung SDLSTRTDT = '20260702'
```

Oder per `run_abap_code` (funktionierendes Muster — alte Syntax verwenden!):

```abap
REPORT ztemp.

DATA: lt_jobs  TYPE TABLE OF tbtco,
      ls_job   TYPE tbtco,
      lv_today TYPE d.

lv_today = sy-datum.   " oder gewünschtes Datum als Literal: lv_today = '20260702'.

SELECT jobname jobcount sdlstrtdt sdlstrttm enddate endtime status sdluname
  FROM tbtco
  INTO CORRESPONDING FIELDS OF TABLE lt_jobs
  WHERE sdlstrtdt = lv_today.

SORT lt_jobs BY sdlstrttm.

LOOP AT lt_jobs INTO ls_job.
  WRITE: / ls_job-sdlstrttm.
  WRITE: ls_job-jobname.
  WRITE: ls_job-sdluname.
  WRITE: ls_job-status.
ENDLOOP.
```

### Schritt 2 — Schritte zu einem Job über TBTCP

Am einfachsten per `query_table` (kein Nested-SELECT-Problem):

```
query_table TBTCP, Bedingung: SDLDATE = '20260702'
Felder: JOBNAME, STEPCOUNT, PROGNAME, STATUS, EXITCODE
```

Oder direkt im ABAP — aber **nicht als Nested-SELECT im LOOP** (→ Runtime-Fehler).
Stattdessen erst alle Schritte in einer eigenen SELECT-Abfrage holen,
dann im ABAP-Loop per READ TABLE verknüpfen:

```abap
REPORT ztemp.

DATA: lt_jobs  TYPE TABLE OF tbtco,
      ls_job   TYPE tbtco,
      lt_steps TYPE TABLE OF tbtcp,
      ls_step  TYPE tbtcp,
      lv_today TYPE d.

lv_today = sy-datum.

SELECT jobname jobcount sdlstrtdt sdlstrttm status sdluname
  FROM tbtco INTO CORRESPONDING FIELDS OF TABLE lt_jobs
  WHERE sdlstrtdt = lv_today.

SELECT jobname jobcount stepcount progname status exitcode
  FROM tbtcp INTO CORRESPONDING FIELDS OF TABLE lt_steps
  WHERE sdldate = lv_today.

SORT lt_jobs BY sdlstrttm.

LOOP AT lt_jobs INTO ls_job.
  WRITE: / ls_job-sdlstrttm.
  WRITE: ls_job-jobname.
  WRITE: ls_job-status.

  LOOP AT lt_steps INTO ls_step
    WHERE jobname = ls_job-jobname AND jobcount = ls_job-jobcount.
    WRITE: / '  ->'.
    WRITE: ls_step-progname.
    WRITE: ls_step-exitcode.
    WRITE: ls_step-status.
  ENDLOOP.
ENDLOOP.
```

## Bekannte Fallstricke

| Problem | Ursache | Lösung |
|---------|---------|--------|
| `"Unknown column name ABAPNAME"` | Das Feld heißt `PROGNAME`, nicht `ABAPNAME` | `PROGNAME` verwenden |
| Runtime-Fehler bei `WRITE: ls_job-jobname(30)` | Substring-Zugriff schlägt fehl wenn Feld kürzer als 30 | Längenangabe weglassen: `WRITE: ls_job-jobname` |
| Runtime-Fehler bei Nested-SELECT im LOOP | `run_abap_code`-Kontext unterstützt das nicht stabil | Alle SELECTs vor dem LOOP ausführen, dann per LOOP AT ... WHERE verknüpfen |
| `@`-Syntax / Komma-Feldlisten → "fixed point arithmetic" | Das `run_abap_code`-System läuft ohne Fixed-Point-Flag | **Alte Syntax** verwenden: kein `@`, kein `,` im Feldnamen-Teil, kein `INTO TABLE @DATA(...)` |
| `SDLDATE` in TBTCP ≠ tatsächliches Ausführungsdatum | `SDLDATE` ist das Einplan-Datum des Schritts, nicht zwingend das Lauf-Datum | Für den Lauf-Tag lieber über `TBTCO-SDLSTRTDT` filtern und dann per JOIN/READ TABLE auf TBTCP zugreifen |

## Beispielausgabe (2. Juli 2026)

```
00:10  EU_PUT    F    Programm: SAPRSEUT  RC: (leer=0)
01:40  EU_REORG  F    Programm: SAPRSLOG  RC: (leer=0)
```

Beide Jobs erfolgreich — keine Abbrüche (Status Z).

## Häufige Standard-SAP-Jobs

| Jobname | Programm | Zweck |
|---------|----------|-------|
| `EU_PUT` | SAPRSEUT | Update-Aufträge übertragen |
| `EU_REORG` | SAPRSLOG | Update-Protokoll reorganisieren |
| `RDDIMPDP` | RDDIMPDP | Transport-Dispatcher |
| `SAP_REORG_JOBS` | RSBTCDEL | Alte Job-Einträge löschen |
