# Omgevingswet Tool

Webtool voor vastgoedtaxaties: je zoekt een adres en ziet welke bestemmingsplannen en omgevingsplannen daar gelden. De tool haalt de bestemming(en) van het perceel op en maakt een kant-en-klare samenvattingstekst voor een taxatierapport. Daarnaast kan hij de volledige regels downloaden en een PDF maken met een kaartje.

De gebruiker werkt in het Nederlands. Antwoord in het Nederlands en schrijf UI-teksten, functienamen en commentaar ook in het Nederlands.

## Opbouw

- **Alles staat in één bestand: `index.html`.** CSS, HTML en JavaScript staan inline. Er is geen build-stap, geen framework en geen npm. Houd dat zo: geen bundler, TypeScript of React toevoegen.
- Externe libraries komen via een CDN met vaste versie:
  - Leaflet 1.9.4 en leaflet-image 0.4.0 (unpkg)
  - pdf.js 3.11.174 (cdnjs, UMD-build, `window.pdfjsLib`); deze wordt pas geladen als een analoog plan erom vraagt
- Openen kan direct (`file://`) of via een lokale proxy (`http://localhost`). Bij http(s) zet `VIA_PROXY` de DSO-calls om naar `/api/...` (zie `proxyfy()`), zodat de proxy ze server-side ophaalt. Die proxy staat niet in dit bestand.
- API-sleutels staan **nooit** in het bestand (de repo is openbaar). Bewust zo gelaten, ook voor collega's. Ontbreekt de DSO- of RP-key, dan opent Instellingen (tandwiel) automatisch met een korte handleiding om ze aan te vragen (`#sleutelUitleg`, `werkSleutelUitlegBij()`). `api-keys.txt` (lokaal, met Ramons eigen sleutels) staat in `.gitignore`. Sleutels worden in `localStorage` bewaard:
  - DSO-key: `dso_key`
  - Ruimtelijke-plannen-key: `dso_rpkey`
  - CARTO-key: `carto_key` (optioneel; er is een standaardkey)
  - onthouden aan/uit: `dso_remember`

  Elke toegang tot `localStorage` loopt via `lsGet`/`lsSet`/`lsDel` met try/catch.

## Externe bronnen

| Wat | Endpoint (constante) | Key |
|---|---|---|
| Plannen op een punt | DSO Ontsluiten v2 `/documenten/_zoek` (`ONTSLUITEN`) | DSO-key |
| Bestemmingsvlakken, regelteksten | Ruimtelijke Plannen API v4 (`RP`) | RP-key |
| Omgevingsplan: functies en regels | DSO Presenteren v8 (`PRESENTEREN`), Ozon | DSO-key |
| Oude regels (HTML/PDF) | `ruimtelijkeplannen.nl/documents/` (`RP_DOCS`), geen CORS, dus via corsproxy.io of allorigins | — |
| Adres zoeken / reverse | PDOK Locatieserver (`LS`) | — |
| Kadastraal perceel | PDOK BRK OGC API (`KADASTER_OGC`) | — |
| Plankaart-overlay; bestemmingen, functieaanduidingen en maatvoering op een punt of perceel | PDOK `plu` WMS (`PLU_WMS`, GetFeatureInfo) | — |
| Achtergrondkaart | CARTO (standaard), PDOK BRT | CARTO-key |

Coördinaten: de DSO- en RP-API's werken in RD (EPSG:28992), Leaflet in WGS84. De omrekening zit ingebouwd in `wgs2rd()`/`rd2wgs()`; er is geen proj4.

## Codekaart (volgorde in het `<script>`)

