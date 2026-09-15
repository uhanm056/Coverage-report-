# Coverage Report — kontext projektu

Nástroj pro sledování pokrytí u zákazníka. Jeden samostatný HTML soubor, běží celý
v prohlížeči, nasazuje se na SharePoint. Žádný build, žádné CDN, žádný backend.

**Soubor:** `Coverage_Report.html` (~320 kB)
**Online:** https://uhanm056.github.io/Coverage-report-/ (GitHub Pages, veřejné, viz Nasazení)
**Autor / vlastník:** Milan, Operations Manager, Yanfeng Planá, OV51/64
**Zákazník:** Mercedes-Benz, projekt **X540**, závod Brémy, díly Instrument Panel
**Jazyk UI:** čeština. Komentáře v kódu česky.

---

## Tvrdá omezení

Tyhle věci neporušuj, jinak nástroj v cílovém prostředí přestane fungovat:

1. **Jeden soubor.** Žádné externí `<script src>`, `<link>`, fonty ani obrázky.
   Firemní síť je blokuje. SheetJS je proto inlinovaný v souboru.
2. **Žádný build step.** Soubor se otevírá přímo z disku nebo SharePointu.
   Žádné moduly, žádný bundler, žádné `import`.
3. **Nic neodchází ven.** Parsování Excelu, generování exportů i kreslení obrázků
   běží lokálně. Zákaznická data nesmí opustit prohlížeč.
4. **`localStorage` jen v `try/catch`.** Používá se na uložení pravidel. Když selže,
   aplikace musí normálně fungovat dál.
5. **Žádná zapečená data.** Repozitář i nasazená stránka jsou veřejné. V souboru
   musí být `let D = null;`, data se berou výhradně z nahraného sešitu. Nikdy
   necommitovat verzi s vyplněným `D` ani testovací sešit s reálnými čísly.
   Workflow to před nasazením kontroluje a s daty v souboru nenasadí.

---

## Struktura souboru

HTML → CSS v `<style>` → jeden `<script>`. Skript má sekce označené komentáři
ve tvaru `/* ---------- název ---------- */` (deset pomlček z každé strany),
v tomto pořadí:

| Sekce | Obsah |
|---|---|
| `SheetJS (inline)` | knihovna xlsx 0.18.5, mini build. **Needituj, přeskoč.** Vlastní kód začíná až za ní. |
| `data` | `let D = null` (data až po uploadu), konstanty `VARS`, `PROJECT`, `CUSTOMER`, `MAIL_SUBJECT`, `SUBLINE` |
| `rules` | stav pravidel `R`, ukládání, výpočet požadavků, barvicí funkce |
| `parsing` | `parseWorkbook()` — čtení nahraného sešitu |
| `rendering` | vykreslení všech tabulek a karet |
| `interactions` | obsluha vstupů, tlačítek, přepínačů |
| `export` | HTML a XLSX exporty |
| `mail` | tělo mailu a `.eml` |
| `PNG obrázek` | kreslení na canvas |
| `upload` | drag & drop, načtení souboru |

Seznam sekcí i s čísly řádků vypíše:

```sh
grep -n '^/\* ---------- .* ---------- \*/' Coverage_Report.html
```

Konstanty projektu jsou na jednom místě hned na začátku sekce `data`:

```js
const PROJECT  = 'X540';
const CUSTOMER = 'Mercedes-Benz';
const MAIL_SUBJECT = 'Coverage Report / Bestandsübersicht / Plana / ' + PROJECT;
const SUBLINE = CUSTOMER+' · '+PROJECT+' · Instrument Panel · Yanfeng Planá';
```

Změna platformy = přepsat tyhle řádky, propíše se do hlaviček, obrázků i mailu.

---

## Datový model

`D` je výsledek `parseWorkbook()` a drží všechno, co se zobrazuje:

```js
D = {
  covDate,          // '10.09.2026' — první den horizontu
  dates[],          // ['10.09.', '11.09.', …] krátké popisky sloupců
  datesFull[],      // ['10.09.2026', …]
  iso[],            // ['2026-09-10', …] pro výpočet kalendářního rozpětí
  summary[],        // agregace za varianty, viz níž
  items[],          // jednotlivé díly
  status[],         // {src, file, date, status} z listu Status
  mng[],            // bloky z listu MNG: {title, dates, datesFull, rows}
  rules             // nepovinná pravidla z listu Pravidla, jinak null
}
```

