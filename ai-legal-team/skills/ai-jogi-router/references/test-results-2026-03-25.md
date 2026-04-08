# KKV Jogi Skill Rendszer — Teszt Eredmények

**Teszt dátum:** 2026-03-25
**Tesztelő:** Claude + Gergő

---

## Összesített Scorecard

### Skill + Router tesztek

| Metrika | Eredmény | Cél | Státusz |
|---|---|---|---|
| Router hit rate | 4/4 (100%) | 90% | ✅ Megfelelt |
| MCP `get_provision` használat | 4/4 (100%) | 100% | ✅ Megfelelt |
| Output formátum compliance | 4/4 (100%) | 90% | ✅ Megfelelt |
| Kockázatszint pontosság | 4/4 (100%) | 80% | ✅ Megfelelt |
| Disclaimer jelenlét | 4/4 (100%) | 100% | ✅ Megfelelt |
| Keresztterületi routing | 2/2 (100%) | 80% | ✅ Megfelelt |

### Document ID validáció (minden skill)

| Skill | Törvény | document_id | `get_provision` működik? |
|---|---|---|---|
| kkv-munkajog | Mt. (2012. évi I. tv.) | `hu-law-2012-1-00-00` | ✅ |
| kkv-szerzodes | Ptk. (2013. évi V. tv.) | `hu-law-2013-5-00-00` | ✅ |
| kkv-cegjog | Ctv. (2006. évi V. tv.) | `hu-law-2006-5-00-00` | ✅ |
| kkv-cegjog | Ptk. (társasági jog) | `hu-law-2013-5-00-00` | ✅ |
| kkv-gdpr | Infotv. (2011. évi CXII. tv.) | `act-cxii-2011-info-self-determination` | ✅ |
| kkv-ado | ÁFA tv. (2007. évi CXXVII. tv.) | `hu-law-2007-127-00-00` | ✅ |
| kkv-ado | Art. (2017. évi CL. tv.) | `hu-law-2017-150-00-00` | ✅ |
| kkv-koveteleskezeles | Fmhtv. (2009. évi L. tv.) | `hu-law-2009-50-00-00` | ✅ |
| kkv-ingatlan | Ptk. bérleti szerz. | `hu-law-2013-5-00-00` (6:331. §) | ✅ |
| kkv-fogyasztovedelmi | Fgytv. (1997. évi CLV. tv.) | `hu-law-1997-155-00-00` | ✅ |
| kkv-szellemi-tulajdon | Vt. (1997. évi XI. tv.) | `hu-law-1997-11-00-00` | ✅ |
| kkv-szellemi-tulajdon | Szjt. (1999. évi LXXVI. tv.) | `hu-law-1999-76-00-00` | ✅ |
| kkv-engedelyek | Kertv. (2005. évi CLXIV. tv.) | `hu-law-2005-164-00-00` | ✅ |

**Eredmény: 13/13 document_id helyes és működik (100%)**

### MCP Tool tesztek (javítás után — 2. futtatás)

| Tool | Működik? | Megjegyzés |
|---|---|---|
| `search_legislation` | ✅ | FTS5 keresés magyar és angol kulcsszavakkal |
| `get_provision` | ✅ | Pontos, szó szerinti szöveg minden tesztelt §-ra |
| `check_currency` | ✅ | Státusz, hatályba lépés dátuma helyes |
| `build_legal_stance` | ✅ | Több törvényből aggregál |
| `validate_citation` | ✅ **JAVÍTVA** | Magyar formátum ("2012. évi I. törvény 116. §") + hu-law ID formátum is működik |
| `format_citation` | ✅ **JAVÍTVA** | `hu-law-2012-1-00-00 s116` → `"2012. évi I. törvény a munka törvénykönyvéről 116. §"` |
| `get_eu_basis` | ✅ **JAVÍTVA** | Infotv.→GDPR, Fgytv.→Consumer Rights+Omnibus, ÁFA→VAT Directive, Szjt.→InfoSoc+Software, Mt.→Working Time |
| `validate_eu_compliance` | ✅ **JAVÍTVA** | Infotv. → `"compliant"`, `eu_references_found: 1` |

### MCP Tool javítások (elvégezve a teszt során)

#### `validate_citation` — JAVÍTVA ✅

**Probléma:** A `resolveDocumentId()` és `parseCitation()` nem ismerte fel a magyar formátumot.

**Javítás:**
- `statute-id.ts`: `parseHungarianReference()` + `romanToArabic()` helper-ek — `"2012. évi I. törvény"` → `hu-law-2012-1-00-00`
- `validate-citation.ts`: Magyar `"NNN. §"` + `"hu-law-* sNNN"` regex pattern-ek

