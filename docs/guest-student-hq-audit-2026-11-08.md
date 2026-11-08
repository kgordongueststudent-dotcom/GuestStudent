# Guest Student HQ – átvilágítás és fejlesztési javaslatok

**Projekt:** Collegium Ops Compass / Guest Student HQ (Lovable, `34e90a43-…`)
**Állapot:** commit `2a1c0cf` (2026-11-08 04:59)
**Készítette:** Claude · **Felülvizsgálatra:** ChatGPT (lásd a végén)
**Szabály:** semmi nem lett módosítva vagy törölve. Minden törlés előtt Gordon jóváhagyása kell.

---

## 1. Számítások – pontosság

A kaucióelszámolás képlete helyes: a minta rekordot (DS-R-DEMO-900) kézzel újraszámoltam,
310 000 − 97 925 = **212 075 Ft** egyezik a tárolt eredménnyel. A hibák a széleken vannak:

| # | Probléma | Hatás | Javaslat |
|---|----------|-------|----------|
| S1 | **Pénznem-keverés.** A kaució lehet EUR, a levonások HUF-ban vannak, a rendszer pénznem-ellenőrzés nélkül összeadja őket. | EUR-os kauciónál hibás végösszeg. | Minden tételhez pénznem; eltérésnél kötelező, dátumozott árfolyam. |
| S2 | **Visszatérítés (REFUND) és „PAYMENT PROOF” típus kimarad** a havi befizetési előzményből; csak RENT / UTILITIES / EXTRA / LATE számít. | Pl. a 25 000 Ft-os visszatérítés sehol nem jelenik meg az egyenlegben. | Minden típust be kell sorolni; ismeretlen típus → adatminőségi hiba. |
| S3 | **Ellentmondás nem jelez:** a kulcs „visszaadva” (+20 000 Ft jóváírás), közben „elveszett kulcs pótlása” levonás (6 500 Ft). | Kétes elszámolás véglegesíthető. | Logikai ellenőrzés: ütköző tételeknél „ELLENŐRIZENDŐ”. |
| S4 | **Árfolyam forrás és dátum nélkül** (kézzel beírt 395). | Nem igazolható utólag. | Árfolyam + dátum + forrás (pl. MNB) kötelező mező. |
| S5 | **Negatív összegek nincsenek tiltva** (hátralék, tételek). | Elírás jóváírásként csökkenti a levonást. | Validáció: levonás ≥ 0. |
| S6 | **Dátumok szövegként** tárolva (`payments.date`, `deposits.received_at`, `audit_log.at`); van „ELLENŐRIZENDŐ” érték a dátum oszlopban. | Hibás rendezés, szűrés, időszaki összesítés. | `date` / `timestamptz` típus; hiányzó érték = `NULL`. |
| S7 | **Lejárati napok UTC-ben** (`daysUntil`). Budapesten éjfél és 01:00/02:00 között egy napot téved. | 7/14/30 napos jelzés rossz napon vált. | Budapest szerinti „ma”. |
| S8 | **Kapcsolás névvel, nem azonosítóval:** befizetés ↔ lakó a nevén, szoba ↔ lakó ingatlannév + szobaszám szövegen át. | Névelírás vagy átnevezés → eltűnő befizetés, rossz kihasználtság. | `resident_id`, `room_id` idegen kulcsok. |
| S9 | **Túlfoglalás rejtve:** ha több lakó van egy szobában, mint ágy, a rendszer csendben „megnöveli” az ágyszámot. | Túlfoglalás nem látszik. | Piros jelzés + adatminőségi hiba. |
| S10 | **XLSX export:** az összeg oszlopban szám és „ELLENŐRIZENDŐ” szöveg keveredik. | Excelben nem összegezhető. | Számoszlop külön, státusz külön oszlopban. |

## 2. Struktúra és sebesség

**Ami már jó** (a ma éjjeli módosítások óta): indexek, lapozott listák, táblánkénti lusta betöltés,
a PDF/XLSX könyvtárak csak exportkor töltődnek le.

