# Voedingstracker

Persoonlijke voedingstracker in het Nederlands. Houdt kcal en macro's (eiwit, koolhydraten, vet) per dag bij, met een gewichtsgrafiek, doelen, productzoeker, barcodescanner en tips over wat nog past in de resterende macro's.

De hele app is **één bestand**: `voedingstracker.html` (HTML + CSS + vanilla JS, geen build-stap, geen framework). Zo is hij oorspronkelijk gemaakt en gepubliceerd als claude.ai Artifact:
https://claude.ai/artifact/E3crA6XWdYoLPynBfrL9Ci

## Eerst beslissen: Artifact of zelf hosten

Dit bepaalt wat er kan. Vraag de gebruiker welke route hij wil voordat je grote wijzigingen doet.

**Route A: blijven als claude.ai Artifact (huidige situatie)**
- Opslag via `window.claude` capabilities `db` + `user` (privé per gebruiker), en AI via `sample`. Deze bestaan alleen binnen claude.ai. Lokaal is `window.claude` undefined en valt de app terug op `localStorage`.
- De CSP van gepubliceerde pagina's blokkeert alle netwerkverzoeken naar andere sites. Scripts mogen alleen van `cdnjs.cloudflare.com`, `cdn.jsdelivr.net/npm/`, `cdn.tailwindcss.com`, `code.jquery.com`. Stylesheets alleen van `fonts.googleapis.com`. Geen `fetch` naar externe API's, dus **geen Open Food Facts**.
- Claude Code kan een Artifact niet zelf publiceren. Wijzigingen publiceert de gebruiker opnieuw via claude.ai (het bestand uploaden of in een chat plakken met het verzoek om te updaten).

**Route B: zelf hosten (bijv. statische hosting, Netlify/Vercel/GitHub Pages)**
- Volledige netwerktoegang: je kunt Open Food Facts gebruiken voor echte barcode- en zoekfunctie (zie "Roadmap").
- `db`, `user` en `sample` bestaan dan niet. Je moet zelf opslag kiezen (localStorage, of een backend/Supabase/Firebase) en eventueel AI via een eigen backend. Zet nooit een API-sleutel in de frontend.
- Camera (`getUserMedia`) vereist HTTPS, behalve op `localhost`.

## Bestanden

```
voedingstracker.html   De hele app (CSS in <style>, JS in één IIFE in <script>)
CLAUDE.md              Dit bestand
```

Geen package.json, geen tests, geen build. Lokaal draaien: open het bestand in een browser, of `npx serve .` / `python3 -m http.server` in de projectmap (nodig voor camera-tests op localhost).

## Architectuur van het JS-bestand

Alles zit in één IIFE met `"use strict"`. Globale variabelen binnen de IIFE:

- `S`: de volledige app-state (zie datamodel).
- `tab` (`today|weight|tips|goals`), `date` (geselecteerde dag), `range`, `selW`, `tipFocus`, `tipSeed`, `aiIdeas`, `aiBusy`, `calc`.
- `sh`: toestand van de invoer-sheet (bottom sheet) of `null` als hij dicht is.
- `sampleFn`, `sampleImages`, `dbCol`: capabilities, `null` als niet beschikbaar.

Rendering: `render()` bouwt de HTML voor het actieve tabblad als string en zet die in `#view` via `innerHTML`. Geen virtuele DOM. Na elke wijziging roep je `markDirty(...)` (opslaan) en `render()` aan. **Alle gebruikers- of AI-tekst die in HTML terechtkomt moet door `esc()`.**

Belangrijkste functies:

| Onderdeel | Functies |
|---|---|
| Tabbladen | `renderTabs`, `renderToday`, `renderWeight`, `renderTips`, `renderGoals` |
| Vandaag | `ringSVG`, `macroCard`, `streak`, `avg7`, `sparkline`, `heroMsg` |
| Invoer-sheet | `openSheet`, `renderSheet`, `buildResults`, `searchLocal`, `portionHTML`, `updatePortion`, `manualHTML`, `saveProduct` |
| Scanner | `loadScanLib`, `startScan`, `stopScan`, `scanFromFile`, `onCode` |
| AI | `aiSearch` (zoeken), `readLabel` (etiketfoto), `askIdeas` (tips) |
| Gewicht | `weightChart` (inline SVG), `weightPick`, `readoutFor` |
| Tips | `remaining`, `suggestions`, `tipList`, `doCalc`, `showCalc` |
| Opslag | `loadLocal`, `saveLocal`, `markDirty`, `flush`, `initStore` |

Events: twee `document`-brede `click` listeners. De eerste handelt alleen de sheet af (`if(!sh) return`), de tweede alleen de pagina (`if(sh) return`). Knoppen gebruiken `data-*` attributen (`data-act`, `data-tab`, `data-add`, `data-edit`, `data-pick`, enz.). Voeg nieuwe acties toe in de juiste listener.

## Datamodel

```js
S = {
  goals:    { kcal, p, c, f, kg },            // dagdoel + streefgewicht (kg=0: geen doel)
  days:     { 'YYYY-MM-DD': [ {id, name, kcal, p, c, f, meal} ] },
                                               // meal: ontbijt | lunch | diner | snack
  weights:  [ { d: 'YYYY-MM-DD', kg } ],       // één meting per datum
  products: [ { id, name, kcal, p, c, f, portion, unit, barcode } ],
                                               // "Mijn producten", waarden PER 100 G
  water:    { 'YYYY-MM-DD': aantalGlazen },
  profile:  { sex, age, h, w, act, goal }      // invoer van de doelcalculator
}
```

