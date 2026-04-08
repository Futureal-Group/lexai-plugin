# KKV Jogi Skill Rendszer — Tesztelési és Kiértékelési Terv

## Cél

Szisztematikusan ellenőrizni, hogy a 11 KKV jogi skill (1 router + 10 domain) helyesen működik-e:
- A router a megfelelő skill(ek)et választja ki
- A skillek az MCP tool-okat hívják (nem hardkódolt választ adnak)
- Az output formátum, kockázatszint és disclaimer mindig megjelenik
- Keresztterületi kérdéseknél több skill tartalma integrálódik

---

## Tesztelési rétegek

### 1. réteg — Router tesztek (skill kiválasztás)

**Cél:** A `kkv-jogi-router` a helyes skill(ek)et hívja-e?

**Módszer:** `/skill-creator` eval mód vagy manuális futtatás

**Metrika:** Hit rate — 20 prompt-ból hányszor választja a helyes skill(ek)et

#### Teszt esetek — egyszerű routing (1 skill)

| # | Prompt | Elvárt skill | Típus |
|---|---|---|---|
| R1 | "Hány nap szabadság jár egy 42 éves munkavállalónak?" | kkv-munkajog | Egyértelmű |
| R2 | "Melyik adónem éri meg egy szoftverfejlesztő EV-nek?" | kkv-ado | Egyértelmű |
| R3 | "A partnerem 3 hónapja nem fizet, mit tegyek?" | kkv-koveteleskezeles | Egyértelmű |
| R4 | "Védjegyeztessem a logómat?" | kkv-szellemi-tulajdon | Egyértelmű |
| R5 | "Mi kell egy Kft. alapításához?" | kkv-cegjog | Egyértelmű |
| R6 | "Kötelező a cookie banner?" | kkv-gdpr | Egyértelmű |
| R7 | "Mennyi a jótállás egy 50 ezres termékre?" | kkv-fogyasztovedelmi | Egyértelmű |
| R8 | "Irodát bérelnék a cégnek" | kkv-ingatlan | Egyértelmű |
| R9 | "Kávézót akarok nyitni, mi kell hozzá?" | kkv-engedelyek | Egyértelmű |
| R10 | "Mi a különbség a vállalkozási és megbízási szerződés közt?" | kkv-szerzodes | Egyértelmű |

#### Teszt esetek — keresztterületi routing (2+ skill)

| # | Prompt | Elvárt skillek | Típus |
|---|---|---|---|
| R11 | "Kötelező adatkezelési tájékoztató a webshopba?" | kkv-gdpr + kkv-fogyasztovedelmi | Kereszt |
| R12 | "Milyen engedély kell étterem nyitáshoz?" | kkv-engedelyek + kkv-ado | Kereszt |
| R13 | "Székhelyszolgáltatást vennék igénybe a Kft.-mhez" | kkv-ingatlan + kkv-cegjog | Kereszt |
| R14 | "Freelancert szerződtetnék React fejlesztésre" | kkv-szerzodes + kkv-szellemi-tulajdon + kkv-munkajog | Kereszt (3-as) |
| R15 | "EV-ből Kft.-t akarok csinálni" | kkv-cegjog + kkv-ado | Kereszt |
| R16 | "Airbnb-ztetnék, mi kell hozzá?" | kkv-engedelyek + kkv-ado | Kereszt |
| R17 | "Home office-ban dolgoznak a munkavállalóim" | kkv-munkajog + kkv-gdpr | Kereszt |
| R18 | "Marketplace-en akarok eladni" | kkv-fogyasztovedelmi + kkv-ado | Kereszt |

#### Teszt esetek — nehéz / edge case routing

| # | Prompt | Elvárt skill(ek) | Típus |
|---|---|---|---|
| R19 | "Nem akar fizetni a vevőm a webshopból" | kkv-koveteleskezeles (+ esetleg kkv-fogyasztovedelmi) | Ambiguous |
| R20 | "Szoftvert fejlesztettem, hogyan védjem le?" | kkv-szellemi-tulajdon | Indirekt |
| R21 | "Baj van, betörtek a rendszerbe" | kkv-gdpr (incidens) | Implicit |
| R22 | "Mennyit kereshetek KATA-sként?" | kkv-ado (+ esetleg kkv-munkajog KATA elhatárolás) | Ambiguous |
| R23 | "Kirúghatom a munkavállalóm?" | kkv-munkajog (⛔ kritikus) | Kockázat trigger |

**Értékelési szempontok:**
- ✅ Helyes skill(ek) kiválasztva
- ✅ Keresztterületi kérdésnél mindkét/mindhárom skill tartalma megjelenik
- ✅ Ambiguus kérdésnél ésszerű döntés (nem kell tökéletesnek lennie)
- ❌ Rossz skill kiválasztva VAGY releváns skill kihagyva

**Elvárt minimum:** 90% hit rate (20/23)

---

