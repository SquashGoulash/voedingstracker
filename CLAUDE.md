# Voedingstracker

Persoonlijke voedingstracker in het Nederlands. Houdt kcal en macro's (eiwit, koolhydraten, vet) per dag bij, met een weekoverzicht, vaste maaltijden, een gewichtsgrafiek, doelen, productzoeker, barcodescanner en tips over wat nog past in de resterende macro's.

De hele app is **één bestand**: `voedingstracker.html` (HTML + CSS + vanilla JS, geen build-stap, geen framework). Zo is hij oorspronkelijk gemaakt en gepubliceerd als claude.ai Artifact:
https://claude.ai/artifact/E3crA6XWdYoLPynBfrL9Ci

## Eerst beslissen: Artifact of zelf hosten

Dit bepaalt wat er kan. **Gekozen op 2026-10-01: route B (zelf hosten).** Route A staat hieronder ter referentie, omdat de code nog `window.claude`-ondersteuning bevat.

**Route A: blijven als claude.ai Artifact (huidige situatie)**
- Opslag via `window.claude` capabilities `db` + `user` (privé per gebruiker), en AI via `sample`. Deze bestaan alleen binnen claude.ai. Lokaal is `window.claude` undefined en valt de app terug op `localStorage`.
- De CSP van gepubliceerde pagina's blokkeert alle netwerkverzoeken naar andere sites. Scripts mogen alleen van `cdnjs.cloudflare.com`, `cdn.jsdelivr.net/npm/`, `cdn.tailwindcss.com`, `code.jquery.com`. Stylesheets alleen van `fonts.googleapis.com`. Geen `fetch` naar externe API's, dus **geen Open Food Facts**.
- Claude Code kan een Artifact niet zelf publiceren. Wijzigingen publiceert de gebruiker opnieuw via claude.ai (het bestand uploaden of in een chat plakken met het verzoek om te updaten).

**Route B: zelf hosten (bijv. statische hosting, Netlify/Vercel/GitHub Pages)**
- Volledige netwerktoegang: je kunt Open Food Facts gebruiken voor echte barcode- en zoekfunctie (zie "Roadmap").
- `db`, `user` en `sample` bestaan dan niet. Je moet zelf opslag kiezen (localStorage, of een backend/Supabase/Firebase) en eventueel AI via een eigen backend. Zet nooit een API-sleutel in de frontend.
- Camera (`getUserMedia`) vereist HTTPS, behalve op `localhost`.

## Online

Gehost op GitHub Pages: https://squashgoulash.github.io/voedingstracker/ (repo `SquashGoulash/voedingstracker`, publiek, branch `main`, map `/`). Elke `git push` naar `main` zet de nieuwe versie binnen ±1 minuut online. Commits gebruiken het noreply-adres van GitHub (lokaal ingesteld in `git config user.email`); zet nooit een privé e-mailadres in commits of bestanden. Gegevens staan per apparaat in localStorage, dus laptop en telefoon synchroniseren niet.

## Bestanden

```
voedingstracker.html   De hele app (CSS in <style>, JS in één IIFE in <script>)
index.html             Alleen een doorverwijzing naar voedingstracker.html (voor GitHub Pages)
manifest.webmanifest   Maakt de app installeerbaar (beginscherm, zonder browserbalk); start_url = voedingstracker.html
icon-192.png, icon-512.png, icon-maskable-512.png   App-iconen (ring in macrokleuren op #18181B)
CLAUDE.md              Dit bestand
```

Geen package.json, geen tests, geen build. Lokaal draaien: open het bestand in een browser, of `npx serve .` / `python3 -m http.server` in de projectmap (nodig voor camera-tests op localhost).

## Architectuur van het JS-bestand

Alles zit in één IIFE met `"use strict"`. Globale variabelen binnen de IIFE:

- `S`: de volledige app-state (zie datamodel).
- `tab` (`today|week|weight|tips|goals|settings`), `date` (geselecteerde dag), `week` (maandag van de getoonde week), `selDay` (aangetikte dag in de weekgrafiek), `range`, `selW`, `tipFocus`, `tipSeed`, `aiIdeas`, `aiBusy`, `calc`.
- `sh`: toestand van de invoer-sheet (bottom sheet) of `null` als hij dicht is.
- `sampleFn`, `sampleImages`, `dbCol`: capabilities, `null` als niet beschikbaar.

