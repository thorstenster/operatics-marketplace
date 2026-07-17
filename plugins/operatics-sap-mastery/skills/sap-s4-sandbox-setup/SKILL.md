---
name: sap-s4-sandbox-setup
description: Use this skill whenever a long-dormant or freshly-provisioned S/4HANA test/sandbox system throws errors related to closed posting periods (MM, FI, Material Ledger), incomplete material master data (missing views like Disposition, Bestandsführung, Buchhaltung), missing shipping point determination, missing/expired tax condition records, or when extending the item table control (VBAP/VBAPD) on classic sales-order screen 4900 or adding a custom function code to the VA01/VA02/VA03 GUI status. Trigger on messages like M7053, F5286, F5201, FINS_ML_UTIL002, VU019, V1801, V1216, V1382, V1505, V0104, V0102, V0306, or on requests to add fields to the item overview table control, propagate a header field to all item rows, or add a custom pushbutton/menu function that navigates away and back (STARTING NEW TASK) from VA02. Also consult this skill proactively when standing up a brand-new S/4 sandbox, since these are the most common "first transaction fails" blockers in an aged or never-productive test system.
---

# S/4HANA Sandbox Setup & Sales-Order Extension Gotchas

Gesammeltes Wissen aus der Reparatur eines seit ca. 1998–2023 eingefrorenen Testsystems (Buchungskreis 0001) plus einer klassischen VBAK/VBAP-Dynpro-4900-Erweiterung samt eigenem Fcode. Alle Punkte sind auf Thorstens S4S-System (SAP_BASIS 758, SP0002) verifiziert.

## 1. Periodensteuerung — drei unabhängige Ebenen

Ein eingefrorenes Testsystem blockiert fast immer an **mehreren** Periodenständen gleichzeitig, nicht nur an einem. Alle drei prüfen, bevor man sich auf eine einzelne Fehlermeldung stürzt:

### 1a. Materialwirtschaftsperiode (MM)
- **Symptom:** M7053 „Buchen nur in Perioden X/Y möglich"
- **Tabelle:** `MARV` (`BUKRS`, `LFGJA`, `LFMON`)
- **Fix:** `MMPI` (Periodeninitialisierung), **nicht** `MMPV`. `MMPV` verschiebt immer nur exakt eine Periode pro Aufruf und bricht bei großem Rückstand mit „Falsche Periode im Verwaltungssatz … hier keine Umsetzung" ab, ohne die Periode zu ändern — bei >300 Perioden Rückstand ist das kein gangbarer Weg.
- `MMPI` setzt die Periode direkt, vorausgesetzt es existieren noch keine periodenrelevanten MM-Belege im Buchungskreis (bei einem nie produktiv genutzten Testsystem so gut wie immer unkritisch).

### 1b. Material Ledger Produktivstart
- **Symptom:** `FINS_ML_UTIL002` „ML muss für den Bewertungskreis X manuell produktiv gesetzt werden"
- Seit S/4HANA ist ML technisch immer aktiv, muss aber pro Bewertungskreis einmalig **produktiv gesetzt** werden.
- **Transaktion:** `CKMSTART`, Bewertungskreis (= i. d. R. Werk) eingeben.
- **Falle:** Die Transaktion unterscheidet Testlauf/Echtlauf. Ein Testlauf hinterlässt keine Wirkung — die Meldung kommt danach identisch wieder. Prüfen, ob wirklich ausgeführt und nicht nur geprüft wurde.
- **Reihenfolge beachten:** Erst MM-Periode korrigieren (1a), dann CKMSTART — CKMSTART bezieht sich intern auf die aktuelle MM-Periode zum Ausführungszeitpunkt.
- Konfigurationsvoraussetzung (meist schon vorhanden, sonst zuerst nötig): `OMX2` (ML-Typen/Währungstypen), `OMX1`/`OMX3` (Bewertungskreis → ML-Typ-Zuordnung), sichtbar in Tabelle `TCKM2` (`BWKEY` → `MATLED`).

