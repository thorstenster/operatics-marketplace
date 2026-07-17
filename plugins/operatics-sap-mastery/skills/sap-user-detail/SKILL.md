---
name: sap-user-detail
description: >
  Use this skill whenever you need to read SAP user data — address, name, email, phone,
  company, logon data — via ABAP code on Thorsten's system. Covers the correct BAPI
  (BAPI_USER_GET_DETAIL, function group SU_USER), the right ABAP syntax, the exact fields
  of structure BAPIADDR3, and known pitfalls to avoid. Trigger whenever the user asks
  "lies die Adresse von User X", "welche E-Mail hat Benutzer Y", "zeig mir die Stammdaten
  von USER Z", or similar.
---

# SAP User Detail — BAPI_USER_GET_DETAIL

## Vorgehen

Benutze immer `run_abap_code` mit dem nachfolgenden Muster.  
Kein `query_table` — Benutzerdaten sitzen in verteilten Tabellen, der BAPI liefert alles
sauber auf einmal.

## Funktionsbaustein

- **Name:** `BAPI_USER_GET_DETAIL`
- **Funktionsgruppe:** `SU_USER`
- **Suchergebnis im ADT:** `/sap/bc/adt/functions/groups/su_user/fmodules/bapi_user_get_detail`

## Funktionierender Quelltext (Vorlage)

```abap
REPORT ztemp.

DATA: ls_address  LIKE bapiaddr3,
      lt_return   TYPE TABLE OF bapiret2,
      ls_return   TYPE bapiret2.

CALL FUNCTION 'BAPI_USER_GET_DETAIL'
  EXPORTING
    username = 'FRANZT'          " <-- Benutzername anpassen
  IMPORTING
    address  = ls_address
  TABLES
    return   = lt_return.

WRITE: / 'Name:      ', ls_address-lastname, ls_address-firstname.
WRITE: / 'Anrede:    ', ls_address-title_p.
WRITE: / 'Funktion:  ', ls_address-function.
WRITE: / 'Abteilung: ', ls_address-department.
WRITE: / 'Firma:     ', ls_address-name.
WRITE: / 'Strasse:   ', ls_address-street, ls_address-house_no.
WRITE: / 'PLZ/Ort:   ', ls_address-postl_cod1, ls_address-city.
WRITE: / 'Land:      ', ls_address-country.
WRITE: / 'Telefon:   ', ls_address-tel1_numbr.
WRITE: / 'E-Mail:    ', ls_address-e_mail.

LOOP AT lt_return INTO ls_return.
  WRITE: / ls_return-type, ls_return-message.
ENDLOOP.
```

## Bekannte Fallstricke

| Problem | Ursache | Lösung |
|---------|---------|--------|
| `"REPORT/PROGRAM statement is missing"` | run_abap_code braucht zwingend einen Programm-Header | Immer `REPORT ztemp.` als erste Zeile |
| `"LS_ADDRESS does not have a component"` | Falscher Feldname oder `TYPE` statt `LIKE` | Deklaration als `LIKE bapiaddr3` (nicht `TYPE`), und nur Felder aus der Liste unten verwenden |
| `company1` existiert nicht | BAPIADDR3 hat kein Feld `company1` | Firmenname steht in `name` (Zeile 1), `name_2`, `name_3`, `name_4` |

## Felder von BAPIADDR3 (Auswahl)

| Feld | Bedeutung |
|------|-----------|
| `lastname` | Nachname |
| `firstname` | Vorname |
| `title_p` | Anrede / Titel (Person) |
| `function` | Funktion / Stellenbezeichnung |
| `department` | Abteilung |
| `name` | Firmenname Zeile 1 |
| `name_2` / `name_3` / `name_4` | Firmenname Zeilen 2–4 |
| `street` | Straße |
| `house_no` | Hausnummer |
| `postl_cod1` | Postleitzahl |
| `city` | Ort |
| `country` | Länderschlüssel (z. B. `DE`) |
| `tel1_numbr` | Telefon (Hauptnummer) |
| `e_mail` | E-Mail-Adresse |
| `building_p` | Gebäude (Person) |
| `floor_p` | Etage (Person) |
| `room_no_p` | Raumnummer (Person) |

## Weitere IMPORTING-Parameter des BAPI

Falls zusätzliche Daten benötigt werden:

```abap
DATA: ls_logondata TYPE bapilogond,   " Anmeldedaten (Typ, Gültig-bis, ...)
      ls_defaults  TYPE bapidefaul,   " Standardwerte (Drucker, Menü, ...)
      ls_islocked  TYPE bapislockd.   " Sperrstatus

CALL FUNCTION 'BAPI_USER_GET_DETAIL'
  EXPORTING
    username  = 'FRANZT'
  IMPORTING
    address   = ls_address
    logondata = ls_logondata
    defaults  = ls_defaults
    islocked  = ls_islocked
  TABLES
    return    = lt_return.
```

## Beispielausgabe (FRANZT)

```
Name:       Franz   Thorsten
Anrede:     Mr.
Firma:      operatics Thorsten Franz
Strasse:    Im Bachfeld    12
PLZ/Ort:    53179   Bonn
Land:       DE
E-Mail:     thorsten.franz@operatics.de
```
