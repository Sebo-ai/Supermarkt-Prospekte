# Projektanweisung – Supermarkt-Prospekte Aachen (v3)

## MÄRKTE (jede Woche prüfen)
1. EDEKA, Filiale Vieler, Aachen
2. HIT (Aachen), Vaalser Straße
3. Netto Marken-Discount, Filiale Boxgraben, Aachen
4. REWE, Filiale Stenten, Aachen
5. Aldi Süd (Aachen)

---

## GOLDENE REGEL – ABLAUF

**Erst ALLE 5 Märkte vollständig erfassen und in rohdaten/KW[XX].md speichern, DANN erst angebote.md und index.html schreiben. Niemals nach dem ersten oder zweiten Markt mit dem Schreiben beginnen.**

**Rohdaten-Datei zuerst anlegen:** Vor dem Scan `/rohdaten/KW[XX].md` aus der VORLAGE.md erstellen. Bei Kontextverlust kann eine neue Session dort weiterlesen, ohne von vorne zu beginnen.

---

## SCHRITT 0 – Kalenderwoche bestimmen

Bestimme immer die Prospekte der **nächsten Woche** (ab kommendem Montag). Alle Dateinamen, Header und KW-Angaben beziehen sich auf diese Woche.

---

## SCHRITT 1 – Chrome-Verfügbarkeit prüfen

Rufe `mcp__claude-in-chrome__list_connected_browsers` auf.

- **Chrome verbunden** → direkt weiter
- **Chrome nicht verbunden** → Benachrichtigung + 10 Min. pollen (alle 60 Sek.) → nach 10 Min.: 5 Min. Pause → erneute Benachrichtigung → nächster Versuch. Nach 3 Versuchen (~45 Min.): Abbruch, Fehlerdatei schreiben, NICHTS an angebote.md oder index.html ändern.

---

## SCHRITT 2 – Rohdaten-Datei anlegen

Vor dem ersten Markt-Scan:
```
cp rohdaten/VORLAGE.md rohdaten/KW[XX]_[YYYY-MM-DD].md
```
In diese Datei werden alle Rohdaten eingetragen. Sie ist der Sicherheitsanker.

---

## SCHRITT 3 – Alle 5 Märkte erfassen

### Reihenfolge und Quellen

| Markt | Primärquelle | Tool | Fallback |
|-------|-------------|------|---------|
| NETTO Boxgraben | kaufda.de/Aachen → „Netto" | Web-Fetch; bildbasiert: Chrome | netto.de |
| REWE Stenten | rewe.de/angebote/ | Web-Fetch; unvollständig: Chrome | kaufda.de „REWE Aachen" |
| HIT Vaalser Str. | kaufda.de/Aachen → „HIT" | **ZWINGEND Chrome** – immer bildbasiert | hit.de Filiale Aachen |
| EDEKA Vieler | **BEIDE**: kaufda.de/marktguru.de + edeka.de/maerkte/071409/angebote/ | Web-Fetch + **Chrome** | marktguru.de |
| ALDI Süd | **NEUE METHODE** (s. unten) | **ZWINGEND Chrome** | marktguru.de „Aldi Aachen" |

---

### ALDI Süd – Korrigierte Scan-Methode (v3)

**Nie mehr aldi-sued.de/de/angebote.html verwenden** – zeigt unvollständige nationale Übersicht.

**Korrekte Methode:**
1. Navigiere zu `https://www.aldi-sued.de/angebote` (Chrome, JS erforderlich)
2. Suche den Link „Alle anzeigen" → lese seine href aus → führt zu URL wie:
   `https://www.aldi-sued.de/produkte/wochenangebote/k/[ID]`
3. Navigiere Seite 1, 2, 3 durch: `?page=1`, `?page=2`, `?page=3`
   (Anzahl Seiten wöchentlich prüfen – kann variieren)