### 1c. FI-Buchungsperiode
- **Symptom:** F5286 „Periode X ist für Kontoart Y und Hauptbuch Z nicht geöffnet"; danach oft zusätzlich F5201 „… für Variante V und Kontoart + nicht geöffnet"
- **Tabelle:** `T001B`, geschlüsselt über `BUKRS` + `MKOAR` (Kontoart) + Kontenbereich (`VKONT`/`BKONT`) + `BRGRU` (Berechtigungsgruppe Sonderperioden).
- **Prüflogik des Systems:** Erst wird nach einem Eintrag für die **spezifische Kontoart** der Buchung gesucht (z. B. `S` für Sachkonten, `D` für Debitoren, `K` für Kreditoren). Erst wenn keiner existiert, greift der generelle **`+`**-Fallback-Eintrag.
- **Falle 1:** Ein System kann für dieselbe Kontoart/denselben Buchungskreis **mehrere** `+`- oder kontoartspezifische Einträge haben, die sich nur in `BRGRU` unterscheiden (Berechtigungsgruppe für Sonderperioden). Der Eintrag **ohne** Berechtigungsgruppe gilt für normale User — wenn der abgelaufen ist, hilft ein zweiter, weiter offener Eintrag mit spezieller Berechtigungsgruppe nichts, außer der User hat diese Berechtigung.
- **Falle 2:** F5286 (spezifische Kontoart) zu beheben reicht oft nicht — direkt danach kann F5201 für eine andere Kontoart oder den `+`-Fallback kommen, weil der Beleg mehrere Kontoarten gleichzeitig bebucht (z. B. Sachkonto + Debitor in einer Faktura).
- **Fix:** `OB52`, betroffene Zeile(n) suchen, Zeitraum 1 (Von/Bis Periode+Jahr) erweitern.
- **Praxistipp:** Wenn ein System an einer Stelle so lange eingefroren war, lohnt sich ein einmaliger Rundumschlag über alle Kontoarten in `OB52` statt iterativem Nachbessern pro Fehlermeldung.

## 2. Materialstamm — fehlende Sichten bei Minimal-Testmaterialien

Testmaterialien, die nur für einen einzelnen Verkaufsbeleg angelegt wurden, haben oft nur Grunddaten- + Verkaufssicht. Folgefehler und Fixes in typischer Reihenfolge:

| Symptom | Fehlende Sicht | Tabelle (leer/fehlend) |
|---|---|---|
| Lagerortbestand nicht auffindbar | Disposition + Bestandsführung, Lagerort | `MARD` |
| M7090 „Daten der Buchhaltung … noch nicht gepflegt" | Buchhaltung 1 | `MBEW` |
| V1382 „Material X in VkOrg/VtWeg/Sprache nicht vorgesehen" | Verkauf: Vertriebsorg. Daten 1 | `MVKE` |

**Zentrale Falle bei allen dreien:** Die Sichtenauswahl-Checkbox-Liste in MM02 erscheint nur **einmalig automatisch** beim ersten Öffnen. Ist das Material schon offen (z. B. auf der Verkaufssicht), muss sie über **Menü „Zusätze → Sichten auswählen"** (oder das entsprechende Symbol in der Werkzeugleiste) explizit erneut aufgerufen werden — sonst wirkt es, als gäbe es die Sicht schlicht nicht.

**Weitere Falle:** Scheitert die Sicherung einer neu hinzugefügten Sicht an einem Folgefehler (z. B. Buchhaltung 1 an FINS_ML_UTIL002, siehe 1b), wird die Sicht **nicht** teilweise angelegt — der komplette Sicherungsvorgang bricht ab. Nach Beheben der eigentlichen Ursache muss die Sichtenauswahl + Dateneingabe **komplett wiederholt** werden, nicht nur erneut gespeichert.

**Bewertungskreis:** In Standard-Konfiguration meist identisch mit dem Werk — beim Organisationsebenen-Popup für Buchhaltung 1 reicht i. d. R. die Werkseingabe, der Bewertungskreis wird automatisch abgeleitet.

## 3. Bestand zubuchen (Testdaten)

`MIGO`, Kopfzeile auf **„Warenbewegung / Sonstige"**, Bewegungsart **`561`** (Anfangsbestand ohne Bezug — bewegt Menge und Wert ohne Referenz auf Bestellung/Lieferung, unkritisch für FI). Pflichtfelder: Material, Menge, ME, Werk, Lagerort. Zeile per Ampel/Kästchen aktivieren, „Prüfen" vor „Buchen".

Setzt voraus: Lagerort-Sicht (2) UND Buchhaltungssicht (2) UND offene MM-Periode (1a) UND produktives ML (1b) — alle vier gleichzeitig, sonst kommt der jeweils nächste Fehler in der Kette.

## 4. Versandstellenfindung

