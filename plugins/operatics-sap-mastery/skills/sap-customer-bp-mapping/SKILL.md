---
name: sap-customer-bp-mapping
description: Use this skill whenever a Debitor (KUNNR) or Kreditor (LIFNR) muss einem Geschäftspartner (BU_PARTNER) zugeordnet werden, oder umgekehrt. 
WICHTIG - KUNNR und BU_PARTNER sind NICHT automatisch identisch, auch wenn die Nummern zufällig gleich aussehen — sie leben in getrennten Nummernkreisen und müssen über die Customer-Vendor-Integration (CVI) verknüpft werden. Trigger bei Fragen wie "wie heißt der Kunde/Geschäftspartner zu Debitor X", "welcher Geschäftspartner steckt hinter Kreditor Y", "löse KUNNR in BU_PARTNER auf" oder immer dann, wenn ein KUNNR/LIFNR aus einem Beleg (VBAK, EKKO, ...) in einen lesbaren Partnernamen übersetzt werden soll.
---

# Debitor/Kreditor ↔ Geschäftspartner (CVI-Verknüpfung)

## Der Fallstrick

`KUNNR` (Debitor, z. B. aus VBAK) und `BU_PARTNER` (Geschäftspartner, z. B. aus BUT000)
sind **verschiedene Nummernkreise**. Eine identische Ziffernfolge (z. B. beide `0000000001`)
ist **reiner Zufall** und darf niemals als Gleichheit angenommen werden. Die korrekte
Zuordnung läuft immer über eine eigene Verknüpfungstabelle, nicht über einen Vergleich
der Nummern.

## Vorgehen

Immer `query_table` verwenden (kein `run_abap_code` nötig) — zwei Schritte:

**1. Debitor/Kreditor → Partner-GUID**

| Richtung | Tabelle | Schlüsselfeld | Ergebnisfeld |
|---|---|---|---|
| Debitor → Partner | `CVI_CUST_LINK` | `CUSTOMER` = KUNNR | `PARTNER_GUID` |
| Kreditor → Partner | `CVI_VEND_LINK` | `VENDOR` = LIFNR | `PARTNER_GUID` |

**2. Partner-GUID → Geschäftspartnernummer + Name**

Tabelle `BUT000`, Feld `PARTNER_GUID` gegen das Ergebnis aus Schritt 1 abgleichen.
Liefert `PARTNER` (= BU_PARTNER), sowie je nach Partnertyp `NAME_LAST`/`NAME_FIRST`
(Person) oder `NAME_ORG1` (Organisation).

⚠️ Feldliste bei `query_table` auf `BUT000` **schmal halten** (z. B. nur `PARTNER`) —
eine zu breite Feldauswahl kann zu einem Parser-Fehler führen ("Arbeitsbereich hat
weniger Felder als selektiert werden").

## Beispiel (verifiziert auf S4S)

```
Debitor 0000000001
  → CVI_CUST_LINK: CUSTOMER = 0000000001 → PARTNER_GUID = Y8se9InHH9Gggn/2ULGFpQ==
  → BUT000: PARTNER_GUID = Y8se9InHH9Gggn/2ULGFpQ== → PARTNER = 0000000012,
    NAME_ORG1 = "Kundenfutter GmbH"
```

Der naheliegende, aber falsche Kurzschluss wäre gewesen: Debitor `0000000001` =
Geschäftspartner `0000000001` ("Willi Winzig"). Tatsächlich gehört Debitor `0000000001`
zum Geschäftspartner `0000000012` ("Kundenfutter GmbH").

## Merksatz

**KUNNR ≠ BU_PARTNER.** Immer über `CVI_CUST_LINK`/`CVI_VEND_LINK` + `BUT000` auflösen,
nie über Nummerngleichheit raten.