4. Auf jeder Seite extrahieren mit:
   ```javascript
   var tiles = Array.from(document.querySelectorAll('.product-tile__link'));
   var seen = new Set();
   var results = [];
   tiles.forEach(t => {
     var raw = t.textContent.replace(/\s+/g,' ').trim();
     if(seen.has(raw.slice(0,50)) || raw.length < 10) return;
     seen.add(raw.slice(0,50));
     results.push(raw);
   });
   results.join('\n');
   ```
5. Kein CookieBot-Problem mit diesem Selektor.
6. Typisch: ~70–90 Produkte über 3 Seiten.

**Datenformat verstehen:**
`[Kategorie-Label][Marken-Label][Produktname] [Menge][Grundpreis] [Spare X%][Aktionspreis]² [UVP]`
Beispiel: `KühlungMEINE METZGEREIHähnchen-Schenkel 2 kg 2 kg(3,50 €/1 kg) 6,99 €`
- Kategorie-Label: `Kühlung`, `Tiefkühlung`, `Vegan`, `BIO NATURLAND`, `NATUR LIEBLINGS` etc.
- Spare X%: nur bei Aktionspreis vorhanden; sonst direkt Preis ohne Streichpreis

---

### EDEKA Vieler – Zwei Pflicht-Quellen

**Quelle A (national):** kaufda.de oder marktguru.de → EDEKA Aachen
**Quelle B (filialspezifisch):** `https://www.edeka.de/maerkte/071409/angebote/` via Chrome

Extraktionsmethode für Quelle B:
```javascript
// Namen
var names = Array.from(document.querySelectorAll('h2,h3,h4'))
  .map(h => h.textContent.trim()).filter(t => t.length > 2);
// Preise (ACHTUNG: CookieBot blockt .innerText auf Body!)
var spans = Array.from(document.querySelectorAll('span'));
var prices = [];
spans.forEach(s => {
  var t = s.textContent.trim();
  if(t.match(/^\d[,\d\.]+$/) && parseFloat(t.replace(',','.')) < 50) prices.push(t);
});
```

**App-Preise:** Zwei Preise pro Produkt – App-Preis (günstigerer) zuerst, dann regulär.
Beispiel Spans: `["1,49","1,69"]` → App €1,49 / regulär €1,69. App-Preis als Hauptpreis eintragen.

---

### Pro Markt: Rohliste ZUERST (OHNE Kategorisierung)

Für jeden Markt zuerst ALLE Produkte + Preise in die Rohdaten-Datei eintragen:
- Produktname vollständig (Marke + Variante)
- Aktionspreis
- Streichpreis / UVP falls angegeben
- Menge / Grundpreis
- Gültigkeitszeitraum falls abweichend

**Dann Favoriten-Check:** Explizit für alle Favoriten-Kategorien prüfen. Fehlende → als „kein Angebot" notieren.

**NIEMALS erfassen:** Tiernahrung (Hunde-, Katzen-, sonstiges Tierfutter)

---

## SCHRITT 4 – Archivieren

Bestehende `angebote.md` → `/archiv/angebote_[YYYY-MM-DD].md` (heutiges Datum).

---

## SCHRITT 5 – angebote.md schreiben (EINMALIG, vollständig)

Erst wenn die Rohdaten-Datei für alle 5 Märkte als vollständig markiert ist.
In einem einzigen Durchgang schreiben. Kein nachträgliches Ergänzen.

**Preisvergleich zur Vorwoche:** Archivdatei einlesen → gleiche Produkte vergleichen.

**Struktur:**
1. Header: KW, Datum, Gültigkeitszeitraum, alle 5 Märkte
2. ⭐ Favoriten-Block (marktübergreifend, nach günstigstem Preis sortiert)
3. Alle Kategorien (s. Kategorisierungssystem)
4. 📋 Hinweise / Fehler
5. 📊 Zusammenfassung-Tabelle

---

## SCHRITT 6 – index.html aktualisieren

- Nur Dateninhalte (P.push-Einträge) ersetzen
- Layout, Farbschema, Suche/Filter unverändert lassen
- Kategorien und Favoriten synchron zur angebote.md

---

## SCHRITT 7 – GitHub Push

```bash
bash push.sh "KW [XX] – Wöchentliches Update [Datum]"
```