Kette: `VBAK-VSBED` (Versandbedingung, Kopf) ← normalerweise kopiert aus `KNVV-VSBED` (Kundenstamm/Business Partner, Verkaufsbereichsdaten „Versand") + Ladegruppe (Material) + Werk → Tabelle **`TVSTZ`** → Versandstelle.

- **Symptom bei fehlender Versandstelle:** VU019 „Fehlende Daten: Versandstelle/Annahmestelle"
- **Root Cause typischerweise:** `KNVV-VSBED` beim Kunden ist leer → Auftrag erbt keine Versandbedingung → `TVSTZ`-Lookup findet keinen Eintrag.
- **Fix nachhaltig:** In S/4HANA über **Transaktion `BP`**, nicht VD02 (obsolet) — Rolle „Kunde"/Verkauf (FLVN00) wählen, Verkaufsbereich, Reiter „Verkauf" → „Versand" → Versandbedingung pflegen.
- **Fix pro Beleg:** In VA02, Kopfbild Reiter „Versand", Versandbedingung manuell nachtragen — wirkt nur für bereits existierende Belege, nicht rückwirkend durch reine Stammdatenänderung.

## 5. MWST-Preisfindung — abgelaufene Testkonditionen

- **Symptom:** V1801 „Obligatorische Kondition MWST fehlt"
- **Konditionstabelle:** `A002` (Land + Kundensteuerklassifizierung `TAXK1` + Materialsteuerklassifizierung `TAXM1`) — nicht zu verwechseln mit `A003` (reine Länder+Steuerkennzeichen-Tabelle) oder Export-Steuertabellen (Länderpaar `ALAND`/`LLAND`).
- Kundensteuerklassifizierung: `KNVI` (`TATYP='MWST'`, `TAXKD`). Materialsteuerklassifizierung: `MLAN` (`TAXM1`) — **nicht** `MVKE`, wie man vermuten könnte.
- Klassisches Testsystem-Muster: Konditionssätze in `A002`/`KONP` existieren, sind aber alle mit `DATBI` weit in der Vergangenheit befristet (z. B. 31.12.1999) — Klassifizierung stimmt, Gültigkeit nicht.
- **Fix:** `VK12`, Bildvariante **„Steuern Inland"** (nicht „Steuern Export" — das ist die Länderpaar-Variante), Gültigkeitszeitraum verlängern.
- **Wichtige Sackgasse, die man sich sparen kann:** Manuelles Nachtragen der Kondition MWST direkt im Beleg schlägt mit V1216 fehl. Grund: `T685A-KMANU = 'D'` bei Steuerkonditionen (`KOAID='D'`) — manuelle Eingabe ist bei Steuerkonditionen bewusst gesperrt, einziger Weg ist der Konditionssatz über VK12.

## 6. VBAP/VBAK erweitern: Zusatzfeld in Dynpro 4900 (Item-TC)

Szenario: Eigenes Feld in VBAK und VBAP ergänzt, soll im Positions-Table-Control von VA02 (Dynpro 4900, SAPMV45A) angezeigt und bei Kopfänderung in alle Zeilen propagiert werden.

### 6a. Reine Anzeige (VBAPD ist transient)
- `VBAPD` ist die transiente Anzeige-Struktur für Dynpro 4900 — **kein** `USEREXIT_MOVE_FIELD_TO_VBAPD` existiert dafür (anders als bei VBAK/VBAP).
- Feld in der PBO-Loop-Ablauflogik in die FIELD-Liste aufnehmen (Modifikation, da SAPMV45A Standardcode ist — SSCR-Zugriffsschlüssel nötig).
- Befüllung über eigenes PBO-Modul, **Include `MV45AOZZ`** — nicht `MV45AIZZ` (I = Input/PAI, O = Output/PBO — leicht zu verwechseln).
- Datentransport zum Dynpro läuft über die Workarea `vbap`, **nicht** `xvbap` — Zuweisung an `xvbap` im PBO-Modul verpufft für die Anzeige.
- **Leerzeilen-Guard zwingend:** Die TC-Loop läuft auch über Füllzeilen unterhalb der letzten Position. `CHECK vbap-posnr IS NOT INITIAL` filtert das zuverlässig — jede Zeile aus XVBAP hat spätestens nach dem ersten PAI (Enter) eine POSNR, auch vor dem Sichern.

```abap
MODULE zz_feld_fuellen OUTPUT.
  CHECK vbap-posnr IS NOT INITIAL.
  vbap-zzfeld = vbak-zzfeld.
ENDMODULE.
```