### 2. réteg — MCP tool használat tesztek

**Cél:** A skill valóban hívja-e az MCP tool-okat, nem a hardkódolt tartalmat adja-e vissza?

**Módszer:** Manuális futtatás + MCP Inspector (ha elérhető) — a válaszban keressük az MCP hívás nyomait

#### Teszt esetek

| # | Prompt | Elvárt MCP hívás(ok) | Ellenőrzés |
|---|---|---|---|
| M1 | "Hány nap szabadság jár egy 42 éves munkavállalónak?" | `get_provision(document_id: "hu-law-2012-1-00-00", section: "115")` | Pontos Mt. szöveg a válaszban, nem kerek szám |
| M2 | "Mekkora a Kft. minimum törzstőkéje?" | `get_provision()` a Ptk. releváns §-ára | Aktuális Ptk. szöveg, nem "3M Ft" magában |
| M3 | "Mekkora bírság jár GDPR sértésért?" | `get_provision()` + `get_eu_basis()` | GDPR cikk + Infotv. párhuzamos hivatkozás |
| M4 | "Mikortól hatályos az Mt. home office szabálya?" | `check_currency(document_id: "hu-law-2012-1-00-00")` | Hatályossági dátum az MCP-ből |
| M5 | "Mi a késedelmi kamat mértéke B2B-ben?" | `get_provision()` a Ptk. 6:48. §-ra | Pontos szöveg, nem kerekített % |

**Értékelési szempontok:**
- ✅ Az MCP tool meghívásra került (a válasz az MCP-ből származó szöveget tartalmaz)
- ✅ `check_currency()` futott a válasz előtt
- ✅ Ha hivatkozás van: `validate_citation()` is futott
- ❌ A válasz kizárólag a skill hardkódolt tartalmát ismétli
- ❌ A válaszban elavult vagy téves összeg/határérték jelenik meg

**Elvárt minimum:** 100% — minden válasz MCP-alapú kell legyen

---

### 3. réteg — Output formátum és minőség tesztek

**Cél:** A válasz megfelel-e az `output-template.md` struktúrának?

**Módszer:** Manuális ellenőrzés vagy LLM-as-judge (skill-creator eval)

#### Checklist minden válaszra

```
OUTPUT FORMÁTUM ELLENŐRZÉS:
☐ Kockázatszint jelölve (🟢/🟡/🔴/⛔)
☐ Rövid válasz jelen van (1-2 mondat, mit kell tenni)
☐ Releváns jogszabályok felsorolva (törvény + § + egy mondat)
☐ Teendők lista (konkrét, cselekvőképes lépések)
☐ "Ügyvédhez fordulj, ha:" blokk jelen van
☐ Disclaimer a végén ("Ez a tájékoztatás nem minősül jogi tanácsadásnak...")
☐ Magyar, KKV-barát nyelv (nem jogi szakzsargon)
☐ Határidők számban (nem "haladéktalanul")
```

#### Kockázatszint validáció

| # | Prompt | Elvárt kockázatszint |
|---|---|---|
| O1 | "Hány nap szabadság jár?" | 🟢 Alacsony |
| O2 | "Hiányzik az adatkezelési tájékoztatónk" | 🟡 Közepes |
| O3 | "Betörtek az adatbázisba, személyes adatok szivárogtak" | 🔴 Magas vagy ⛔ Kritikus |
| O4 | "Kirúghatom azonnali hatállyal a munkavállalóm?" | ⛔ Kritikus (ügyvéd flag) |
| O5 | "Webshop tájékoztatókat frissíteném" | 🟢 Alacsony |

**Értékelési szempontok:**
- ✅ A kockázatszint megfelel az elvárt szintnek (±1 szint elfogadható)
- ✅ ⛔ kritikus esetben ügyvéd ajánlás automatikusan megjelenik
- ✅ Minden output elem jelen van
- ❌ Hiányzik a disclaimer
- ❌ Nincs kockázatszint jelölés
- ❌ Jogi szakzsargon dominál

**Elvárt minimum:** 100% a formátum checklist-en, 80% a kockázatszint pontosságon

---

### 4. réteg — MCP tool description quality (tool szintű)

**Cél:** A hungarian-law MCP server tool leírásai elég egyértelműek-e?

**Módszer:** `mcp-tef` futtatás VAGY manuális tool description audit

#### Tool átfedés ellenőrzés

Potenciálisan átfedő tool-párok, amiket tesztelni kell:

| Tool A | Tool B | Mikor melyiket kell választani |
|---|---|---|
| `search_legislation` | `build_legal_stance` | search: ismert törvény keresés; build: átfogó kutatás több törvényből |
| `search_legislation` | `get_provision` | search: nem tudom a §-t; get_provision: tudom a §-t |
| `get_eu_basis` | `get_provision_eu_basis` | get_eu_basis: törvény szintű; get_provision_eu_basis: § szintű |
| `validate_citation` | `format_citation` | validate: létezik-e; format: csak formázás |

