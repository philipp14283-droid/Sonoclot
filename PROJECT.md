# Sonoclot Interpreter

Web-App zur **Entscheidungsunterstützung** bei der Auswertung von Sonoclot-Gerinnungsanalysen
in Herzchirurgie und Intensivmedizin. Ein Foto des Signature-Viewer-Bildschirms wird ausgelesen,
die Messwerte extrahiert und anschließend zu einer phasenabhängigen, strukturierten Einordnung
verarbeitet — mit dem ausdrücklichen Anspruch, **keine Diagnose und keine Therapieentscheidung**
zu treffen.

> **Kein Medizinprodukt.** Die App liefert Entscheidungsunterstützung auf Basis anonymisierter
> Gerätebilder. Ärztliche Beurteilung und klinischer Kontext sind immer maßgeblich.

---

## Was das Programm tut

Ein Sonoclot Signature-Viewer zeigt über den Verlauf eines herzchirurgischen Eingriffs mehrere
Messungen (Baseline, unter Heparin an der Herz-Lungen-Maschine, nach Protamin). Jede Messung
liefert je nach Testtyp Parameter wie **ACT** (Activated Clotting Time), **ClotRate** und
**PlateletFunction**, jeweils mit einem gerätespezifischen Referenzbereich.

Der Ablauf für die anwendende Ärztin / den Arzt:

1. **Foto aufnehmen** vom Viewer-Bildschirm (ein Bild darf den ganzen Verlauf zeigen).
2. **Werte auslesen** — das Modell extrahiert alle Tabellenzeilen als strukturiertes JSON.
   Die Werte lassen sich vor der Interpretation manuell korrigieren.
3. **Kontext ergänzen** (optional) — Heparin-/Protamin-Dosis, Blutungsstatus, Patientendaten,
   Hausschwellen.
4. **Interpretieren** — das Modell ordnet die Messungen selbst den Phasen zu (Ausgang → HLM →
   post Protamin), erkennt Muster (Restheparin, Fibrinogenmangel, Faktorenmangel,
   Thrombozytenfunktion, Hyperfibrinolyse) und formuliert priorisierte Erwägungen.

Die Ausgabe beginnt bewusst mit einer **Kurzfassung**, die im OP unter Zeitdruck in wenigen
Sekunden gelesen werden kann.

---

## Architektur

Die App ist eine **einzelne, eigenständige HTML-Datei** ([`index.html`](index.html), ~700 Zeilen)
ohne Build-Schritt, ohne externe Laufzeit-Abhängigkeiten. HTML, CSS und JavaScript sind inline.

```
┌─────────────────────┐      HTTPS POST       ┌──────────────────────┐      ┌─────────────┐
│  index.html         │ ────────────────────▶ │  Cloudflare Worker   │ ───▶ │ Anthropic   │
│  (Browser, Handy)   │   {model, system,     │  (hält den API-Key   │      │ Messages    │
│                     │ ◀──────────────────── │   als Secret)        │ ◀─── │ API         │
└─────────────────────┘   content-Blöcke      └──────────────────────┘      └─────────────┘
```

- **Frontend** (`index.html`): Bildaufnahme, Herunterskalieren, zwei aufeinanderfolgende
  Modell-Aufrufe, Rendering von Wertetabelle und Interpretation.
- **Backend** (Cloudflare Worker, *nicht Teil dieses Repos*): nimmt die Anfrage entgegen und
  reicht sie mit dem serverseitig hinterlegten Anthropic-API-Key weiter. Der API-Key liegt damit
  **auf Cloudflare als Secret**, nie im Browser. Der Client kennt nur die Worker-URL und ein
  optionales App-Passwort.

### Warum ein Worker dazwischen?

Der Anthropic-API-Key darf nicht ins clientseitige JavaScript — jeder Websitebesucher könnte ihn
sonst auslesen. Der Worker ist ein dünner, authentifizierter Proxy: der Browser schickt
`{model, max_tokens, system, messages}`, der Worker ergänzt den Key und leitet an die Messages API
weiter. Ein optionales `x-app-token` (App-Passwort) schützt den Worker vor fremder Nutzung.

---

## Die zwei-stufige Modell-Pipeline

Der Kern der App sind zwei getrennte Prompts, als `<script type="text/plain">` im HTML abgelegt
(damit Backticks und Codefences im Prompt nichts zerbrechen).

### Prompt 1 — Extraktion (`#prompt1`)

**Aufgabe:** reines Ablesen, **keine** Interpretation. Gibt ausschließlich gültiges JSON zurück.

- Liest die Ergebnistabelle Zeile für Zeile und extrahiert **alle** Messungen eines Fotos.
- Pro Parameter: Wert, Referenzbereich (`range_min`/`range_max`) **immer aus der Bildzeile**,
  nie aus dem Gedächtnis ergänzt.