push.sh erkennt Workspace-Pfad automatisch (Mac + Sandbox). Commit-Hash bestätigen.

---

## FAVORITEN
(marktübergreifend, günstigsten Preis zuerst, ganz oben im Dokument)

- Milch (inkl. Butter, Sahne, Buttermilch)
- Eier
- Skyr
- Joghurt & Süße Joghurts
- Geflügel (Hähnchen, Pute)
- Rind / Kalb
- Barilla- und De-Cecco-Nudeln (inkl. Saucen)

**Doppelt aufführen:** einmal im Favoriten-Block oben, einmal in der jeweiligen Kategorie.

---

## KATEGORISIERUNG

Exakte Kategorie-Schlüssel verwenden:

### 🌿 Frische & Kühlung
- **Obst** | **Gemuese** | **Fleisch** | **Wurst** | **Fisch** | **Milch** | **Kaese** | **Eier**

### ❄️ Tiefkühl
- **TKGemuese** | **TKFleisch** | **TKFertig**

### 🥫 Trockenware & Vorrat
- **Brot** | **Nudeln** | **Konserven** | **Fruehstueck** | **Oel** | **Backen**

### 🥤 Getränke
- **Wasser** | **Softdrinks** | **KaffeeTee** | **Alkohol**

### 🍫 Snacks & Süßes
- **Suesses** | **Snacks** | **Eis**

### 🧹 Haushalt & Drogerie
- **Reinigung** | **Koerperpflege**

---

## PRO PRODUKT ANZUGEBEN
- Name (vollständig mit Marke und Variante)
- Preis (Aktionspreis)
- Menge / Grundpreis
- Preisvergleich zur Vorwoche

---

## AUSGABE-FORMAT (angebote.md)

```
# Supermarkt-Angebote Aachen — KW [XX] · [TT.MM.]–[TT.MM.YYYY]

**Erfasst:** [Datum] | **Gültig:** [Zeitraum]
**Märkte:** E center Vieler (EDEKA), HIT Aachen, REWE Stenten, NETTO Boxgraben, ALDI Süd

---

## ⭐ Favoriten
### 🥛 Milch — sortiert nach Preis
| Produkt | Markt | Preis | Menge | Vorwoche |

[alle Favoriten-Kategorien]

---

[alle Kategorien]

---

## 📋 Hinweise / Fehler

## 📊 Zusammenfassung
| Markt | Angebote | Favoriten-Treffer | Status |
```

---

## MOBILE-ANFORDERUNGEN für index.html
- Vollständig responsiv (ab ca. 360px), komplett OFFLINE-fähig
- Keine externen CDN-Links, Google Fonts, externe Scripts
- Touch-Ziele mind. 44×44px, Schriftgröße mind. 16px
- Favoriten-Sektion ganz oben, Suchfeld und Filter sticky
- Kategorien als Accordion, dunkles Farbschema
- Preisvergleich farblich: grün = günstiger, rot = teurer

---

## QUALITÄTSKONTROLLE (Checkliste vor dem Push)
- [ ] Chrome war verbunden (sonst: abgebrochen, nichts geschrieben)
- [ ] rohdaten/KW[XX].md angelegt und alle 5 Märkte als vollständig markiert
- [ ] ALDI: Neue Methode (wochenangebote/k/[ID] Seiten 1–3) verwendet
- [ ] EDEKA: BEIDE Quellen (kaufda/marktguru + edeka.de/maerkte/071409/) geprüft
- [ ] Alle Favoriten-Kategorien pro Markt geprüft, fehlende notiert
- [ ] Tiernahrung NICHT erfasst
- [ ] Vorwochen-Vergleich aus Archiv durchgeführt
- [ ] Alte angebote.md archiviert VOR der neuen
- [ ] angebote.md in EINEM Durchgang geschrieben
- [ ] index.html aktualisiert (P.push-Einträge, nicht Layout)
- [ ] push.sh ausgeführt, Commit-Hash bestätigt

Lieber zu ausführlich als zu knapp – lieber ein Angebot zu viel als eines zu übersehen.
