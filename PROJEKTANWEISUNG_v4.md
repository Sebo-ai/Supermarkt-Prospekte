# Projektanweisung – Supermarkt-Prospekte Aachen (v4)

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
| NETTO Boxgraben | `netto-online.de/filialangebote/1` | Chrome JS-Extraktion | kaufda.de „Netto Aachen" |
| REWE Stenten | `rewe.de/angebote/aachen/14/rewe-markt-krugenofen-62-70/` | **Chrome + Scroll-Pflicht** | kaufda.de „REWE Aachen" |
| HIT Vaalser Str. | `hit.de/maerkte/aachen/prospekte/wochenprospekt?seite=1` | **Chrome, alle Seiten** | hit.de/prospekte |
| EDEKA Vieler | **BEIDE**: kaufda.de/marktguru.de + `edeka.de/maerkte/071409/angebote/` | Web-Fetch + Chrome | marktguru.de |
| ALDI Süd | `aldi-sued.de/produkte/wochenangebote/k/[ID]` Seiten 1–3 | **Chrome** | marktguru.de „Aldi Aachen" |

---

### NETTO Boxgraben – Korrekte Scan-Methode (v4)

**Korrekte URL:** `https://www.netto-online.de/filialangebote/1`
- **OHNE Kategorie-Suffix** — URLs wie `/filialangebote/1/35791` zeigen nur Teilsortiment
- Kein Login nötig, kein CookieBot-Problem

**Extraktion:**
```javascript
var links = Array.from(document.querySelectorAll('a[href*="AddStoreArticle"]'));
var seen = new Set();
var results = [];
links.forEach(a => {
  var url = new URL(a.href);
  var name = url.searchParams.get('Name');
  var price = url.searchParams.get('Price');
  var bundle = url.searchParams.get('BundleText') || '';
  if (!name || seen.has(name.slice(0,30))) return;
  seen.add(name.slice(0,30));
  results.push(name + ' | €' + price + ' | ' + bundle);
});
results.join('\n');
```
Liefert typisch ~130–200 Produkte in einem Durchgang.

---

### REWE Stenten – Korrekte Scan-Methode (v4)

**Korrekte URL:** `https://www.rewe.de/angebote/aachen/14/rewe-markt-krugenofen-62-70/`
- Markt-ID: `14`, Adresse: Krugenofen 62-70

**PFLICHT: Erst komplett scrollen, dann extrahieren.**
Die Seite nutzt virtuelles Scrolling — ohne Scroll-Trigger sind nur ~10 Tiles im DOM (statt ~270).

**Schritt 1 – Scrollen:**
```javascript
for (var pos of [8000, 16000, 24000, 32000, 40000]) {
  window.scrollTo(0, pos);
  await new Promise(r => setTimeout(r, 500));
}
window.scrollTo(0, document.body.scrollHeight);
await new Promise(r => setTimeout(r, 1000));
```

**Schritt 2 – Extraktion (nur echte Aktionspreise):**
```javascript
var tiles = Array.from(document.querySelectorAll('.cor-offer-renderer-tile.cor-link'));
var results = [];
var seen = new Set();
tiles.forEach(tile => {
  var tag = tile.querySelector('.cor-offer-price__tag');
  var tagText = tag?.innerText?.trim() || '';
  if (!tagText.includes('Aktion') && !tagText.includes('Knaller')) return;
  var nameEl = tile.querySelector('.cor-offer-information a');
  var name = nameEl?.innerText?.trim();
  if (!name || seen.has(name.slice(0,30))) return;
  seen.add(name.slice(0,30));
  var priceMatch = tagText.match(/(\d+[,]\d{2})\s*€/);
  var price = priceMatch ? '€' + priceMatch[1] : '';
  var contentDiv = tile.querySelector('.cor-offer-renderer-tile__content');
  var spans = Array.from(contentDiv?.querySelectorAll('span') || []);
  var menge = spans.map(s => s.innerText?.trim())
    .filter(t => t && t !== name && t.match(/\d|g|ml|l|kg|Fl|Stk/i))[0] || '';
  results.push(name + ' | ' + price + ' | ' + menge);
});
results.join('\n');
```

Filter-Logik: `.cor-offer-price__tag` enthält "Aktion" oder "Knaller" → echte Wochenangebote. Dauersortiment (Tiefpreis) wird damit automatisch ausgeschlossen.

Liefert typisch ~150–200 Aktion-Produkte.

**Wenn Prospekt „technisches Problem" zeigt:** Ignorieren — Kategorien (Knalleraktion, Kühlung, etc.) sind trotzdem vollständig geladen.

---

### HIT Vaalser Str. – Korrekte Scan-Methode (v4)

**Korrekte URL:** `https://www.hit.de/maerkte/aachen/prospekte/wochenprospekt?seite=1`
- **NICHT** `/angebote` — diese Seite zeigt nur veraltete Thumbnails ohne aktuelle Preise
- Seiten 1, 2, 3 … bis keine weiteren Produkte mehr erscheinen (typisch ~13 Seiten, ~480 Produkte)
- Seitenzahl wöchentlich prüfen: `?seite=N` iterieren bis leer/404

**Extraktion pro Seite:**

Produktnamen (als Text verfügbar):
```javascript
var cards = Array.from(document.querySelectorAll('.ga_product_detail_view'));
var seen = new Set();
cards.map(card => {
  var name = card.querySelector('h2,h3,h4')?.innerText?.trim();
  if (!name || seen.has(name.slice(0,30))) return null;
  seen.add(name.slice(0,30));
  return name;
}).filter(Boolean).join('\n');
```

Preise (aus Split-Spans, da HIT Preise visuell aufteilt):
```javascript
cards.forEach(card => {
  var spans = Array.from(card.querySelectorAll('span')).map(s => s.innerText?.trim());
  // Format: spans[i] = "X." und spans[i+1] = "YY" → €X,YY
  for (var i = 0; i < spans.length - 1; i++) {
    if (spans[i].match(/^\d{1,2}\.$/) && spans[i+1].match(/^\d{2}$/)) {
      var price = '€' + spans[i].replace('.', '') + ',' + spans[i+1];
      // ... kombinieren mit Produktname
    }
  }
});
```

AKTION!-Preise: `fullText.match(/AKTION!\s*(\d+)\.\s+(\d{2})/)`
DAUER DISCOUNT: Fallback `spans.find(s => s.match(/^\d{1,2}\.\d{2}$/))`

**WICHTIG:** Nie `card.innerText` verwenden → löst [BLOCKED: Cookie/query string data] aus.
Stattdessen: `card.querySelector('h2,h3,h4').innerText` für Namen + einzelne `span.innerText` für Preise.

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

### ALDI Süd – Scan-Methode (v3, unverändert korrekt)

**Nie** `aldi-sued.de/de/angebote.html` — zeigt unvollständige nationale Übersicht.

**Korrekte Methode:**
1. Navigiere zu `https://www.aldi-sued.de/angebote` (Chrome, JS erforderlich)
2. Suche Link „Alle anzeigen" → lese href aus → führt zu `wochenangebote/k/[ID]`
3. Seiten 1, 2, 3 durchgehen (`?page=1`, `?page=2`, `?page=3`)

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

Typisch: ~70–90 Produkte über 3 Seiten.

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
- [ ] NETTO: URL `netto-online.de/filialangebote/1` OHNE Kategorie-Suffix verwendet
- [ ] REWE: Erst komplett gescrollt (8000px-Schritte), dann extrahiert — min. 100 Aktion-Tiles
- [ ] HIT: Prospekt-URL `wochenprospekt?seite=1` verwendet, alle Seiten durchgegangen
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