1. **Foutafhandeling en verbindingstest**: `panelMsg`, `showLibBanner`, de Test-verbinding-knop.
2. **Kaart en coördinaten**: Leaflet-setup, CARTO-lagen, `wgs2rd`/`rd2wgs`.
3. **Perceel**: `haalPerceel`/`toonPerceel` (punt-in-vlak), `tekenPerceelOpCanvas` voor de export.
4. **Zoeken op locatie**: `zoekOpLocatie`, daarna `toonResultaten` → `beoordeelPlannen` → `startWeergave` (`kiesStartFilter` → `renderDocs` → `docKaart`, `toonSituatie`). Plansoorten: `isBestemmingsplan` (ook inpassings-, wijzigings-, uitwerkingsplan en beheersverordening), `isBesluit` (voorbereidingsbesluit), `planSoort`/`planLabel`.
5. **Filters**: het object `FILTER`, `isHerziening` (`RE_HERZIENING`) en `renderFilterBar`.
6. **Bestemmingsplan**:
   - `haalBestemmingen` zoekt met het hele perceel, of met `BUFFER_M` meter rond het punt.
   - `toonBestemmingen` toont de bestemmingenkaart (`#bestCard`).
   - `geselecteerd()` geeft de aangevinkte bestemmingen.
   - **Functieaanduidingen en maatvoering** (maximale bouwhoogte): `haalAanduidingen` (PDOK, lagen `functieaanduiding` en `maatvoering`), `aanduidingenHTML`, `aandGekozen`, `tekenAanduidingen` (laag `aandLaag`), `tekenAanduidingenOpCanvas` voor de PDF, `aanduidingZinnen`/`maatZin` voor de samenvatting.
7. **Regelteksten**:
   - `haalTekst` en `tekstNaarHTML` halen de XHTML uit de RP-API.
   - `downloadRegels` maakt een HTML-download.
   - `xhtmlNaarLijst` zet XHTML om naar platte tekst met a/b/c-opmaak (zie hieronder).
   - `regelsOpmaken` is de gedeelde opmaak van die tekst.
8. **Analoge plannen** (zie hieronder): `analoogInfo`, `toonAnaloog`, `bestemmingenUitPdf`, `pdfAlineas`, `artikelenUitAlineas`.
   - **Besluiten zonder bestemmingen** (voorbereidingsbesluit, beheersverordening zonder vlakken, uitspraak): `toonPlanDocumenten`, `planDocLinks`, `artikelenUitPdf`/`artikelenUitHtml`, `bouwBesluitSamenvatting`.
9. **Terugval voor oude plannen**: `haalExternDoc`, `haalArtikelOud` en `artikelTekstUitHTML` voor losse HTML-regels op ruimtelijkeplannen.nl. `haalArtikelOud` geeft `v._pdfArtikel` direct terug als de bestemming uit een PDF komt.
10. **Samenvatting bestemmingsplan**: `bouwSamenvatting`, `omschrijvingUitArtikel`, `kopieerSamenvatting`.
11. **Omgevingsplannen (Ozon)**: `toonOmgevingsregels`, gebiedsaanwijzingen en functies tekenen, `omgSamenvatting` (`OMG_INTRO`), `omgVolledigePDF`. Bouwhoogte en andere maten komen uit de omgevingsnormen: `omgNormen`, `omgNormenHTML`, `omgGekozenNormen`.
12. **PDF en kaartje**: `volledigePDF` en `maakKaartje`. Beide gebruiken leaflet-image en openen een printvenster, dus pop-ups moeten aan staan.
13. **Adres zoeken, startscherm, instellingen, sleutels onthouden.**

## Domeinkennis die je nodig hebt

- **Bestemmingsplan** (oud recht, Wro, IMRO) en **omgevingsplan** (Omgevingswet, sinds 1-1-2024) kunnen tegelijk op één locatie gelden. Het "tijdelijk deel" van het omgevingsplan bevat de oude bestemmingsplannen.
- **Herzieningen en parapluplannen** liggen over het hoofdplan heen. De gebruiker wil meestal alleen het hoofdplan zien, bijvoorbeeld "Binnenstad".
- **Enkelbestemming** is de hoofdbestemming. **Dubbelbestemming** (Leiding, Waarde, Waterstaat) komt er bovenop. Enkelbestemmingen staan altijd bovenaan.
- **Analoge plannen** zijn "bestemmingsplan artikel 10" en andere plannen met alleen PDF's, zoals Hillesluis in Rotterdam:
  - Het plan-id eindigt op `-`, zonder versie.
  - De RP-API heeft geen bestemmingsvlakken, dus de bestemming is niet automatisch te bepalen.
  - Documenten volgen een vaste naamgeving: `p_<id>.pdf` (plankaart), `v_`/`r_` (regels), `t_` (toelichting, soms met de regels erin).
  - De tool laat de gebruiker de bestemming zelf aanvinken. Die lijst komt uit de regels-PDF (pdf.js, inclusief inspringing op x-positie). Vang dit niet "slim" af met gokken op de kaart: de PDF-kaart heeft geen coördinaten.