**Eredmény:** `"2012. évi I. törvény 116. §"` → `valid: true, document_id: "hu-law-2012-1-00-00", provision_ref: "s116"`

#### `format_citation` — JAVÍTVA ✅

**Probléma:** Nem kapott `db` paramétert, rossz return típus (`FormatCitationResult` vs `ToolResponse`).

**Javítás:**
- `db` paraméter + `ToolResponse<FormatCitationResult>` return típus + `_metadata`
- Magyar formátum parser + DB title lookup
- `registry.ts` frissítve

**Eredmény:** `"hu-law-2012-1-00-00 s116"` → `"2012. évi I. törvény a munka törvénykönyvéről 116. §"`

#### `get_eu_basis` + `validate_eu_compliance` — JAVÍTVA ✅

**Probléma:** Az `eu_documents` és `eu_references` táblák üresek voltak.

**Javítás:**
- `data/eu-mappings.json` seed fájl létrehozva (8 mapping: GDPR, Consumer Rights, Omnibus, VAT, Trade Marks, InfoSoc, Software, Working Time)
- `build-db.ts` kiegészítve a seed fájl betöltéssel
- Adatbázis újraépítve: **58 EU documents, 103 EU references**

**Eredmény:** Infotv. → GDPR `"compliant"`, Fgytv. → Consumer Rights + Omnibus, ÁFA → VAT Directive

#### Megmaradt kisebb korlátok

- `validate_citation`: Ptk. `"6:272. §"` formátumnál a provision_ref nem jön vissza (a kettőspont nem standard section formátum) — a document felismerés helyes
- `format_citation`: Ha a bemeneti formátum már magyar (`"2009. évi L. törvény 3. §"`), nem bővíti ki a teljes címmel (csak akkor, ha `hu-law-*` ID-t kap)

---

## Részletes teszt eredmények

### R1 — Egyszerű routing (kkv-munkajog)

**Prompt:** "Hány nap szabadság jár egy 42 éves munkavállalónak?"
**Elvárt skill:** kkv-munkajog
**Kapott skill:** ✅ kkv-munkajog

| Szempont | Eredmény | Megjegyzés |
|---|---|---|
| Routing | ✅ | "szabadság" + "munkavállaló" → munkajog |
| MCP hívás | ✅ | `get_provision(hu-law-2012-1-00-00, 116)` + `get_provision(..., 117)` + `check_currency()` |
| Pontos szöveg | ✅ | Mt. 116-117. § szó szerint az MCP-ből |
| Output formátum | ✅ | Kockázatszint + számítás + teendők |
| Kockázatszint | ✅ | 🟢 Alacsony |
| Disclaimer | ✅ | |

**Eredmény:** 20 alap + 8 pótszabadság (41. életévtől) = **28 munkanap**

---

### R14 — Háromszoros keresztterületi routing

**Prompt:** "Freelancert szerződtetnék React fejlesztésre"
**Elvárt skillek:** kkv-szerzodes + kkv-szellemi-tulajdon + kkv-munkajog
**Kapott skillek:** ✅ mindhárom

| Szempont | Eredmény | Megjegyzés |
|---|---|---|
| Routing | ✅ | 3-as routing helyes |
| MCP hívás | ✅ | `get_provision(Ptk. 6:272)` + `get_provision(Szjt. 58)` + `get_provision(Szjt. 30)` + `search_legislation(színlelt szerződés)` |
| Pontos szöveg | ✅ | Ptk., Szjt. szó szerint |
| Output formátum | ✅ | 3 szekció (szerződés, IP, munkajog) + összefoglalt teendők |
| Kockázatszint | ✅ | 🟡 Közepes |
| IP figyelmeztetés | ✅ | "Ha a szerződésben nem rendelkezel az IP átruházásról → a forráskód a freelancernél marad!" |
| KATA elhatárolás | ✅ | Átminősítés kockázat + védekezési tippek |
| Disclaimer | ✅ | |

---

### R21 — Edge case: implicit GDPR incidens

**Prompt:** "Baj van, betörtek a rendszerbe"
**Elvárt skill:** kkv-gdpr
**Kapott skill:** ✅ kkv-gdpr

| Szempont | Eredmény | Megjegyzés |
|---|---|---|
| Routing | ✅ | Nincs explicit kulcsszó, de kontextusból felismerte |
| MCP hívás | ✅ | `build_legal_stance()` + `search_legislation(incidens NAIH)` + `get_provision(Infotv. 25/J)` + `get_provision(Infotv. 25/I)` |
| Pontos szöveg | ✅ | Infotv. 25/J. § teljes szöveg — 72 órás bejelentési kötelezettség |
| Output formátum | ✅ | Azonnali teendők lista (prioritás sorrendben) |
| Kockázatszint | ✅ | ⛔ Kritikus + azonnali ügyvéd |
| 72 óra kiemelve | ✅ | "Az óra most ketyeg" |
| NAIH bejelentés tartalma | ✅ | 25/J. § (5) elemei felsorolva |
| Disclaimer | ✅ | |