Rendering: `render()` bouwt de HTML voor het actieve tabblad als string en zet die in `#view` via `innerHTML`. Geen virtuele DOM. Na elke wijziging roep je `markDirty(...)` (opslaan) en `render()` aan. **Alle gebruikers- of AI-tekst die in HTML terechtkomt moet door `esc()`.**

Belangrijkste functies:

| Onderdeel | Functies |
|---|---|
| Tabbladen | `renderTabs`, `renderToday`, `renderWeight`, `renderTips`, `renderGoals`, `renderSettings` |
| Vandaag | `renderToday` (datumbalk, samenvattingskaart met `ringSVG` + `macroCard`/`thinBar`, inklapbare maaltijden), `mealOpen` (welke maaltijden open zijn; `addEntry` klapt de maaltijd open, `data-toggle` klapt in/uit), `copyFromPrev('all'|maaltijd)` (kopieert van de dag vóór `date`; links `data-copy` alleen bij een lege dag/maaltijd) |
| Week | `renderWeek`, `weekChart` (inline SVG, staven + doellijn, tik = `selDay`), `weekDays`, `weekAvg` (gelogde dagen vóór vandaag; vandaag alleen als er verder niets is), `mondayOf`, `isoWeek` |
| Vaste maaltijden | `mealItem` (entry → onderdeel zonder id/meal), `mealHTML` (sheet-modus `meal`, `sh.draft`), `checkedItems`, `mealsCard` (Doelen). Toevoegen: `data-usemeal` in de zoek-sheet |
| Invoer-sheet | `openSheet(meal, entry, prod)`, `renderSheet`, `buildResults`, `searchLocal`, `portionHTML` (ook voor bewerken), `updatePortion`, `manualHTML`, `productHTML`, `saveProduct` |
| Entries | `addEntry`, `replaceEntry`, `entryFrom(item, g, meal)` (maakt een entry mét `g` en `base`; gebruik deze voor elke entry uit een product) |
| Scanner | `loadScanLib`, `startScan`, `stopScan`, `scanFromFile`, `onCode` |
| Open Food Facts | Barcode: `offLookup`, `offCache`, `offStore`. Op naam: `offSearch` (knop of Enter in zoekveld), `offSearchFetch` (`cgi/search.pl`, filter Nederland, sortering op populariteit), `offQCache`, `offQStore`. Gedeeld: `offFetch` (max. 3 pogingen, ook bij netwerkfout: een 503 van OFF heeft geen CORS-header en komt in de browser binnen als TypeError, niet als status 503), `offErrMsg`, `offToItem` (OFF-product → item met `src:'off'`) |
| AI | `aiSearch` (zoeken), `readLabel` (etiketfoto), `askIdeas` (tips) |
| Gewicht | `weightChart` (inline SVG), `weightPick`, `readoutFor` |
| Tips | `remaining`, `suggestions`, `tipList`, `doCalc`, `showCalc` |
| Doelen | `renderGoals`, `productsCard` (Mijn producten: tik = bewerken in de sheet, modus `product`) |
| Instellingen | `renderSettings` (Weergave, Bijhouden, Opslag). Vinkjes `data-track`, doelen `data-tgoal` (opslaan bij `change`) |
| Extra stoffen | `EXTRA` (sleutel, naam, max/min, standaarddoel, eenheid), `XK`, `tracked()`, `has`, `copyX` (gebruik deze om extra waarden mee te kopiëren), `rx` (afronden; cafeïne op hele mg), `extraBlock` (Vandaag/Week), `extraLine` (hoeveelheidsstap), `xFields`/`readX` (formulieren) |
| Opslag | `loadLocal`, `saveLocal`, `markDirty`, `flush`, `initStore` |

Events: twee `document`-brede `click` listeners. De eerste handelt alleen de sheet af (`if(!sh) return`), de tweede alleen de pagina (`if(sh) return`). Knoppen gebruiken `data-*` attributen (`data-act`, `data-tab`, `data-add`, `data-edit`, `data-pick`, enz.). Voeg nieuwe acties toe in de juiste listener.

## Datamodel

