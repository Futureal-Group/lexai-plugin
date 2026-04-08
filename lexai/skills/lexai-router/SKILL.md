---
name: lexai-router
description: "LexAI jogi asszisztens orchestrátor: természetes kérdésből meghatározza a területet és a megfelelő skill-t hívja"
mcp_tools:
  - search_legislation
  - build_legal_stance
  - get_provision
  - check_currency
  - validate_eu_compliance
  - get_eu_basis
---

# LexAI Router — Orchestrátor

Ez a skill az összes AI jogi skill belépési pontja. Természetes magyar kérdésből azonosítja az érintett jogterületet(eket), meghívja a megfelelő specializált skill(eket), és egységes formátumban adja vissza a választ.

## Indítás

Amikor a felhasználó jogi kérdést tesz fel egy vállalkozással vagy jogi kérdéssel kapcsolatban, ezt a skill-t kell először futtatni.

---

## Routing logika

### 1. lépés — Terület azonosítása

| Ha a kérdés tartalmaz ilyen kulcsszavakat… | → Hívd ezt a skill-t |
|---|---|
| munkaszerződés, felmondás, szabadság, túlóra, próbaidő, munkaviszony, bér, munkaidő, munkavállaló, home office, távmunka, egyszerűsített foglalkoztatás, KATA-s alvállalkozó | **lexai-munkajog** |
| személyes adat, GDPR, cookie, adatkezelés, NAIH, adatvédelem, hozzájárulás, adatszivárgás, incidens, DPA | **lexai-gdpr** |
| szerződés, ÁSZF, NDA, titoktartás, szavatosság, késedelem, kártérítés, felelősség, vállalkozási, megbízási, freelancer, SaaS, szoftver fejlesztés | **lexai-szerzodes** |
| webshop, fogyasztó, elállás, jótállás, garancia, visszaküldés, online értékesítés, panasz, reklamáció, marketplace, influencer, DSA | **lexai-fogyasztovedelmi** |
| cég, Kft., Bt., taggyűlés, törzstőke, ügyvezető, cégjegyzés, alapítás, osztalék, üzletrész, egyéni vállalkozó, EV átalakulás, egyszemélyes | **lexai-cegjog** |
| ÁFA, adó, TAO, KIVA, KATA, számla, NAV, adóellenőrzés, bevallás, áfa-kulcs, alanyi mentesség, fordított adózás, iparűzési, HIPA, e-számla | **lexai-ado** |
| nem fizet, tartozás, követelés, fizetési meghagyás, FMH, végrehajtás, inkasszó, késedelmi kamat, felszólítás, elévülés, csőd, felszámolás | **lexai-koveteleskezeles** |
| védjegy, szerzői jog, szabadalom, IP, szellemi tulajdon, domain, logó, márka, licenc, bitorlás, copyright, open source | **lexai-szellemi-tulajdon** |
| székhely, telephely, bérlet, iroda, bérleti szerződés, kaució, bérleti díj, kiürítés, székhelyszolgáltatás, fióktelep | **lexai-ingatlan** |
| engedély, működési engedély, bejelentés, telepengedély, NÉBIH, HACCP, vendéglátás, szálláshely, NTAK, Airbnb, kereskedelem bejelentés | **lexai-engedelyek** |

### 2. lépés — Keresztterületek kezelése

Ha a kérdés több területet érint, **mindkét skill-t futtasd**:

Tipikus keresztterületek:
- Webshop + adatvédelem → **lexai-fogyasztovedelmi** + **lexai-gdpr**
- Munkaviszony megszűnés + szerz. → **lexai-munkajog** + **lexai-szerzodes**
- Cégalapítás + munkáltatói kötelezettségek → **lexai-cegjog** + **lexai-munkajog**
- Cégalapítás + adónem választás → **lexai-cegjog** + **lexai-ado**
- Webshop indítás + engedélyek → **lexai-engedelyek** + **lexai-fogyasztovedelmi** + **lexai-gdpr**
- Vendéglátás indítás → **lexai-engedelyek** + **lexai-ado** + **lexai-munkajog**
- Nem fizető partner + szerz. → **lexai-koveteleskezeles** + **lexai-szerzodes**
- Szoftver fejlesztési szerz. + IP → **lexai-szerzodes** + **lexai-szellemi-tulajdon**
- Iroda bérlet + székhely → **lexai-ingatlan** + **lexai-cegjog**
- Freelancer megbízás → **lexai-szerzodes** + **lexai-munkajog** (KATA elhatárolás)
- Márkaépítés + védjegy → **lexai-szellemi-tulajdon** + **lexai-cegjog**
- EV → Kft. átmenet → **lexai-cegjog** + **lexai-ado**

### 3. lépés — Ha a terület nem egyértelmű

Futtasd ezt:
```
build_legal_stance(
  query: "[a felhasználó kérdése angolul/magyarul]",
  limit: 5
)
```
A találatok alapján azonosítható, melyik törvény a releváns.

---

## Output formátum

Minden válasz a `references/output-template.md` alapján épül fel.

**Alap:** rövid válasz (kockázatszint + teendők)
**Opcionális:** ha a felhasználó kéri, vagy a kockázat 🔴/⛔ → részletes jogi háttér automatikusan

Kockázatszint logika: `references/risk-levels.md`

---

## Példa kérdések → routing

| Kérdés | Router döntés |
|---|---|
| "Hány nap szabadság jár egy 38 éves munkavállalónak?" | lexai-munkajog |
| "Kötelező adatkezelési tájékoztató a webshopba?" | lexai-gdpr + lexai-fogyasztovedelmi |
| "Kirúghatom azonnali hatállyal a munkavállalóm?" | lexai-munkajog (⛔ ügyvéd flag) |
| "Mi kötelező egy Kft. alapításához?" | lexai-cegjog |
| "Mi a különbség a szavatosság és jótállás között?" | lexai-szerzodes vagy lexai-fogyasztovedelmi (B2C esetén mindkettő) |
| "Mekkora bírság jár GDPR sértésért?" | lexai-gdpr |
| "ÁSZF-be kell írni az elállási jogot?" | lexai-fogyasztovedelmi + lexai-szerzodes |
| "Melyik adónem éri meg nekem jobban?" | lexai-ado |
| "A partnerem nem fizet 3 hónapja, mit tegyek?" | lexai-koveteleskezeles |
| "Védjegyeztessem a logómat?" | lexai-szellemi-tulajdon |
| "Milyen engedély kell étterem nyitáshoz?" | lexai-engedelyek + lexai-ado |
| "Székhelyszolgáltatást szeretnék igénybe venni" | lexai-ingatlan + lexai-cegjog |
| "EV-ből Kft.-t akarok csinálni" | lexai-cegjog + lexai-ado |
| "Freelancert szerződtetnék szoftverfejlesztésre" | lexai-szerzodes + lexai-szellemi-tulajdon + lexai-munkajog |
| "Airbnb-ztetnék, mi kell hozzá?" | lexai-engedelyek + lexai-ado |
| "Marketplace-en akarok eladni" | lexai-fogyasztovedelmi + lexai-ado |
| "Home office-ban dolgoznak a munkavállalóim" | lexai-munkajog + lexai-gdpr |

---

## Mindig add hozzá a választ végéhez

```
---
⚠️ *Ez a tájékoztatás nem minősül jogi tanácsadásnak. Konkrét jogi döntés előtt kérj ügyvédi véleményt.*
```
