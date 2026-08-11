# Verifizierte Quellen & Scan-Anleitung
## Supermarkt-Prospekte Aachen — Stand: 11.08.2026

Dieses Dokument enthält für jeden Markt die **getestete, funktionierende** URL,
den richtigen CSS-Selektor und bekannte Fallstricke. Bei jedem Scan zuerst lesen.

---

## 1. ALDI Süd ✅ KORRIGIERTE METHODE

**Problem bisher:** `aldi-sued.de/de/angebote.html` → zeigt nur nationale Highlights,
unvollständig, unstrukturiert.

**Korrekte Methode (getestet 11.08.2026):**

```
Schritt 1: https://www.aldi-sued.de/angebote  (Chrome, JS erforderlich)
Schritt 2: Link "Alle anzeigen" anklicken / Href auslesen
           → führt zu: https://www.aldi-sued.de/produkte/wochenangebote/k/[ID]
           Hinweis: [ID] ändert sich wöchentlich!
Schritt 3: Seiten 1, 2, 3 durchgehen (?page=1, ?page=2, ?page=3)
           Paginierung prüfen – Anzahl Seiten kann wöchentlich variieren.
```

**JavaScript-Extraktion (funktioniert, kein CookieBot-Problem):**
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

**Datenformat:** `[Label][Marke][Produktname] [Menge][Grundpreis] [Spare X%][Aktionspreis]² [UVP]`
Beispiel: `KühlungMEINE METZGEREIHähnchen-Schenkel 2 kg2 kg(3,50 €/1 kg)6,99 €`

**Typische Produktanzahl:** ~70–90 Produkte über 3 Seiten
**Kategorien im Label:** `Kühlung`, `Tiefkühlung`, `Vegan`, `BIO NATURLAND`, `NATUR LIEBLINGS`, etc.
**Tiernahrung:** Nicht im Angebot → kein Filter nötig bei Aldi

---

## 2. EDEKA Vieler ✅ ZWEI PFLICHTQUELLEN

### Quelle A: Nationaler Prospekt (kaufda.de / marktguru.de)
```
URL: https://www.kaufda.de/Hamburg/Aachen (Suche: "EDEKA")
     oder https://www.marktguru.de/m/edeka/ (Standort: Aachen)
Tool: Web-Fetch; falls bildbasiert Chrome
```

### Quelle B: Filialspezifisch — PFLICHT! ← das war die Lücke
```
URL: https://www.edeka.de/maerkte/071409/angebote/
Tool: ZWINGEND Chrome (JavaScript-Rendering)
Markt-ID: 071409 = E center Vieler, Schillerstraße 20-40, 52064 Aachen
```

**Extraktion Filialseite:**
```javascript
// Produktnamen (h2/h3/h4)
var names = Array.from(document.querySelectorAll('h2,h3,h4'))
  .map(h => h.textContent.trim()).filter(t => t.length > 2);

// Preise (spans mit Zahl-Komma-Format)
var spans = Array.from(document.querySelectorAll('span'));
var prices = [];
spans.forEach(s => {
  var t = s.textContent.trim();
  if(t.match(/^\d[,\d\.]+$/) && parseFloat(t.replace(',','.')) < 50) prices.push(t);
});
```

**WICHTIG: App-Preise**
Bei EDEKA gibt es zwei Preise: App-Preis (günstiger) + regulärer Preis.
Im Span-Array erscheint zuerst der App-Preis, dann der reguläre.
Beispiel: `["1,49","1,69"]` → App €1,49 / regulär €1,69
→ In angebote.md den App-Preis als Hauptpreis eintragen, mit Hinweis `(App)`

**Achtung CookieBot:**
`document.body.innerText` und große `.innerText`-Aufrufe werden geblockt!
→ Immer per-Element extrahieren, Batches klein halten (max. 40 Artikel pro Aufruf)

---

## 3. HIT Vaalser Str. ✅

```
Primär:  https://www.kaufda.de → Aachen → "HIT"
         oder https://www.marktguru.de (Suche HIT Aachen)
Tool:    ZWINGEND Chrome – Prospekt ist immer bildbasiert
Fallback: https://www.hit.de → Filialsuche Aachen → Angebote
```

**Hinweis:** HIT hat eine eigene App mit App-Preisen (oft -15% auf regulär).
Im Prospekt erscheinen beide Preise. App-Preis in angebote.md separat ausweisen.

---

## 4. REWE Stenten ✅

```
Primär:  https://www.rewe.de/angebote/
Tool:    Web-Fetch (oft ausreichend)
         Falls unvollständig: Chrome nachschalten
Fallback: https://www.kaufda.de → Aachen → "REWE"
```

**REWE-Besonderheit:** Eigenmarken (ja! Landliebe, Weihenstephan etc.) und
Markenprodukte gemischt. Tab-Struktur auf rewe.de beachten.

---

## 5. NETTO Boxgraben ✅

```
Primär:  https://www.kaufda.de → Aachen → "Netto"
Tool:    Web-Fetch; falls bildbasiert Chrome
Fallback: https://www.netto.de → Filialsuche → Prospekt
```

**NETTO-Besonderheit:** Viele Preise mit Streichpreis (UVP) und %-Rabatt.
"Dauertiefpreis" = kein Aktionspreis, trotzdem erfassen.
Fr+Sa-Sonderangebote explizit mit Hinweis kennzeichnen.

---

## Allgemeine Fallstricke

| Problem | Ursache | Fix |
|---------|---------|-----|
| `[BLOCKED: Cookie/query string data]` | CookieBot blockiert `.innerText` auf großen Elementen | Per-Element mit `textContent`, kleine Batches |
| EDEKA leitet auf Marktsuche um | Session nicht gesetzt | Direkt-URL `/maerkte/071409/angebote/` verwenden |
| Aldi zeigt nur 12 Produkte | Falsche URL (nationale Übersicht) | Wochenangebote-URL mit ?page=1-3 verwenden |
| push.sh schlägt fehl | Mac-Pfad in Sandbox nicht vorhanden | push.sh klon't nach /tmp – automatisch korrekt |
| Kontext geht verloren | Zu lange Session | Rohdaten ZUERST in rohdaten/KW[XX].md speichern |

---

## Push-Workflow (sandbox-kompatibel)

push.sh klon't automatisch nach /tmp und verwendet den richtigen Workspace-Pfad.
Falls es trotzdem fehlschlägt, manuell:

```bash
cd /tmp && rm -rf sp_push
git clone https://[TOKEN]@github.com/Sebo-ai/Supermarkt-Prospekte sp_push
# Token steht in push.sh – liegt lokal, wird nie in GitHub eingecheckt
cp [WORKSPACE]/index.html /tmp/sp_push/
cp [WORKSPACE]/angebote.md /tmp/sp_push/
cp -r [WORKSPACE]/archiv /tmp/sp_push/
cd /tmp/sp_push
git config user.email "sebastiangosgens@users.noreply.github.com"
git config user.name "Sebastian"
git add -A && git commit -m "[message]" && git push origin main
```