- **Adres zonder bestemming** (testgeval: Draaibank 13, Zaandam, Hembrugterrein):
  - Het bestemmingsplan is vernietigd (bij PDOK `historisch`=`1`, dossierstatus `vervallen`) en komt niet meer uit de DSO-zoekactie. Een ander plan raakt het adres alleen met gebiedsaanduidingen (geluidzone).
  - Wat er wel geldt, is een **voorbereidingsbesluit** met voorbeschermingsregels (type `voorbereidingsbesluit`, alleen een besluit-PDF, geen vlakken). Onder de Omgevingswet vervallen die regels in beginsel anderhalf jaar na inwerkingtreding.
  - Het omgevingsplan geeft daar alleen algemene regels (bruidsschat): wel `activiteiten` en `regelteksten`, geen `gebiedsaanwijzingen`.
  - De tool legt dit uit in de kaart "Situatie op dit adres" (`toonSituatie`, gegevens in `_situatie`). Beweer daar niet meer dan de bronnen zeggen: na een vernietigd plan kan een ouder, niet-digitaal plan weer gelden.
- **TAM-plannen** (`imroDocumentMetadata.isTamPlan`): wijzigingen van het omgevingsplan die nog in IMRO-vorm worden gepubliceerd. Een "TAM-omgevingsplan" heeft type `bestemmingsplan`; `regelStatus` "in ontwikkeling" betekent nog niet in werking.
- **Uitwerkings- en wijzigingsplan** geven het adres de echte bestemming; het moederplan heeft daar bijvoorbeeld "Wonen - Uit te werken". Het nieuwste plan met een enkelbestemming op het adres wint.

## Vaste gedrag en keuzes van de gebruiker (niet zomaar veranderen)

- **Start-tab**: bij elke nieuwe zoekopdracht begint de filter op "Bestemmingsplan" als daar een geldend bestemmingsplan ligt, anders op "Omgevingsplan" (als die er is), anders op "Alle". Dit wordt niet onthouden. Zie `kiesStartFilter()`. Is er alleen één omgevingsplan, dan worden de functies meteen geladen (`openOmgevingsplanDirect()`).
- **"Herzieningen, paraplu- en thematische plannen verbergen"** staat bij elke start aan (`FILTER.verberg=true`) en wordt niet onthouden.
- **Vigerend hoofdplan bepalen** (`beoordeelPlannen`): de DSO geeft alle plannen als "geldend" terug, ook thematische plannen voor de hele gemeente (Darkstores, Datacenters) en oudere plannen. De bestemmingen op het adrespunt komen uit één PDOK-verzoek voor alle plannen (`pdokBestemmingenOpPunt`: WMS GetFeatureInfo op de `plu`-plankaart, veld `plangebied` = plan-id, geen key, CORS open). Lukt PDOK niet, dan per plan de RP-API (`puntVlakken`, `PUNT_M`=0,5 m). Houd het aantal RP-verzoeken laag: bij te veel verzoeken achter elkaar blokkeert de RP-server het IP-adres tijdelijk (403, "Request forbidden by administrative rules"). Antwoordt de RP-API niet, dan toont `haalBestemmingen` alleen de PDOK-bestemmingen op het punt (`vlakken._alleenPunt`). Geen vlak: thematisch, verbergen. Meerdere plannen met een enkelbestemming: het nieuwste (planstatusdatum) geldt, de oudere verbergen. Analoge plannen (id op `-`) altijd tonen. De reden staat op de plankaart (`_planCheck.reden`).
  - Een tweede PDOK-verzoek (`pdokContextOpPunt`, lagen `bestemmingsplangebied` en `gebiedsaanduiding`) vertelt welke plangebieden op het punt liggen, ook vervallen. Kent PDOK het plan op dit punt, dan is "geen vlak" betrouwbaar en gaat er geen RP-verzoek uit. Kent PDOK het plan daar niet (bijvoorbeeld net vastgesteld), dan beslist de RP-API.
  - Heeft alleen een herziening een enkelbestemming op het adres, dan wordt die herziening getoond (`_planCheck.toon`). Is er een hoofdplan én een nieuwere herziening met een eigen enkelbestemming, dan blijft het hoofdplan staan en meldt "Situatie op dit adres" de herziening.