### 6b. Persistenz beim Sichern
- `USEREXIT_MOVE_FIELD_TO_VBAK` (MV45AFZZ) läuft am Ende von `VBAK_FUELLEN` — Name irreführend (klingt nach „Feld nach VBAK", tatsächlich nur „Zeitpunkt: Kopf fertig aufgebaut"), eignet sich aber als Trigger-Punkt für Kopf→Position-Propagierung, weil dort der komplette Beleg als `xvbap` vorliegt.
- **Sackgasse, die V1505 erzeugt:** Direktes `MODIFY xvbap` + manuelles Setzen von `updkz = 'U'` umgeht die Standard-Änderungslogik. Die Sicherung sucht dann in `YVBAP` (Vorher-Abbild-Tabelle) nach einem Eintrag, der nie geschrieben wurde, weil `YVBAP` nur über den offiziellen Weg (`VBAP_BEARBEITEN`) befüllt wird → „Position 000010 fehlt in Tabelle YVBAP".
- **Korrekt:** Über `PERFORM vbap_bearbeiten` gehen — das pflegt `updkz` UND `YVBAP` konsistent selbst:

```abap
FORM userexit_move_field_to_vbak.
  LOOP AT xvbap WHERE updkz NE 'D'.
    CHECK xvbap-zzfeld NE vbak-zzfeld.
    vbap        = xvbap.
    svbap-tabix = sy-tabix.
    vbap-zzfeld = vbak-zzfeld.
    PERFORM vbap_bearbeiten.
  ENDLOOP.
ENDFORM.
```

- **Falle bei Navigation ohne Speichern** (z. B. Kopf-Zusatzdaten B ändern, dann zur Übersicht springen ohne zu sichern): `USEREXIT_MOVE_FIELD_TO_VBAK` feuert aus einem laufenden Verarbeitungszyklus heraus, globale Workareas (`vbap`, `*vbap`, `svbap`) können in einem inkonsistenten Zwischenzustand hängen → wieder V1505, diesmal auch bei bestehenden (nicht neuen) Positionen. **Fix:** Loop stattdessen in `USEREXIT_SAVE_DOCUMENT_PREPARE` (MV45AFZZ) — dort ist der Beleg vollständig aufgebaut, kein Zyklus mehr aktiv. Zusätzlich Vorher-Abbild explizit setzen: `*vbap = xvbap.` vor der Änderung.

## 7. Eigener Fcode im Verkaufsbeleg-GUI-Status (z. B. Absprung in Fremdtransaktion)

Szenario: Eigener Button/Menüpunkt in VA02, der z. B. eine Lieferung zum Auftrag sucht und in eine externe Transaktion abspringt, ohne die aktuelle Bearbeitung zu unterbrechen.

### 7a. Fcode-Dispatch
Zentrale Fcode-Verarbeitung in SAPMV45A dispatcht dynamisch: `PERFORM ('FCODE_' && fcode) IN PROGRAM SAPMV45A IF FOUND`. Für Fcode `ZPACK` also einfach:
```abap
FORM fcode_zpack.
  " Logik
ENDFORM.
```
in MV45AFZZ anlegen — kein CASE-Verzweigung nötig, kein Registrieren des Formnamens irgendwo sonst.

### 7b. Zulässigkeitsprüfung — T185F
Bevor die Form überhaupt erreicht wird, prüft das System den Fcode gegen die Bildfolgesteuerung (`SCREEN_SEQUENCE_CONTROL`). Fehlt der Eintrag: **V0104** „Die angeforderte Funktion X ist hier nicht vorgesehen".

