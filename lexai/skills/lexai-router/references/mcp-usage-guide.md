# MCP Használati Útmutató — Hungarian Law MCP Server

Ez a referencia fájl dokumentálja a `lexmind-hu` MCP server tool-jainak helyes használatát az összes LexAI skill számára.

---

## Alapelv: MCP-first

> **A pontos jogszabályszöveget, határértékeket és összegeket MINDIG az MCP tool-okból kérd le.**
> A skill-ekben lévő számok, összegek és jogszabályi hivatkozások csak **routing hintek** — nem autoritatív források.
> A jogszabályok változhatnak. Az MCP adatbázis naponta frissül.

---

## Kötelező MCP hívási sorrend

Minden válasz előtt kövesd ezt a sorrendet:

```
1. search_legislation(query: "[téma]", limit: 8)
   → Releváns szakaszok keresése

2. get_provision(document_id: "[törvény]", section: "[§]")
   → Pontos, szó szerinti jogszabályszöveg lekérése

3. check_currency(document_id: "[törvény]")
   → Hatályosság ellenőrzése (in_force / amended / repealed)

4. validate_citation(citation: "[hivatkozás]")
   → Hivatkozás validálása a válasz előtt
```

EU-releváns kérdéseknél kiegészítve:
```
5. get_eu_basis(document_id: "[magyar törvény]", include_articles: true)
   → EU irányelv/rendelet alap

6. validate_eu_compliance(document_id: "[magyar törvény]")
   → EU megfelelőség ellenőrzése
```

---

## Tool-ok részletes leírása

### `search_legislation` — Jogszabály keresés
**Input:** `query` (FTS5 keresés), `document_id?`, `status?`, `limit?`
**Output:** Snippet-ek BM25 relevanciával, `>>>` és `<<<` jelölőkkel
**Mikor:** Amikor nem tudod pontosan melyik § a releváns

### `get_provision` — Pontos jogszabályszöveg
**Input:** `document_id`, `section?`, `provision_ref?`
**Output:** **Teljes, szó szerinti jogszabályszöveg** (nem snippet, nem összefoglalás)
**Mikor:** Amikor ismered a törvényt és a §-t — ez adja a hiteles szöveget

### `check_currency` — Hatályosság ellenőrzés
**Input:** `document_id`, `provision_ref?`
**Output:** `status` (in_force/amended/repealed), `issued_date`, `in_force_date`, `warnings[]`
**Mikor:** MINDIG, mielőtt egy törvényre hivatkozol

### `validate_citation` — Hivatkozás validálás
**Input:** `citation` (pl. "Mt. 115. §", "2012. évi I. tv. 115. §")
**Output:** `valid` (boolean), `status`, `warnings[]`
**Mikor:** MINDEN hivatkozás előtt — ez a zero-hallucination garancia

### `build_legal_stance` — Átfogó jogi kutatás
**Input:** `query`, `document_id?`, `limit?`
**Output:** Aggregált találatok több törvényből
**Mikor:** Komplex kérdéseknél, ahol nem tudod melyik törvény releváns

### `format_citation` — Hivatkozás formázás
**Input:** `citation`, `format?` ('full'/'short'/'pinpoint')
**Output:** Formázott hivatkozás
**Mikor:** Egységes hivatkozás formátum biztosítása az output-ban

### `get_eu_basis` — EU alap
**Input:** `document_id`, `include_articles?`
**Output:** EU irányelvek/rendeletek, cikk hivatkozásokkal
**Mikor:** Magyar törvény EU háttere

### `get_provision_eu_basis` — § szintű EU alap
**Input:** `document_id`, `provision_ref`
**Output:** Pontosan melyik EU cikk(ek)hez kapcsolódik az adott §
**Mikor:** Részletes EU megfelelőség

### `validate_eu_compliance` — EU megfelelőség
**Input:** `document_id`, `provision_ref?`, `eu_document_id?`
**Output:** `compliance_status`, `warnings[]`, `recommendations[]`
**Mikor:** EU transzpozíció ellenőrzése

---

## Gyakori document_id-k

| Rövidítés | Teljes név | document_id |
|---|---|---|
| Mt. | Munka Törvénykönyve (2012. évi I. tv.) | `hu-law-2012-1-00-00` |
| Ptk. | Polgári Törvénykönyv (2013. évi V. tv.) | `hu-law-2013-5-00-00` |
| Ctv. | Cégtörvény (2006. évi V. tv.) | `hu-law-2006-5-00-00` |
| Infotv. | Info törvény (2011. évi CXII. tv.) | `act-cxii-2011-info-self-determination` |
| ÁFA tv. | ÁFA törvény (2007. évi CXXVII. tv.) | `hu-law-2007-127-00-00` |
| TAO tv. | Társasági adó tv. (1996. évi LXXXI. tv.) | `hu-law-1996-81-00-00` |
| Art. | Adózás rendje (2017. évi CL. tv.) | `hu-law-2017-150-00-00` |
| Fgytv. | Fogyasztóvédelmi tv. (1997. évi CLV. tv.) | `hu-law-1997-155-00-00` |
| Szjt. | Szerzői jogi tv. (1999. évi LXXVI. tv.) | `hu-law-1999-76-00-00` |
| Vt. | Védjegytörvény (1997. évi XI. tv.) | `hu-law-1997-11-00-00` |
| Fmhtv. | Fizetési meghagyás tv. (2009. évi L. tv.) | `hu-law-2009-50-00-00` |
| Vht. | Végrehajtási tv. (1994. évi LIII. tv.) | `hu-law-1994-53-00-00` |
| GDPR | EU 2016/679 rendelet | `regulation:2016/679` |

**Ha nem ismered a document_id-t:** használd a `search_legislation()` tool-t a törvény keresésére, és az eredményből olvasd ki a `document_id`-t.

---

## Amit SOHA ne csinálj

- ❌ Ne idézz jogszabályszöveget a skill tartalma alapján — mindig `get_provision()`
- ❌ Ne adj meg pontos összegeket/határértékeket anélkül, hogy az MCP-vel ellenőriznéd
- ❌ Ne feltételezd, hogy egy törvény hatályos — mindig `check_currency()`
- ❌ Ne használj hivatkozást validálás nélkül — mindig `validate_citation()`