- Berechnet `ausserhalb_norm` selbst (`hoch`/`niedrig`/`nein`) und nutzt die **Farbe** (rot =
  außerhalb) als Gegenkontrolle. Widerspruch → `konflikt: true` + `konfidenz: niedrig`.
- Normalisiert Testtypen (`kACT` | `gbACT+` | `HgbACT+` | `unbekannt`).
- Liest die Kurvengrafik grob mit: Startsignal im gültigen Bereich (10–25) als Verwertbarkeits-
  kriterium, Abfall nach Peak als möglicher Fibrinolyse-Hinweis.
- Nicht sicher Lesbares wird `null` statt geraten; abgeschnittene Zeilen bleiben mit `null`-Feldern
  erhalten und werden in `warnungen` vermerkt.

Ausgabeschema (gekürzt):

```json
{
  "messungen": [
    {
      "zeit": "13:51:13", "test": "kACT", "application": "STZ",
      "ACT":              { "wert": 273, "range_min": 80, "range_max": 116, "ausserhalb_norm": "hoch", "in_rot": true, "konfidenz": "hoch", "konflikt": false },
      "ClotRate":         { "wert": 9.4, "range_min": 9, "range_max": 56, "ausserhalb_norm": "nein", "in_rot": false, "konfidenz": "hoch", "konflikt": false },
      "PlateletFunction": null,
      "kurve":            { "startsignal_im_gueltigen_bereich_10_25": true, "abfall_nach_peak": false, "anmerkung": "" }
    }
  ],
  "lesbarkeit_gesamt": "hoch",
  "warnungen": []
}
```

### Prompt 2 — Interpretation (`#prompt2`)

**Rolle:** Entscheidungs-*Unterstützer* für erfahrene Ärztinnen und Ärzte. Stellt keine Diagnose,
trifft keine Entscheidung. Arbeitet ausschließlich mit anonymisierten Messungen.

- **Phasenzuordnung selbst:** aus Testzusammensetzung + Zeitstempeln (+ optionalem Kontext)
  werden Baseline, `an_hlm` und `post_protamin` abgeleitet. Frühere Messungen im selben Bild
  dienen automatisch als Baseline für Delta-/Trendaussagen (immer nur innerhalb desselben Testtyps).
- **Post-Protamin-Fragen in fester Reihenfolge:** Restheparin → Protamin-Überdosierung →
  Fibrinogenmangel → Faktorenmangel → Thrombozytenfunktion → FXIII (nur klinisch) →
  Hyperfibrinolyse.
- **Substitutions-Trigger:** eine Substitution wird nur erwogen, wenn ein relevanter Wert unter
  der Range liegt **und** eine Blutung angegeben ist. Ohne Blutungsangabe wird bewusst
  zurückhaltender formuliert.
- **Sicherheitsregeln übersteuern alles:** niedrige Konfidenz / `null` / Konflikt → keine
  Empfehlung darauf stützen; an der HLM nie Substitution empfehlen; FXIII nie als gemessen
  darstellen; Patientendaten nie für konkrete Dosisberechnung nutzen.

**Ausgabeformat** (Markdown): die **Kurzfassung steht zuerst** (10-Sekunden-Lesbarkeit im OP),
danach Verlauf & Kontext, Werte, Befund/Muster, Erwägungen, „Was fehlt/unsicher" und ein wörtlich
festgelegter Abschlusssatz.

---

## Datenfluss im Frontend

1. **Bildwahl** (`#pick` → `#file`): Foto oder Bilddatei. `downscale()` skaliert auf max.
   **2000 px** Kantenlänge herunter und kodiert als JPEG (Qualität 0.9, Base64) — spart Tokens
   und Upload.
2. **Extraktion** (`#extractBtn`): `callClaude(PROMPT1, [Bild, Text], 8000)`. Die Antwort wird mit
   `stripJson()` bereinigt; schlägt `JSON.parse` fehl, versucht `repairJson()` abgeschnittenes JSON
   zu schließen (offene Strings/Klammern). Ist es nicht reparierbar, werden die Rohdaten im
   JSON-Feld zur manuellen Korrektur angeboten — **nichts wird verworfen**.
3. **Wertetabelle** (`renderVals()`): eine Tabelle je Messung mit Richtungspfeilen (↑/↓/→) und
   Badges für „rot", „Konflikt", „unsicher" sowie Kurven-Hinweisen.
4. **Interpretation** (`#interpretBtn`): baut den Kontextblock (Kontextfelder + Hausschwellen),
   ersetzt die Platzhalter `ACT_ZIEL_HLM_SEK` / `CR_HEPARIN_SCHWELLE` im System-Prompt und ruft
   `callClaude(system, [Text], 4500)`.