**Varianta** je vždy jedna ze čtyř: `'LHD'`, `'LHD HUD'`, `'RHD'`, `'RHD HUD'`
(konstanta `VARS`). Určuje se v `classify(desc)` z názvu dílu regulárem
`\bRHD\b` a `\bHUD\b`. Ověřeno proti listu Overview ve zdrojovém sešitu, sedí přesně.

**Položka** `D.items[i]`:
`{variant, desc, item, plana, planaPart, custStock, transit, firstUnc, bal[]}`
`bal[]` jsou bilance po dnech, délka odpovídá `D.dates`.
`transit` je součet sloupců Transit a Additional Transit.

**Souhrn** `D.summary[i]`: totéž agregované za variantu plus `items` (počet dílů).

---

## Zdrojový sešit Coverage.xlsx

Parsuje se tolerantně, ne podle pevných adres buněk:

- **list `Coverage`** — hlavička se hledá podle textu `Description` ve sloupci A.
  Datové sloupce se poznají tak, že hodnota v hlavičce je skutečné datum, takže
  horizont se může zkrátit nebo prodloužit. Ostatní sloupce podle názvu
  (`Plana stock`, `Customer stock`, `Transit`, `Additional transit`, `Item number`,
  `Date uncovered`). Data končí prvním prázdným popisem.
- **list `Status`** — zdroje a jejich stav, hlavička na prvním řádku.
- **list `MNG`** — bloky pohledů managementu. Řádek s hodnotou jen ve sloupci A
  zahajuje nový blok, řádek `Variant` nese datumy, ostatní řádky jsou varianty.
  Reálně jsou tam dva bloky: `Bremen Stock` a `Bremen+Plana Stock`.
- **list `Pravidla`** (nepovinný, uživatel si ho může přidat) — sloupce
  Varianta / Odvolávka / Cíl zákazník / Cíl pipeline. Když existuje, přebije
  hodnoty v UI. Rozpoznává se i pod názvem `Rules` nebo `Settings`.

Když list chybí nebo se struktura rozejde, `parseWorkbook()` vyhodí `Error`
s českou hláškou, kterou `load()` zobrazí uživateli. Nikdy nesmí spadnout do
prázdné stránky.

---

## Logika pokrytí — ZÁSADNÍ

Vyhodnocuje se **na dvou úrovních** a je to vědomé rozhodnutí, ne nedodělek.

### Úroveň varianty — pravidlo ve dnech

```
požadavek (ks) = denní odvolávka × cílové pokrytí pipeline
```

Barví se funkcí `cls(v, first, allZero, req)`:
záporná bilance → červená, pod požadavkem → oranžová, nad požadavkem → zelená.
Platí pro matici po dnech, KPI karty a součtové řádky.

### Úroveň dílu — jen nulová hranice

Funkce `clsItem(v, allZero)`: zelená, nebo červená, nebo šedá u řádků bez pohybu.
**Pravidlo ve dnech se na jednotlivý item number nepoužívá.**

Důvod: u dílu s odvolávkou 1 ks/den by požadavek vyšel na 5 kusů, u dílu s nulovou
odvolávkou na nulu. Dny jsou řídicí ukazatel pro tok, ne pro jeden low-runner.
Tohle se řešilo a zavrhlo, nevracej to zpátky. Kdy díl vypadne, ukazuje sloupec
**První nekrytá objednávka**, to je na úrovni dílu ta podstatná informace.

### Denní odvolávka

`autoDaily(bal)` = `(bal[0] - bal[poslední]) / spanDays()`, kde `spanDays()` je
kalendářní rozpětí horizontu, ne počet sloupců (v datech chybí víkendy).
Ruční hodnota v poli automatiku přebije, prázdné pole ji vrátí zpátky.

### Stav v tabulce pravidel

`OK` = bilance prvního dne ≥ požadavek **a zároveň** pokrytí u zákazníka splňuje
svůj cíl. `POD CÍLEM` = kladná bilance pod požadavkem. `NOK` = záporná bilance.