- **Situatie op dit adres** (`toonSituatie`) verschijnt alleen als er iets te melden is: geen geldend plan met een bestemming (dan ook vervallen plan, uitspraak en plannen met alleen aanduidingen), een geldend voorbereidingsbesluit, of een herziening met een eigen bestemming. In het gewone geval staat de kaart er niet. Nieuwe kaarten komen eronder (`kaartInvoegen`).
- **Samenvatting bij een besluit** (`bouwBesluitSamenvatting`): vaste inleiding plus de aangevinkte artikelen, zonder de zin "Conclusie: het huidig gebruik …". Begripsbepalingen staan standaard uit.
- **Bestemmingen aanvinken**: alleen de enkelbestemming op het adrespunt (`v._opPunt`) staat aangevinkt. Wat verder op of naast het perceel ligt (bijv. Verkeer-2, Tuin-1), staat onder een eigen kopje, uitgevinkt. Dubbelbestemmingen (Waarde, Leiding) worden nooit automatisch aangevinkt.
- **Functieaanduidingen en maatvoering bij een bestemmingsplan** (`haalAanduidingen`): uit de PDOK-plankaart, dus zonder RP-verzoeken. GetFeatureInfo zoekt in een straal van ongeveer 5 beeldpunten rond het midden; met grotere beeldpunten dekt één verzoek het hele perceel (de lagen zijn bevraagbaar tot schaal 1:50.000, beeldpunt 14 m; een groot perceel gaat in maximaal negen stukken, een nog groter perceel alleen rond het adres). De vlakvorm komt mee.
  - Alleen wat op het adrespunt ligt, staat aangevinkt; wat elders op het perceel ligt, staat uitgevinkt met het aandeel van het perceel erbij. Een aanduiding die alleen de rand van het perceel raakt (`AAND_RAND_M`), hoort bij de buren en valt weg.
  - Aangevinkte functieaanduidingen en maten komen in de samenvatting (één alinea vóór "Conclusie: …") en in de volledige PDF (lijst en getekend op het kaartje). Op de kaart staan de functieaanduidingen standaard aan (schakelaar `#aandKaart`): aangevinkt met volle lijn, niet aangevinkt met stippellijn.
  - Staat er geen maatvoering op de plankaart, dan zegt de kaart dat de bouwhoogte in de bouwregels staat. De samenvatting zwijgt er dan over.
- **Bouwhoogte bij een omgevingsplan** (`omgNormen`): uit `omgevingsnormen` in het antwoord van Presenteren. Alleen de normwaarden op het punt tellen (`geselecteerdeLocatieRefs`). Een waarde is een getal met eenheid, een omschrijving (Haarlem-centrum: "bouwhoogte handhaven") of een verwijzing naar de regeltekst. Bouw- en goothoogte staan aangevinkt, de andere normen niet. Wat is aangevinkt komt als lijstje onder de werkingsgebieden in de samenvatting en in de PDF. De variant met een getal is alleen met nagebootste gegevens getest: er is nog geen echt adres gevonden waar het omgevingsplan de bouwhoogte als getal geeft.
- **Omgevingsplan-functies**: gegroepeerd per functiegroep (`omgGroepVan`, `omgSorteer`), erfgoed apart onderaan, en niets vooraf aangevinkt.
- Blijft er na filteren één plan over, dan klapt dat open.
- **Geen logo linksboven in de header.** Het logo staat alleen op het startscherm, in de PDF's en in het kaartje (`LOGO`).
- **Samenvattingstekst bestemmingsplan**: de vaste zinnen staan in `bouwSamenvatting()`:
  > Het object valt onder het vigerende bestemmingsplan '…'. Dit bestemmingsplan is {planstatus} door de {Gemeente X} op {datum}. Het object heeft op basis van dit plan de bestemming '…'.

  Daarna volgt per bestemming de bestemmingsomschrijving, en de tekst sluit af met "Conclusie: het huidig gebruik …". Verander deze zinnen alleen als de gebruiker dat vraagt.