#### Teszt: helyes tool kiválasztás

| # | Szituáció | Elvárt tool | Rossz választás |
|---|---|---|---|
| T1 | "Keresem az Mt. szabadságra vonatkozó részét" | `search_legislation` | `get_provision` (nem tudom a §-t) |
| T2 | "Mi az Mt. 115. § szövege?" | `get_provision` | `search_legislation` (felesleges keresés) |
| T3 | "Hatályos-e még a 2006-os cégtörvény?" | `check_currency` | `get_provision` (nem az kell) |
| T4 | "Milyen EU irányelven alapul az Infotv.?" | `get_eu_basis` | `search_legislation` (nem erről szól) |
| T5 | "Helyes ez a hivatkozás: Ptk. 6:48. §?" | `validate_citation` | `get_provision` (nem validálás) |

---

## Végrehajtási terv

### Fázis 1 — Router smoke test (30 perc)

1. Futtatsd végig az R1-R23 prompt-okat a `kkv-jogi-router` skill-lel
2. Jegyezd fel: melyik skill(ek)et választotta
3. Számold ki a hit rate-et
4. Ha < 90%: a router kulcsszó tábláját kell finomítani

**Hogyan futtasd:**
```
# Claude Code-ban:
/kkv-jogi-router
> [beírod a teszt prompt-ot]
> figyeled melyik skill(ek)et hívja
```

### Fázis 2 — MCP integration smoke test (30 perc)

1. Futtatsd az M1-M5 prompt-okat
2. Ellenőrizd: az MCP tool-ok meghívásra kerültek-e
3. Ellenőrizd: a válasz tartalmaz-e pontos jogszabályszöveget (nem kerekített számot)
4. Ha nem hívja az MCP-t: a skill MCP-first blokkját kell erősíteni

**Hogyan ellenőrizd:**
- A válaszban szerepelnek-e pontos § szövegrészletek (idézőjelben vagy blockquote-ban)
- A válasz hivatkozik-e a `check_currency` vagy `validate_citation` eredményére
- Az MCP Inspector-ban látszanak-e a tool hívások (ha fut)

### Fázis 3 — Output minőség (20 perc)

1. Az előző fázisok válaszaiból 10-et ellenőrizz az output checklist-tel
2. Kockázatszint validáció az O1-O5 prompt-okkal
3. Ha hiányzik formátum elem: az output-template.md hivatkozást kell erősíteni a skill-ben

### Fázis 4 — skill-creator eval (opcionális, mélyebb)

Ha a skill-creator elérhető:

```
/skill-creator eval
```

Prompt bank definiálása:
- 23 router teszt (R1-R23) — elvárt: helyes skill kiválasztás
- 5 MCP teszt (M1-M5) — elvárt: MCP tool hívás történik
- 5 output teszt (O1-O5) — elvárt: kockázatszint + formátum helyes

A/B összehasonlítás: skill vs. skill nélkül (mennyivel jobb a válasz a skill-lel?)

---

## Eredmény dokumentálás

### Scorecard sablon

Minden teszt futtatás után töltsd ki:

```
TESZT FUTTATÁS: [dátum]
TESZTELŐ: [név]

ROUTER HIT RATE:        __/23 (__%)
MCP TOOL HASZNÁLAT:     __/5  (__%)
OUTPUT FORMÁTUM:        __/10 (__%)
KOCKÁZATSZINT HELYES:   __/5  (__%)

HIBÁK:
- [R#] [mi történt] [elvárt vs. kapott]
- ...

JAVÍTÁSI JAVASLATOK:
- [melyik skill / router / tool leírás kell módosítani]
- ...
```

### Regressziós teszt

Minden skill módosítás után futtasd újra:
1. Az adott skill-hez tartozó router teszteket (R#)
2. A hozzá tartozó MCP tesztet (M#)
3. Legalább 1 output formátum ellenőrzést

---

## Eszközök

| Eszköz | Mire való | Prioritás |
|---|---|---|
| **Claude Code manuális futtatás** | Router + skill + output tesztelés | ⭐ Elsődleges |
| **MCP Inspector** | MCP tool hívások valós idejű figyelése | ⭐ Elsődleges |
| **/skill-creator eval** | Automatizált prompt eval + A/B teszt | 🔄 Másodlagos |
| **mcp-tef** | Tool description quality audit | 🔄 Opcionális |
| **DeepEval MCP Use** | LLM-as-judge az MCP hívások minőségére | 🔄 Opcionális |

---

## Sikeresség kritériumai

| Metrika | Minimum | Cél |
|---|---|---|
| Router hit rate | 90% | 95%+ |
| MCP tool használat | 100% | 100% |
| Output formátum compliance | 90% | 100% |
| Kockázatszint pontosság | 80% | 90%+ |
| Disclaimer jelenlét | 100% | 100% |
| Keresztterületi helyes routing | 80% | 90%+ |