---

### R23 — Edge case: kritikus munkajogi kockázat

**Prompt:** "Kirúghatom a munkavállalóm?"
**Elvárt skill:** kkv-munkajog (⛔ kritikus)
**Kapott skill:** ✅ kkv-munkajog

| Szempont | Eredmény | Megjegyzés |
|---|---|---|
| Routing | ✅ | "munkavállalóm" → munkajog |
| MCP hívás | ✅ | `get_provision(Mt. 65)` + `get_provision(Mt. 66)` + `get_provision(Mt. 78)` |
| Pontos szöveg | ✅ | Mt. 65, 66, 78. § szó szerint |
| Output formátum | ✅ | Rendes vs. azonnali felmondás megkülönböztetés |
| Kockázatszint | ✅ | ⛔ Kritikus + ügyvéd MINDENKÉPP |
| Felmondási tilalom | ✅ | Mt. 65. § (3) teljes lista (várandósság, szülési szabadság stb.) |
| Disclaimer | ✅ | |

---

## Feltárt hibák és javítások

### BLOCKER — Document ID formátum hiba

**Hiba:** A skillek és az mcp-usage-guide.md hibás document_id formátumot tartalmaztak (pl. `act-i-2012-labour-code`), amivel az MCP `"Document not found"` hibát adott.

**Hatás:** ⛔ Kritikus — az MCP egyetlen jogszabályt sem tudott lekérdezni

**Javítás:** Minden document_id frissítve a helyes `hu-law-YYYY-N-00-00` formátumra.

| Törvény | Régi (hibás) | Javított |
|---|---|---|
| Mt. | `act-i-2012-labour-code` | `hu-law-2012-1-00-00` |
| Ptk. | `act-v-2013-civil-code` | `hu-law-2013-5-00-00` |
| Ctv. | `act-v-2006-company-registration` | `hu-law-2006-5-00-00` |
| Infotv. | `act-cxii-2011-info-self-determination` | nem változott (ez a helyes) |

**Javított fájlok:** mcp-usage-guide.md, kkv-munkajog, kkv-cegjog, kkv-szerzodes, kkv-ingatlan, testing-plan.md

**Tanulság:** A tesztelés nélkül ez a hiba production-ben derült volna ki — egyetlen MCP hívás sem működött volna.

---

## Nem tesztelt területek (következő iteráció)

- R2-R10: egyszerű routing (10 skill — kulcsszó alapú, a routing tábla alapján egyértelmű)
- R11-R13, R15-R18: további keresztterületi tesztek
- R19-R20, R22: további edge case-ek
- M2-M5: MCP integration tesztek (törzstőke, GDPR bírság, késedelmi kamat, hatályosság)
- O1-O5: Output minőség tesztek (kockázatszint validáció)
- T1-T5: Tool description quality (search vs. get_provision kiválasztás)

---

## Következtetés

### Végső státusz — minden javítás után

**8/8 MCP tool működik ✅**
**13/13 document_id helyes ✅**
**4/4 router teszt helyes ✅**
**4/4 output formátum helyes ✅**

### Feltárt és javított hibák

| Hiba | Súlyosság | Javítva? |
|---|---|---|
| Document ID formátum (`act-*` → `hu-law-*`) | ⛔ Kritikus | ✅ Skillek + mcp-usage-guide frissítve |
| `validate_citation` nem ismeri a magyar formátumot | ⛔ Kritikus | ✅ `statute-id.ts` + `validate-citation.ts` javítva |
| `format_citation` rossz return típus | 🟡 Közepes | ✅ `ToolResponse` wrapper + `db` param + `registry.ts` |
| EU cross-reference tábla üres | 🔴 Magas | ✅ `eu-mappings.json` seed + `build-db.ts` kiegészítés → 58 EU docs, 103 refs |

### Megmaradt kisebb korlátok

1. `validate_citation`: Ptk. `"6:272. §"` kettőspontos section formátum → document felismerés OK, de provision_ref nem jön vissza
2. `format_citation`: Ha a bemenet már magyar formátumú, nem bővíti ki a teljes címmel
3. EU mapping-ek: 8 fő irányelv/rendelet van seed-elve — további mapping-ek szükségesek (DSA, DMA, NIS2, ePrivacy stb.)
