---
name: ai-jogi-router
description: "AI virtuális jogi csapat orchestrátor: természetes kérdésből meghatározza a területet és a megfelelő skill-t hívja"
mcp_tools:
  - search_legislation
  - build_legal_stance
  - get_provision
  - check_currency
  - validate_eu_compliance
  - get_eu_basis
---

# AI Jogi Router — Orchestrátor

Ez a skill az összes AI jogi skill belépési pontja. Természetes magyar kérdésből azonosítja az érintett jogterületet(eket), meghívja a megfelelő specializált skill(eket), és egységes formátumban adja vissza a választ.

## Indítás

Amikor a felhasználó jogi kérdést tesz fel egy AI-val kapcsolatban, ezt a skill-t kell először futtatni.

---

## Routing logika

### 1. lépés — Terület azonosítása

| Ha a kérdés tartalmaz ilyen kulcsszavakat… | → Hívd ezt a skill-t |
|---|---|
| munkaszerződés, felmondás, szabadság, túlóra, próbaidő, munkaviszony, bér, munkaidő, munkavállaló, home office, távmunka, egyszerűsített foglalkoztatás, KATA-s alvállalkozó | **ai-munkajog** |
| személyes adat, GDPR, cookie, adatkezelés, NAIH, adatvédelem, hozzájárulás, adatszivárgás, incidens, DPA | **ai-gdpr** |
| szerződés, ÁSZF, NDA, titoktartás, szavatosság, késedelem, kártérítés, felelősség, vállalkozási, megbízási, freelancer, SaaS, szoftver fejlesztés | **ai-szerzodes** |
| webshop, fogyasztó, elállás, jótállás, garancia, visszaküldés, online értékesítés, panasz, reklamáció, marketplace, influencer, DSA | **ai-fogyasztovedelmi** |
| cég, Kft., Bt., taggyűlés, törzstőke, ügyvezető, cégjegyzés, alapítás, osztalék, üzletrész, egyéni vállalkozó, EV átalakulás, egyszemélyes | **ai-cegjog** |
| ÁFA, adó, TAO, KIVA, KATA, számla, NAV, adóellenőrzés, bevallás, áfa-kulcs, alanyi mentesség, fordított adózás, iparűzési, HIPA, e-számla | **ai-ado** |
| nem fizet, tartozás, követelés, fizetési meghagyás, FMH, végrehajtás, inkasszó, késedelmi kamat, felszólítás, elévülés, csőd, felszámolás | **ai-koveteleskezeles** |
| védjegy, szerzői jog, szabadalom, IP, szellemi tulajdon, domain, logó, márka, licenc, bitorlás, copyright, open source | **ai-szellemi-tulajdon** |
| székhely, telephely, bérlet, iroda, bérleti szerződés, kaució, bérleti díj, kiürítés, székhelyszolgáltatás, fióktelep | **ai-ingatlan** |
| engedély, működési engedély, bejelentés, telepengedély, NÉBIH, HACCP, vendéglátás, szálláshely, NTAK, Airbnb, kereskedelem bejelentés | **ai-engedelyek** |

### 2. lépés — Keresztterületek kezelése

Ha a kérdés több területet érint, **mindkét skill-t futtasd**:

Tipikus keresztterületek:
- Webshop + adatvédelem → **ai-fogyasztovedelmi** + **ai-gdpr**
- Munkaviszony megszűnés + szerz. → **ai-munkajog** + **ai-szerzodes**
- Cégalapítás + munkáltatói kötelezettségek → **ai-cegjog** + **ai-munkajog**
- Cégalapítás + adónem választás → **ai-cegjog** + **ai-ado**
- Webshop indítás + engedélyek → **ai-engedelyek** + **ai-fogyasztovedelmi** + **ai-gdpr**
- Vendéglátás indítás → **ai-engedelyek** + **ai-ado** + **ai-munkajog**
- Nem fizető partner + szerz. → **ai-koveteleskezeles** + **ai-szerzodes**
- Szoftver fejlesztési szerz. + IP → **ai-szerzodes** + **ai-szellemi-tulajdon**
- Iroda bérlet + székhely → **ai-ingatlan** + **ai-cegjog**
- Freelancer megbízás → **ai-szerzodes** + **ai-munkajog** (KATA elhatárolás)
- Márkaépítés + védjegy → **ai-szellemi-tulajdon** + **ai-cegjog**
- EV → Kft. átmenet → **ai-cegjog** + **ai-ado**

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
| "Hány nap szabadság jár egy 38 éves munkavállalónak?" | ai-munkajog |
| "Kötelező adatkezelési tájékoztató a webshopba?" | ai-gdpr + ai-fogyasztovedelmi |
| "Kirúghatom azonnali hatállyal a munkavállalóm?" | ai-munkajog (⛔ ügyvéd flag) |
| "Mi kötelező egy Kft. alapításához?" | ai-cegjog |
| "Mi a különbség a szavatosság és jótállás között?" | ai-szerzodes vagy ai-fogyasztovedelmi (B2C esetén mindkettő) |
| "Mekkora bírság jár GDPR sértésért?" | ai-gdpr |
| "ÁSZF-be kell írni az elállási jogot?" | ai-fogyasztovedelmi + ai-szerzodes |
| "Melyik adónem éri meg nekem jobban?" | ai-ado |
| "A partnerem nem fizet 3 hónapja, mit tegyek?" | ai-koveteleskezeles |
| "Védjegyeztessem a logómat?" | ai-szellemi-tulajdon |
| "Milyen engedély kell étterem nyitáshoz?" | ai-engedelyek + ai-ado |
| "Székhelyszolgáltatást szeretnék igénybe venni" | ai-ingatlan + ai-cegjog |
| "EV-ből Kft.-t akarok csinálni" | ai-cegjog + ai-ado |
| "Freelancert szerződtetnék szoftverfejlesztésre" | ai-szerzodes + ai-szellemi-tulajdon + ai-munkajog |
| "Airbnb-ztetnék, mi kell hozzá?" | ai-engedelyek + ai-ado |
| "Marketplace-en akarok eladni" | ai-fogyasztovedelmi + ai-ado |
| "Home office-ban dolgoznak a munkavállalóim" | ai-munkajog + ai-gdpr |

---

## Mindig add hozzá a választ végéhez

```
---
⚠️ *Ez a tájékoztatás nem minősül jogi tanácsadásnak. Konkrét jogi döntés előtt kérj ügyvédi véleményt.*
```