```js
S = {
  goals:    { kcal, p, c, f, kg, sug, fib, sat, salt, caf },
                                               // dagdoel + streefgewicht (kg=0: geen doel) + doelen extra stoffen
  days:     { 'YYYY-MM-DD': [ {id, name, kcal, p, c, f, meal, g?, base?} ] },
                                               // meal: ontbijt | lunch | diner | snack
                                               // g + base {name, kcal, p, c, f, portion, unit, src}: alleen bij entries uit een product
  weights:  [ { d: 'YYYY-MM-DD', kg } ],       // één meting per datum
  products: [ { id, name, kcal, p, c, f, portion, unit, barcode } ],
                                               // "Mijn producten", waarden PER 100 G
  meals:    [ { id, name, items: [ {name, kcal, p, c, f, g?, base?} ] } ],
                                               // vaste maaltijden; items als entries zonder id/meal
  water:    { 'YYYY-MM-DD': aantalGlazen },   // geen UI meer (verwijderd 2026-10-02); data blijft bewaard
  profile:  { sex, age, h, w, act, goal },    // invoer van de doelcalculator
  track:    { sug?, fib?, sat?, salt?, caf? }  // true = tonen (Instellingen → Bijhouden)
}
```

Voeding in `days` is al omgerekend naar de gegeten hoeveelheid (totalen: `kcal`, `p`, `c`, `f`). Entries uit een product (zoeken, scannen, Eerder gegeten, Tips) bewaren daarnaast `g` en een kopie van de waarden per 100 g in `base`. Bewerken gaat dan via de hoeveelheidsstap. Handmatige entries, entries van vóór 2026-10-02 en entries die via "Waarden zelf aanpassen" zijn gewijzigd hebben geen `g`/`base` en worden op totalen bewerkt. `base` is een kopie, dus een product bewerken of verwijderen verandert eerdere entries niet.

**Extra stoffen** (sinds 2026-10-02): suiker `sug`, vezels `fib`, verzadigd vet `sat`, zout `salt` (gram) en cafeïne `caf` (mg). Optionele velden op items, `base`, entries, producten en maaltijd-onderdelen, op dezelfde basis als kcal (per 100 g bij producten/`base`, totaal bij entries). **Ontbreekt = onbekend, niet 0**; Vandaag meldt dan dat het totaal hoger kan zijn. Altijd opgeslagen, ook als ze niet aangevinkt zijn; aanvinken bepaalt alleen wat je ziet en welke invoervelden er zijn. Open Food Facts: `sugars_100g`, `fiber_100g`, `saturated-fat_100g`, `salt_100g` (of `sodium_100g` × 2,5), `caffeine_100g` (gram, × 1000). AI-zoeken en etiketlezen vragen ze (nog) niet op.

De ingebouwde database `FOODS` (±125 items, per 100 g: naam, kcal, p, c, f, portion, unit, suiker, vezels, verzadigd vet, zout, cafeïne mg; cafeïne mag ontbreken = 0) staat bovenaan het script en wordt omgezet naar `DB`. Waarden zijn gemiddelden uit eigen kennis, niet uit een officiële bron.

## Opslag

- **Altijd:** `localStorage`, sleutel `voedingstracker-v1`, de hele `S` als JSON.
- **Weergave (per apparaat):** `localStorage`, sleutel `voedingstracker-ui`, object `UI` = `{theme: 'auto'|'light'|'dark', open: {maaltijd: true}}`. `applyTheme()` zet `data-theme` en `<meta name="theme-color">`. Keuze onder Instellingen → Weergave. Hoort niet in `S`/`db`.
- **Open Food Facts-cache:** `localStorage`, sleutel `voedingstracker-off2` ("2" sinds de extra stoffen; de oude sleutels worden bij het starten gewist), `{barcode: {t, item}}`, 30 dagen geldig, max. 300 items. Alleen gevonden producten worden bewaard.
- **Open Food Facts-zoekcache:** `localStorage`, sleutel `voedingstracker-offq2`, `{zoekterm: {t, items}}`, 7 dagen geldig, max. 50 zoektermen. Ook lege resultaten worden bewaard. De nieuwere zoek-API (search.openfoodfacts.org) stuurt geen CORS-header en werkt dus niet vanuit de browser.
- **Als `window.claude` beschikbaar is** (alleen in claude.ai): `db` + `user`. Pad `data/users/<uid>/` is privé per gebruiker. Daaronder:
  - doc `meta` met `{json}`: goals, weights, products, meals, water, profile
  - doc `m-YYYY-MM` met `{json}`: alle dagen van die maand
  
  Opgesplitst per maand omdat een document maximaal 256 KiB mag zijn. Schrijven gebeurt gedebounced en serieel in `flush()`. Bij de eerste keer (lege db) wordt lokale data overgezet.
