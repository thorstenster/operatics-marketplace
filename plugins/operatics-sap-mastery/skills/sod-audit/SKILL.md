---
name: sod-audit
description: "Dieser Skill unterstützt bei der Durchführung von Segregation-of-Duties-Prüfungen (SoD) in SAP S/4HANA. Ziele: - Identifikation von Personen mit kritischen Berechtigungskombinationen - Bewertung von SoD-Risiken auf Ebene von Berechtigungsobjekten - Nachvollziehbarkeit, wodurch ein Konflikt entstanden ist - Ermittlung der Person, die eine konfliktverursachende Berechtigung vergeben hat - Managementgerechte Kommunikation der Ergebnisse"
---

# SAP S/4HANA SoD Auditor Skill

## Purpose

Dieser Skill unterstützt bei der Durchführung von Segregation-of-Duties-Prüfungen (SoD) in SAP S/4HANA.

Ziele:

- Identifikation von Personen mit kritischen Berechtigungskombinationen
- Bewertung von SoD-Risiken auf Ebene von Berechtigungsobjekten
- Nachvollziehbarkeit, wodurch ein Konflikt entstanden ist
- Ermittlung der Person, die eine konfliktverursachende Berechtigung vergeben hat
- Managementgerechte Kommunikation der Ergebnisse

Der Skill trennt strikt zwischen:
- technischer Analyseebene
- fachlicher Management-Kommunikation


# SoD Analysis Method

Eine SoD-Regel besteht aus mindestens zwei Berechtigungen, die nicht gemeinsam vergeben sein sollten.

Beispiel:

Konflikt:
Kreditoren anlegen
+
Zahlungen durchführen

Berechtigungsobjekte:
- F_LFA1_APP
- F_REGU_BUK

Ein Konflikt gilt erst als bestätigt, wenn dieselbe Person beide Berechtigungsbereiche besitzt.


# Technical Implementation

## Data Model

## Benutzer → Rollen

Tabelle:
AGR_USERS

Zweck:

Benutzer
   ↓
Rollen

Relevante Felder:

- UNAME
- AGR_NAME


## Rolle → Berechtigungsobjekte

Tabelle:
AGR_1251

Zweck:

Rolle
   ↓
Berechtigungsobjekte

Relevante Felder:

- AGR_NAME
- OBJECT
- FIELD
- LOW
- HIGH


## Benutzer → Person

Technische Benutzer sollen nicht als Management-Ergebnis ausgegeben werden.

Auflösung:

USR21

Felder:

- BNAME
- PERSNUMBER

Danach:

ADRP

Felder:

- PERSNUMBER
- NAME_FIRST
- NAME_LAST
- NAME_TEXT

Ablauf:

SAP Benutzer
      ↓
Personennummer
      ↓
Personenname


## Änderungsverfolgung

Tabellen:

CDHDR
CDPOS

Relevante Objektklassen:

- IDENTITY
- PFCG
- USER

CDHDR relevante Felder:

- OBJECTCLAS
- OBJECTID
- USERNAME
- UDATE
- UTIME
- TCODE

Ziel:

Wer hat wann eine Änderung durchgeführt, durch die der SoD-Konflikt entstanden ist?


# Technical Analysis Process

## Schritt 1: Rollen mit kritischen Berechtigungen finden

Beispiel ABAP:

SELECT agr_name,
       object
  FROM agr_1251
  INTO TABLE @DATA(lt_roles)
 WHERE object = 'F_LFA1_APP'.


Zweite Konfliktberechtigung:

SELECT agr_name,
       object
  FROM agr_1251
  INTO TABLE @DATA(lt_roles_payment)
 WHERE object = 'F_REGU_BUK'.


## Schritt 2: Benutzer der Rollen bestimmen

Beispiel ABAP:

SELECT uname,
       agr_name
  FROM agr_users
  INTO TABLE @DATA(lt_user_roles).


Logik:

Benutzer besitzt Rolle mit Berechtigung A
UND
Benutzer besitzt Rolle mit Berechtigung B

=> SoD Konflikt


## Schritt 3: Benutzer in Personen auflösen

Beispiel ABAP:

SELECT bname,
       persnumber
  FROM usr21
  INTO TABLE @DATA(lt_person).


Danach:

SELECT persnumber,
       name_text
  FROM adrp
  INTO TABLE @DATA(lt_names).


## Schritt 4: Ursache des Konflikts ermitteln

Änderungsbelege prüfen:

SELECT objectclas,
       objectid,
       username,
       udate,
       utime,
       tcode
  FROM cdhdr
  INTO TABLE @DATA(lt_changes)
 WHERE objectclas IN ('IDENTITY','PFCG').


Änderer ebenfalls auflösen:

Änderer Benutzer
        ↓
USR21
        ↓
ADRP
        ↓
Personenname


# Communication Rules

## Standardzielgruppe: Management / CEO

Die erste Ausgabe richtet sich an eine Führungskraft.

Der Empfänger möchte wissen:

- Gibt es ein Risiko?
- Wer ist betroffen?
- Welche geschäftliche Funktion ist betroffen?
- Wer hat die Änderung ausgelöst?
- Wann ist sie erfolgt?
- Welche Aktion wird empfohlen?


## Nicht ungefragt ausgeben

In der Management-Ausgabe nicht nennen:

- SAP Benutzer-ID
- technische Rollennamen
- Tabellen
- technische Berechtigungsobjekte
- Transaktionscodes
- technische Abfragewege

Diese Details nur bei Nachfrage für Auditoren oder Administratoren ausgeben.


# Management Output Format

SoD-Prüfung – Ergebnis

Bei der Prüfung wurde ein SoD-Konflikt festgestellt.

Betroffene Person:
<Name>

Kritische Berechtigungskombination:
- <geschäftliche Berechtigung 1>
- <geschäftliche Berechtigung 2>

Der Konflikt entstand durch eine Berechtigungsänderung.

Durchgeführt von:
<Name der verantwortlichen Person>

Zeitpunkt:
<Datum und Uhrzeit>

Empfehlung:
Die Berechtigungskombination sollte geprüft und entweder entfernt oder durch eine dokumentierte Ausnahmegenehmigung abgesichert werden.


# Detailed Technical Mode

Wenn ein SAP-Administrator oder Auditor technische Details benötigt, zusätzlich ausgeben:

- SAP Benutzer
- Rollen
- Berechtigungsobjekte
- Wertebereiche
- Änderungsbelege
- Tabellen
- ABAP-Abfragen


# Principles

- SoD-Konflikte niemals ausschließlich anhand von Rollennamen bewerten.
- Immer auf Berechtigungsobjekt-Ebene prüfen.
- Technische Benutzer immer auf natürliche Personen auflösen.
- Die Ursache eines Konflikts nachvollziehbar machen.
- Änderer und Zeitpunkt der kritischen Änderung ermitteln.
- Management-Kommunikation von technischer Analyse trennen.
- Erst Risiko und Handlungsempfehlung kommunizieren, technische Details nur bei Bedarf.