- **Opsommingen in de samenvatting** (`xhtmlNaarLijst`):
  - Elke lijst telt zelfstandig: hoofdniveau `a.`/`b.`/`c.`, subniveau `1.`/`2.`/`3.` (3 spaties ingesprongen), daaronder `-`.
  - Expliciete nummering uit de bron gaat voor: `start`, `value`, `type`, of een letter in de tekst zelf.
  - Losse `<ol>`'s achter elkaar tellen door, tenzij een item of tussenkop op `:` eindigt; dan wordt het een sublijst.
  - Dit was een echte bug (letters die doortelden over sublijsten heen, bij Breda Binnenstad Centrum-1 en Gemengd-2). Test na elke wijziging opnieuw met geneste lijsten.
- "Bestemingsomschrijving" (tikfout in echte Breda-regels) moet ook herkend worden.

## Stijl

- Thema "Savills navy": de tokens staan in het tweede `:root`-blok. Gebruik `var(--navy)`, `--teal`, `--yellow`, `--cream` en geen losse kleuren.
- Kaarten (`.doc`, `.best-card`) zijn wit op crème. Knoppen in een kaart zijn wit met een navy rand; de primaire actie is geel (`.wide`). Schakelaars in iOS-stijl gebruiken `.kleur-toggle`.
- Zijpaneel links, 410px breed. Onder 900px komt het paneel boven de kaart.
- Houd de UI rustig: geen extra animaties of badges zonder functie.

## Werkwijze

- De gebruiker wil dat het "goed staat": neem de tijd, test, en houd bestaand gedrag intact.
- Pas `index.html` gericht aan (Edit), niet door het bestand opnieuw te schrijven. Het is groot (~2700 regels).
- Dit is een HTML-pagina, geen Node-project. **Test in een headless browser** met Playwright, met gemockte API's:
  - `page.route("https://ruimte.omgevingswet.overheid.nl/**", …)` voor de RP-API
  - `page.route("https://www.ruimtelijkeplannen.nl/**", …)` voor documenten
  - plannen injecteren met `toonResultaten({_embedded:{documenten:[…]}})`
  - startscherm sluiten met `sluitStart()`
  - `$('rpkey').value='x'` en `_lastRD=[x,y]` zetten om flows zonder echte keys te draaien
- Testgereedschap staat in `.test/` (staat in `.gitignore`): een eigen Python-omgeving met Playwright (`.test/venv`, start de geïnstalleerde Edge met `channel="msedge"`, er is geen Node). `regressie.py` draait zeventien nagebootste situaties (A t/m L plannen en situaties, M t/m Q functieaanduidingen, maatvoering en omgevingsnormen) en kan ook een oudere versie testen (`regressie.py pad/naar/oud.html`) om verschillen te zien. `repro_mock.py` speelt Draaibank 13 na met de echte DSO- en PDOK-antwoorden. Werk nooit buiten `ClaudeProjects`, ook niet in een tijdelijke systeemmap.
- Test niet tegen de echte RP-API: die blokkeert het IP-adres snel (403 op alles, ook op de CORS-voorvraag). Boots hem na met `page.route`.
- Op een telefoon (onder 900px) schuift het hele paneel als één geheel; controleer daar dat de lijst bereikbaar is.
- Voor PDF-tests: serveer de map via `python3 -m http.server` (ES-modules werken niet via `file://`). Zet daarna `window.pdfjsLib` zelf; `laadPdfJs()` gebruikt dan die versie.
- Controleer na een wijziging altijd:
  1. geneste a/b/c-lijsten in de samenvatting;
  2. de filters en de start-tab;
  3. een gewoon digitaal plan, een analoog plan en een plan zonder vlakken op die plek;
  4. een adres zonder bestemming met een voorbereidingsbesluit (Draaibank 13, Zaandam);
  5. geen `pageerror`s.