- Een Artifact-publicatie moet `capabilities: {db:{}, user:{}, sample:{}}` declareren, anders zijn ze niet beschikbaar.

## Ontwerp

- Mobile-first, max. breedte 520 px. Eén lettertype: Bricolage Grotesque (Google Fonts) met system-ui als fallback.
- Kleuren via CSS-variabelen op `:root`, met licht/donker via `prefers-color-scheme` en `data-theme`. Macrokleuren: eiwit `--p` (blauw), koolhydraten `--c` (amber), vet `--f` (roze). Neutraal: licht = wit/grijs (`--bg #F4F4F5`), donker = antraciet (`--bg #0F0F11`); `--accent` is zwart (licht) of wit (donker), geen groen. Kaarten hebben een rand + subtiele `--shadow`.
- Minimalistisch (gekozen 2026-10-02, mix van ontwerpen "Rustig" en "Compact"): geen emoji's, tegels, gekleurde balken of groene koppen; paginakoppen zijn platte tekst. Vandaag = samenvattingskaart + maaltijdkaarten die standaard ingeklapt zijn. Voeg geen nieuwe tegels of decoratie toe zonder overleg.
- Gebruik altijd de variabelen, geen vaste hex-kleuren in componenten (dark mode).
- `viewport-fit=cover` en `env(safe-area-inset-*)` zijn nodig voor telefoons; laat die staan.
- Teksten: Nederlands, zin-hoofdletters, actieve werkwoorden ("Opslaan", "Toevoegen"). Getallen via `nf()` (nl-NL formattering) en klasse `num` (tabular-nums).

## Wat nog niet getest is

Door de gebruiker getest en werkend (2026-10-02): barcodescanner (camera + `html5-qrcode`) en lichte/donkere modus.

Nog niet getest:

1. Kleine schermen (320 px).
2. Randgevallen: dag zonder entries, kcal over doel, één gewichtsmeting, 365+ dagen data.
3. Alleen relevant voor route A (claude.ai): `sample` (AI zoeken, etiketfoto, ideeën) en `db`-opslag inclusief de overgang van localStorage naar db.

## Bekende beperkingen

- Oude en handmatige entries hebben geen `g`/`base` en zijn alleen op totalen te bewerken; "Eerder gegeten" opent bij die entries het handmatige formulier.
- De maaltijd van een entry kun je niet wijzigen (alleen verwijderen en opnieuw toevoegen).
- Barcode: eerst eigen producten, dan Open Food Facts. Zoeken op naam in Open Food Facts gaat via een knop (limiet ±10 zoekopdrachten per minuut; de server geeft vaak 503).
- Waarden uit AI-zoeken en etiketlezen zijn schattingen; de UI markeert dit ("Schatting").
- Een vaste maaltijd voeg je in zijn geheel toe; losse onderdelen pas je daarna per entry aan.
- Geen export/import, geen meerdere profielen.
- Alleen Nederlands.

## Roadmap (voorstellen)

1. ~~Producten beheren en entries met gram~~ **Klaar** (Doelen → Mijn producten; entries met `g` + `base`).
2. ~~Weekoverzicht~~ **Klaar** (tabblad Week).
3. Export/import als CSV/JSON (in een Artifact via de `downloads` capability).
4. ~~Vaste maaltijden~~ **Klaar** (bewaren onder een maaltijd op Vandaag, beheren onder Doelen).
5. ~~Route B: Open Food Facts~~ **Klaar:** barcode én zoeken op naam.
6. ~~Vezels/suiker~~ **Klaar** (Instellingen → Bijhouden: suiker, vezels, verzadigd vet, zout, cafeïne). Vitaminen bewust niet: staan zelden op verpakkingen of in Open Food Facts (keuze gebruiker 2026-10-02).

## Werkafspraken

- Houd het bij één bestand tenzij de gebruiker kiest voor een build-setup (bijv. Vite). Dan eerst overleggen.
- Voeg geen externe scripts toe buiten de toegestane hosts als de app Artifact moet blijven.
- Gebruik geen `localStorage` voor iets dat tussen apparaten moet synchroniseren; dat hoort in `db`.
- Controleer na elke wijziging de JS-syntax en test handmatig in de browser: dagweergave, sheet openen/sluiten, gewicht toevoegen, doel opslaan.
- Vraag de gebruiker bij onduidelijkheid; hij is geen ontwikkelaar, leg keuzes in gewone taal uit.