### Cíl u zákazníka

`R.custDays` se používá **pouze** ve sloupci Pokrytí zákazník a ve stavu.
Do barev v maticích nevstupuje.

### Transit

Do bilance je započítaný přímo výpočtem ve zdrojovém sešitu. Proto přepínač
`R.transitIncluded` (výchozí `true`) a funkce `missingSources()`, která
z varování odfiltruje řádek Transit — jinak by se stav `MISSING` z listu Status
propisoval jako chyba dat až do mailu zákazníkovi. Ostatní zdroje se hlásí normálně.

---

## Stav pravidel

```js
R = {
  pipeDays: 5,           // cíl pro celou pipeline
  custDays: 2,           // cíl u zákazníka
  transitIncluded: true,
  split: '2 dny u zákazníka + 1 den v transitu + 2 dny v Plané',  // jen popisek
  per: { 'LHD': {daily: null}, … }   // null = automatický dopočet
}
```

Ukládá se do `localStorage` pod klíčem `yfCoverageRules`, zápis je debouncovaný
přes `saveSoon()`. Jde exportovat a importovat jako JSON.

Výchozí hodnoty pocházejí z QHELP akčního plánu, kde byl cíl 5 dnů pro celý řetězec.
`split` je jen text vedle polí, nikde se nepočítá.

---

## Prázdný stav bez dat

Při startu je `D = null`. `renderAll()` v tom případě nic nedělá a `<body>` dostane
třídu `nodata`: CSS schová všechno v hlavním `.wrap` kromě pole pro upload, chybové
hlášky, karty `#intro` s návodem a patičky. `load()` po úspěšném `parseWorkbook()`
třídu sejme a vykreslí vše. Když parsování selže, `D` zůstává `null` a stránka
zůstává v prázdném stavu s chybou.

---

## Vykreslování tabulky pravidel — POZOR

Tabulka pravidel se staví **jednou** funkcí `renderRulesShell()`. Při každé změně
se volá `updateRulesTable()`, která přepíše **jen vypočtené buňky**, nikdy inputy.

Původně se překreslovala celá a při psaní skákal kurzor. Když budeš cokoliv
v téhle tabulce měnit, drž tohle rozdělení. Hodnota do inputu se zapisuje jen
tehdy, když v něm uživatel zrovna nestojí (`document.activeElement !== input`).

Vstupy mají navíc: označení obsahu při fokusu, Enter a šipky pro skok mezi poli,
křížek pro návrat na automatický dopočet.

---

## Výstupy

Všechny respektují aktuálně nastavená pravidla.

| Výstup | Funkce | Pro koho |
|---|---|---|
| `.eml` s přílohou | `buildEml(lang)` | zákazník — otevře se v Outlooku jako koncept |
| tělo mailu do schránky | `buildMailHtml(lang)` | zákazník |
| HTML soubor | `buildExportHtml(kind)` | oba, tisknutelné do PDF |
| XLSX | `buildXlsx(kind)` | oba |
| PNG souhrn | `drawPng(kind)` | oba |
| PNG detail | `drawDetailPng()` | zákazník |

`kind` je `'cust'` (bilance celé pipeline) nebo `'mng'` (zásoba v Brémách z listu MNG).

**Mail** má jazykové varianty EN / DE / CZ ve slovníku `T3`, výchozí EN.
Tělo je psané inline styly bez `<style>` bloků a bez flexboxu, protože to musí
projít Outlookem. Hlavičky tabulek jsou `<td>`, ne `<th>` — Outlook u `<th>`
prosazuje vlastní barvu textu a inline `color` ignoruje. Barva je navíc zopakovaná
ve vnořeném `<span>` a řádek má atribut `bgcolor`. Nevracej to na `<th>`.

`.eml` nese hlavičku `X-Unsent: 1`, aby se otevřel jako rozepsaná zpráva.
Předmět je base64 v `=?utf-8?B?…?=`, tělo i příloha base64 zalomené po 76 znacích.

**PNG** se kreslí ručně na canvas, žádná knihovna. Scale 2× kvůli ostrosti.
Barvy jsou v mapách `HEX` a `BADGE`. `dnu(n)` řeší české skloňování dnů.