5. **Rendering** (`renderMd()`): eigener, schlanker Markdown-Renderer — Überschriften, geordnete
   (`<ol>`, Priorisierung bleibt erhalten) und ungeordnete Listen, Tabellen; der Abschlusssatz
   wird als hervorgehobener Block gerendert.

---

## Konfiguration

Im Frontend, oben unter „⚙︎ Backend (Cloudflare)":

| Einstellung        | Speicherung           | Zweck                                             |
| ------------------ | --------------------- | ------------------------------------------------- |
| Worker-URL         | `localStorage`        | Adresse des Cloudflare-Proxys                     |
| App-Passwort       | `localStorage`        | optionales `x-app-token` gegen fremde Nutzung     |

`localStorage`-Zugriffe sind defensiv gekapselt (`lsGet`/`lsSet`/`lsDel`), da Speicher in
`data:`-URLs / manchen Sandboxes deaktiviert ist.

Im Code ([`index.html`](index.html), IIFE unten):

- `MODEL` — Standardmodell `claude-sonnet-5`. Umstellbar (z. B. `claude-opus-4-8` für maximale
  Genauigkeit). Die Anzeige im Backend-Panel folgt automatisch der Konstante.
- `DEFAULT_ENDPOINT` — vorbelegte Worker-URL.
- Hausschwellen im Kontext: ACT-Ziel HLM (Standard 400 s), CR-Heparin-Schwelle (Standard 6).

---

## Bedienoberfläche

- **Handy-optimiert:** `viewport-fit=cover`, Safe-Area-Insets, ein-spaltiges Layout bis 640 px.
- **Dunkelmodus** über `prefers-color-scheme` — für den nächtlichen OP-Einsatz weniger blendend;
  Warnfarben bleiben kräftig genug.
- **Barrierefreiheit:** Status-/Fehlermeldungen mit `role="alert"` / `aria-live`.
- **Kurzfassung hervorgehoben:** die erste Ergebnissektion ist optisch abgesetzt.
- **„Beispiel"-Button:** lädt einen vollständigen Beispielfall (5 Messungen über den Verlauf) —
  zum Testen ohne echtes Foto.
- **„Text kopieren" / „Neuer Fall":** Rohtext für die Dokumentation kopieren; „Neuer Fall" leert
  den Fall, behält aber Worker-URL und Hausschwellen.

---

## Sicherheit & Datenschutz

- **API-Key nie im Browser** — liegt als Secret auf dem Cloudflare Worker.
- **Nur anonymisierte Gerätebilder** verarbeiten; keine patientenidentifizierenden Daten.
- Patientendaten (optional) werden nur qualitativ zur Größenordnung des Heparin-/Protamin-
  Verhältnisses genutzt, **nie** für konkrete Dosisberechnung.
- Der Interpretations-Prompt formuliert bei fehlendem Kontext, Konflikten oder niedriger Konfidenz
  bewusst zurückhaltend und verweist auf die ärztliche Beurteilung.

---

## Entwicklung & Test

Keine Build-Kette. Zum Ausprobieren die Datei lokal im Browser öffnen:

```
open index.html            # macOS
```

- **Ohne Backend testbar:** „Beispiel"-Button lädt einen Fall, die Wertetabelle und der
  Markdown-Renderer lassen sich damit vollständig prüfen.
- **Echter Lauf** braucht eine erreichbare Worker-URL (oben eintragen) und ein reales oder
  Beispiel-Foto.
- Der Interpretationspfad wurde bei der Überarbeitung mit einem lokal gestubbten `fetch`
  verifiziert (Renderer, `<ol>`-Priorisierung, Kurzfassung-Zusammenhalt über Leerzeilen,
  Reset-Verhalten, Dark-Mode-Kontraste). Ein **End-to-End-Test mit echtem Gerätefoto** steht als
  ärztliche Abnahme weiterhin aus.

---

## Deployment

- **Frontend:** statische Datei — beliebiges Hosting (GitHub Pages, Cloudflare Pages, jeder
  Webserver) oder lokal geöffnet.
- **Backend:** ein Cloudflare Worker mit dem Anthropic-API-Key als Secret und optionalem
  `APP_TOKEN`. Die Worker-URL wird im Frontend eingetragen. (Der Worker-Code ist nicht Teil dieses
  Repos.)

---

## Dateien

| Datei         | Zweck                                                       |
| ------------- | ----------------------------------------------------------- |
| `index.html`  | Die gesamte App: HTML, CSS, JS, beide Prompts, Beispielfall |
| `PROJECT.md`  | Dieses Dokument                                             |