Voeding in `days` is al omgerekend naar de gegeten hoeveelheid (totalen). De entry onthoudt dus niet de gram of de waarden per 100 g; de hoeveelheid staat alleen in de naam (bijv. "Havermout · 40 g").

De ingebouwde database `FOODS` (±120 items, per 100 g, met `portion` en `unit`) staat bovenaan het script en wordt omgezet naar `DB`. Waarden zijn gemiddelden uit eigen kennis, niet uit een officiële bron.

## Opslag

- **Altijd:** `localStorage`, sleutel `voedingstracker-v1`, de hele `S` als JSON.
- **Als `window.claude` beschikbaar is** (alleen in claude.ai): `db` + `user`. Pad `data/users/<uid>/` is privé per gebruiker. Daaronder:
  - doc `meta` met `{json}`: goals, weights, products, water, profile
  - doc `m-YYYY-MM` met `{json}`: alle dagen van die maand
  
  Opgesplitst per maand omdat een document maximaal 256 KiB mag zijn. Schrijven gebeurt gedebounced en serieel in `flush()`. Bij de eerste keer (lege db) wordt lokale data overgezet.
- Een Artifact-publicatie moet `capabilities: {db:{}, user:{}, sample:{}}` declareren, anders zijn ze niet beschikbaar.

## Ontwerp

- Mobile-first, max. breedte 520 px. Eén lettertype: Bricolage Grotesque (Google Fonts) met system-ui als fallback.
- Kleuren via CSS-variabelen op `:root`, met licht/donker via `prefers-color-scheme` en `data-theme`. Macrokleuren: eiwit `--p` (blauw), koolhydraten `--c` (amber), vet `--f` (roze). Accent `--accent` (groen), hero-gradient `--hero1..3`.
- Gebruik altijd de variabelen, geen vaste hex-kleuren in componenten (dark mode).
- `viewport-fit=cover` en `env(safe-area-inset-*)` zijn nodig voor telefoons; laat die staan.
- Teksten: Nederlands, zin-hoofdletters, actieve werkwoorden ("Opslaan", "Toevoegen"). Getallen via `nf()` (nl-NL formattering) en klasse `num` (tabular-nums).

## Wat nog niet getest is

De app is gebouwd zonder te kunnen draaien in een browser. Test dit eerst:

1. **Camera/barcodescan.** Gebruikt `html5-qrcode@2.3.8` van cdnjs (URL in `loadScanLib`, niet geverifieerd). Controleer of het script laadt, of de camera start en of EAN-13 wordt herkend. Alternatief: eigen scanner met `BarcodeDetector` (niet in alle browsers, o.a. niet in Safari).
2. **`sample` (AI) in claude.ai.** Zoeken, etiketfoto en ideeën. Controleer de JSON-parsing en foutafhandeling.
3. **`db`-opslag.** Laden, wegschrijven, en de overgang van localStorage naar db.
4. **Donkere modus** en kleine schermen (320 px).
5. Randgevallen: dag zonder entries, kcal over doel, één gewichtsmeting, 365+ dagen data.

## Bekende beperkingen

- Entries onthouden geen gram of waarden per 100 g, dus een entry bewerken kan alleen op totalen.
- "Mijn producten" kun je nu alleen toevoegen of overschrijven, niet bewerken of verwijderen.
- "Eerder gegeten" opent het handmatige formulier, niet de hoeveelheidsstap.
- Barcode wordt alleen gevonden in eigen producten (geen publieke database in route A).
- Waarden uit AI-zoeken en etiketlezen zijn schattingen; de UI markeert dit ("Schatting").
- Geen export/import, geen weekoverzicht, geen meerdere profielen.
- Alleen Nederlands.

## Roadmap (voorstellen)

1. Producten beheren (bewerken/verwijderen) en entries opslaan met gram + per-100 g-waarden.
2. Weekoverzicht: gemiddelde kcal en macro's, grafiek per dag.
3. Export/import als CSV/JSON (in een Artifact via de `downloads` capability).
4. Favorieten en maaltijden combineren (bijv. "mijn standaard ontbijt").
5. Route B: Open Food Facts (`https://world.openfoodfacts.org/api/v2/product/<barcode>.json` en zoek-API) voor echte barcode- en zoekresultaten, met cache in localStorage.
6. Vitaminen/vezels/suiker indien de gebruiker dat wil.

## Werkafspraken

- Houd het bij één bestand tenzij de gebruiker kiest voor een build-setup (bijv. Vite). Dan eerst overleggen.
- Voeg geen externe scripts toe buiten de toegestane hosts als de app Artifact moet blijven.
- Gebruik geen `localStorage` voor iets dat tussen apparaten moet synchroniseren; dat hoort in `db`.
- Controleer na elke wijziging de JS-syntax en test handmatig in de browser: dagweergave, sheet openen/sluiten, gewicht toevoegen, doel opslaan.
- Vraag de gebruiker bij onduidelijkheid; hij is geen ontwikkelaar, leg keuzes in gewone taal uit.