---

## Vizuální styl

Uložený firemní styl Yanfeng, drž se ho:

- hlavička tmavě modrá `#1A4F71`, bílý titulek, světlé pilulkové štítky
- pozadí `#F4F6F8`, bílé karty s tenkým barevným proužkem nahoře
- sémantické barvy: červená `#E74C3C`, modrá `#3F70AE`, oranžová `#E89818`,
  zelená `#28A745`
- výplně buněk: `#FDEAE7` / `#FDF3E0` / `#EAF7EE` / `#F4F6F8`
- pilulkové taby a tlačítka, aktivní stav modrý
- písmo Inter / Segoe UI, v mailu Arial
- celkový dojem: provozní BI pro výrobu, ne redakční layout

---

## Nasazení

GitHub Pages přes `.github/workflows/pages.yml`. Spouští se pushem
`Coverage_Report.html` do `main`, případně ručně (workflow_dispatch).
Prostředí `github-pages` pouští nasazení jen z `main`, z jiných větví
běh spadne na pravidlu ochrany prostředí. Workflow nejdřív ověří, že `let D = null;` a že soubor
neodkazuje ven, pak nasadí soubor jako `index.html` i `Coverage_Report.html`.
Žádný build, nasazuje se přesně to, co je v repu.

Kdyby první běh spadl na zapnutí Pages: Settings → Pages → Source
„GitHub Actions" a workflow spustit znovu.

---

## Jak testovat bez prohlížeče

V repu není test runner, ale osvědčilo se tohle:

1. **Syntaxe** — vyříznout skript za komentářem `data` a prohnat ho
   `new Function(...)` se zastubovanými globály.
2. **Parsování a výpočty** — načíst `Coverage.xlsx` přes SheetJS v Node,
   zavolat `parseWorkbook()` a porovnat proti listu Overview.
3. **PNG** — `@napi-rs/canvas` umí nahradit `document.createElement('canvas')`
   a obrázek se dá uložit a prohlédnout.
4. **`.eml`** — rozparsovat pythonním `email` modulem, ověřit předmět,
   `X-Unsent`, tělo a přílohu.

5. **Prohlížeč** — Playwright s Chromiem: otevřít soubor přes `file://`, ověřit
   třídu `nodata`, přes `setInputFiles('#file', …)` nahrát sešit a zkontrolovat
   KPI karty a tabulku pravidel. Testovací sešit se dá sestavit ze SheetJS
   vyříznutého z HTML, drží se mimo repo.

Kontrolní čísla na datech k 10.09.2026: LHD 174 ks/den, LHD HUD 88, RHD 35,
RHD HUD 32. Bilance prvního dne 1 035 / 462 / 193 / 115. Dílů celkem 58.
Sešit s těmito daty je jen lokálně u Milana, do repa nepatří.

---

## Otevřené věci

- **Štítek na KPI kartě**, když některý díl padá dřív než varianta jako celek.
  Aktuálně je IP LHD jako varianta první den v pravidle (1 035 proti požadavku 870),
  ale tři díly uvnitř padají už 15.09. Mix problém, který teď hlídá jen sloupec
  první nekryté objednávky.
- **Minimální podlaha v kusech na úrovni dílu** — zvažovalo se jako doplněk
  k variantě A, odloženo. Chtělo by to jedno číslo platné pro všechny díly.
- **Historie snapshotů.** Každý upload přepíše ten předchozí, trend přes více
  coverage dat neexistuje. Porovnání proti listu MNG je jediná časová osa.
- **Nejasnost ve zdrojových datech:** list Status hlásí Transit jako `MISSING`,
  ale sloupec Transit hodnoty obsahuje a list Poznámky má „Přidat nezahrnutý
  transit – done". Vyřešeno přepínačem, ale samotný sešit by si zasloužil pročistit.

---

## Styl práce

- Výstupy rovnou použitelné, bez teoretických úvodů.
- Raději dvě až tři varianty s kompromisy než jedno doporučení.
- Postupné iterace, ne velká specifikace dopředu.
- Maximálně jedna doplňující otázka na kolo.