| # | Javaslat | Haszon |
|---|----------|--------|
| P1 | **Két Excel-könyvtár van** (`xlsx` és `exceljs`). Csak az egyik kell. Az `xlsx` 0.18.5 elavult, ismert biztonsági hibákkal. | Kisebb csomag, biztonság |
| P2 | **DejaVu TTF betűtípusok** (~1,4 MB) minden PDF-exportnál base64-be alakítva. Kisebb, csak latin-ext részhalmaz kell. | Gyorsabb export |
| P3 | **Demó és valós adat keveredik** (pl. „MINTA Diák”, `DEMO-000` szoba élő listákban). | Tisztább számok |
| P4 | **Az audit napló önmagát is naplózza:** a `field_audit` 47 sorából 42 az `audit_log` beszúrása. | Kevesebb zaj, gyorsabb napló |
| P5 | **Egyetlen összesítő lekérdezés a Dashboardra** (darabszámok, jóváhagyások) több külön kérés helyett. | Gyorsabb kezdőoldal |

A 23 menüpont **marad így** (Gordon kérése).

## 3. Nem használt elemek – riport (törlés csak jóváhagyással)

| Elem | Állapot | Javaslat |
|------|---------|----------|
| `resident_notes` tábla | 0 sor, pedig a napló szerint készült jegyzet | Hiba kivizsgálása (lásd 4. pont), nem törlés |
| `resident_reminders` tábla | 0 sor, a napló szerint volt ütemezés | Ugyanaz |
| `xlsx` **vagy** `exceljs` csomag | Dupla | Egyik törölhető |
| Agent chat (HQ, Email, Maintenance, Finance, Booking) | A válaszok **szimuláltak**, nincs valódi AI | Dönteni kell: valódi AI vagy elrejtés |
| Ki nem használt UI-komponensek (pl. carousel, menubar, input-otp, resizable, drawer) | Valószínűleg nem használt – **ellenőrizendő** | Törlés előtt kódkeresés |

## 4. Felhasználói előzmények – ki mit csinált

| Ki | Mit | Mennyi | Mikor |
|----|-----|--------|-------|
| **Gordon Kosinszky** (admin, egyetlen munkatárs) | Kaucióelszámolás mentése | 4 napló + 42 mezőváltozás | 2026-11-08 |
| **„Gordon”** (régi név, ugyanaz a személy) | Kaucióelszámolás mentése | 2 | 2026-09-21 |
| **„Guest Student staff”** (névtelen) | Lakói jegyzet + emlékeztető | 2 | 2026-09-21 |
| maintenance / email / finance / booking agent | Automatikus, szimulált műveletek | 9 | 2026-09-20 |
| Rendszer | Adatminőség-ellenőrzés, jogosultság-betöltés | 6 | 2026-09-20, 11-08 |

**Aktivitás:** jelenleg egyetlen valódi felhasználó van (Gordon). Aktivitás-statisztikához legalább
2 munkatárs és pár hét valós használat kell.

**Problémák a naplóban:**
- **Ugyanaz a személy két néven** („Gordon” / „Gordon Kosinszky”), plusz névtelen „Guest Student staff” → nem mérhető, ki mit csinált. Javaslat: mindig a bejelentkezett felhasználó azonosítója.
- **Gépi és emberi műveletek egy oszlopban** → javaslat: `actor_type` (ember / agent / rendszer).
- **Eltűnt adat:** a napló szerint 09-21-én jegyzet és emlékeztető készült, de a táblák üresek. Vagy a mentés hibázott a napló után, vagy törölve lettek naplózás nélkül. **Ezt ki kell vizsgálni.**
- **Javasolt új oldal: „Munkatársi aktivitás”** – felhasználónként: műveletek száma naponta/hetente, utolsó belépés, mely modulokban dolgozik, nyitott feladatok.

## 5. Javasolt sorrend

1. **S1, S2, S3, S5** – pénzügyi pontosság (legnagyobb kockázat)
2. **4. pont: eltűnt jegyzetek kivizsgálása** + egységes felhasználói azonosító a naplóban
3. **S6, S8** – dátumtípusok és azonosító-alapú kapcsolatok (adatbázis-migráció)
4. **P1–P5** – sebesség és tisztítás
5. **Munkatársi aktivitás oldal**

---

## ChatGPT felülvizsgálat

> ChatGPT: kérlek, nézd át a fenti pontokat. Írd ide, miben értesz egyet, mit látsz másképp,
> és mit hiányolsz. A forráskód a Lovable-projektben van; ha nem éred el, a fenti leírások alapján véleményezz.

_(ide jön a ChatGPT válasza)_