- **Transaktion:** `VFBS`.
- **Programmname (AGIDV): `SAPMV45B`** — nicht SAPMV45A! Obwohl die Fcode-Form in SAPMV45A/MV45AFZZ liegt, ist die Bildfolgesteuerung nach dem GUI-Status-Trägerprogramm SAPMV45B geschlüsselt („Dummy Program for New Interface" — Anker für Status UND Folgebildsteuerung).
- Pro erlaubtem **Aktivitätstyp** (`H`=Hinzufügen/VA01, `V`=Verändern/VA02, `A`=Anzeigen/VA03) einen eigenen T185F-Eintrag.
- **Funktionstyp:** leer lassen (keine Sonderfunktion, die ins Bildfolge-Framework eingreift).
- **Dialogart:** leer lassen (normaler PAI-Durchlauf, keine Sonderbehandlung wie bei Abbrechen-Funktionen).

### 7c. Navigationsziel — T185
Selbst eine Funktion, die auf dem aktuellen Bild bleiben soll, braucht einen expliziten T185-Eintrag — „bleib stehen" ist kein Default.
- Bei Wildcard-Schlüsseln (`* * * *`, funktioniert bildübergreifend) muss **Parameterherkunft = `A`** gesetzt sein (dynamisch aus aktuellem Kontext). `space` oder `T` scheitern an fehlenden/nicht auflösbaren Zielparametern bei generischen Einträgen.
- Warnung **V0306** „Quelle = F erfordert eine FORM-Routine ohne Folge-Parameter" heißt: Quelle `F` (dynamische Folgebildbestimmung per FORM) und gleichzeitig statisch eingetragene Folge-Parameter widersprechen sich — bei einem einfachen „bleib stehen"-Fcode Quelle **leer** lassen, Folge-Panel = aktuelles Panel, FORM-Routine leer.
- **Fehlende Feldliste am schnellsten ermitteln:** Fcode einmal auslösen, resultierende V0102 „Kein Eintrag in Tabelle T185 für & & &" liefert die exakten Schlüsselwerte für den nötigen Eintrag direkt in der Meldung.

### 7d. Trotz korrektem T185/T185F: Transaktion wird nach der Form sofort beendet
Die reine T185-Konfiguration reicht bei Wildcard-Einträgen (`* * * *` mit Herkunft `A`) manchmal nicht — die Folgebildauflösung kann ein leeres Ziel liefern, was das Rahmenprogramm als `LEAVE TO SCREEN 0` interpretiert.
**Robuster Fix direkt in der Form**, unabhängig von der T185-Auflösung:
```abap
FORM fcode_zpack.
  " ... eigentliche Logik ...
  PERFORM folge_gleichsetzen(saplv00f).
  fcode = 'ENT1'.
  SET SCREEN syst-dynnr.
ENDFORM.
```
`FOLGE_GLEICHSETZEN` (Funktionsgruppe `SAPLV00F`, derselben Bibliothek wie `SCREEN_SEQUENCE_CONTROL`) setzt den Folge-Verarbeitungsort programmgesteuert gleich dem aktuellen. `fcode = 'ENT1'` neutralisiert den Code für die Nachverarbeitung, `SET SCREEN syst-dynnr` nagelt das Folgedynpro explizit fest.

### 7e. Absprung in Fremdtransaktion ohne Kontextverlust
- `CALL FUNCTION ... STARTING NEW TASK` (aus einem RFC-fähigen Funktionsbaustein heraus) statt direktem `CALL TRANSACTION` in der Form selbst — öffnet ein neues SAPGUI-Fenster/neuen internen Modus, VA02 bleibt mit Sperre und ungesichertem Stand unangetastet.
- **Parameterübergabe:** `SET PARAMETER ID` muss **im aufgerufenen Funktionsbaustein** passieren, nicht in der aufrufenden Form — die neue Task ist eine eigene Session, weder ABAP Memory noch SAP Memory der Ursprungssession sind dort verlässlich verfügbar. Die Nummer als regulärer FB-Importparameter mitgeben.
- Funktionsbaustein muss als **remote-fähig** markiert sein (Pflicht für `STARTING NEW TASK`).
- Modus-Limit beachten (`rdisp/max_alt_modes`, Standard 6) — bei ausgeschöpftem Limit öffnet sich kommentarlos kein neues Fenster.

## 8. Diagnose-Reihenfolge, wenn "es einfach nicht geht"

Bei mysteriösen Abbrüchen ohne aussagekräftige Meldung: Breakpoint auf ABAP-Anweisung (`LEAVE`, `SET SCREEN`) setzen, Aktion auslösen, Aufrufstack (Views → Stack) anschauen — verrät zuverlässiger als Vermutungen, ob der Abbruch aus der eigenen Form, der Fcode-Nachverarbeitung oder der Bildfolgesteuerung kommt.

Bei Tabellenwerten, die man nicht sicher zuordnen kann (z. B. unklarer Formname für einen Verarbeitungsschritt): Breakpoint auf ein Feld setzen, das bereits heute zuverlässig befüllt wird, und im Debugger den Aufrufpfad ablesen — zuverlässiger als sich auf Namenskonventionen aus dem Gedächtnis zu verlassen, die je nach SAP_BASIS-Release variieren können.
