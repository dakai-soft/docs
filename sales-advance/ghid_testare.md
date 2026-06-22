# Ghid de testare manuală — Contracte inteligente (Smart Contract)

**Suita de module:** `smart_contract`, `sale_contract`, `purchase_contract`, `contract_overwrite`,
`smart_contract_blanket_order`, `smart_contract_gdpr`, `smart_contract_notification`,
`smart_contract_upsale_downsale`
**Platformă:** Odoo 19.0 · **Autor module:** Dakai SOFT
**Public țintă:** testeri / QA (testare manuală funcțională și de acceptanță)

> **Cum se folosește acest ghid:** conține **scenarii de test manuale** (cazuri de test) pe
> care un tester le execută în aplicație. Fiecare scenariu are **obiectiv**, **precondiții**,
> **pași** și **rezultat așteptat**. Pentru descrierea câmpurilor/butoanelor consultați
> [`manual_utilizare.md`](manual_utilizare.md), iar pentru pașii operaționali detaliați
> [`ghid_procese.md`](ghid_procese.md).

---

## Cuprins

1. [Cum se folosește acest ghid](#1-cum-se-folosește-acest-ghid)
2. [Precondiții generale și date de test](#2-precondiții-generale-și-date-de-test)
3. [Matricea de acoperire](#3-matricea-de-acoperire)
4. [Configurare (serii și șabloane)](#4-configurare-serii-și-șabloane)
5. [Șabloane: capitole, articole, variabile](#5-șabloane-capitole-articole-variabile)
6. [Contract de vânzare: creare → semnare](#6-contract-de-vânzare-creare--semnare)
7. [Variabile dinamice și raportul PDF](#7-variabile-dinamice-și-raportul-pdf)
8. [Trimiterea pe e-mail](#8-trimiterea-pe-e-mail)
9. [Flux de achiziție → contracte recurente OCA](#9-flux-de-achiziție--contracte-recurente-oca)
10. [Comenzi cadru (Blanket Orders)](#10-comenzi-cadru-blanket-orders)
11. [Notificări și documente GDPR](#11-notificări-și-documente-gdpr)
12. [Modificare abonament (upsale / downsale)](#12-modificare-abonament-upsale--downsale)
13. [Reziliere](#13-reziliere)
14. [Retenția clienților](#14-retenția-clienților)
15. [Drepturi de acces și multi-companie](#15-drepturi-de-acces-și-multi-companie)
16. [Scenarii negative și de validare](#16-scenarii-negative-și-de-validare)
17. [Test de regresie rapidă (smoke test)](#17-test-de-regresie-rapidă-smoke-test)
18. [Șablon de raportare a defectelor](#18-șablon-de-raportare-a-defectelor)

---

## 1. Cum se folosește acest ghid

- Fiecare scenariu are un **cod** (ex. `TC-VANZARE-01`) pentru urmărire.
- Execută pașii **în ordine**; nu sări peste precondiții.
- Pentru fiecare scenariu notează rezultatul: **PASS** / **FAIL** / **BLOCAT**, cu observații.
- La **FAIL**, completează șablonul de defect din cap. 18 (pași de reproducere + capturi).
- **Prioritate**: 🔴 critic (blochează fluxul principal) · 🟡 important · 🟢 secundar.

> **Convenție de denumiri:** etichetele de meniu/buton sunt mixte ro/en (ex. *Sale Contracts*,
> *Creare contract*, *Set Number*, *Incarca Linii*), în funcție de traducerea activă.

---

## 2. Precondiții generale și date de test

Înainte de a începe seria de teste, asigură-te că:

- Toate modulele suitei sunt **instalate** și utilizatorul de test are drepturile necesare
  (vezi cap. 15).
- Există cel puțin **o companie** configurată cu monedă și TVA implicit.
- Sunt disponibile **date de test**:

| Date de test | Recomandare |
|---|---|
| Partener client „Test Client SRL” | cu CUI, IBAN, e-mail și cel puțin un contact persoană |
| Partener furnizor „Test Furnizor SRL” | cu e-mail și contact persoană |
| Produs „Serviciu abonament” | de tip contract (bifa *Is Contract*), cu șablon de contract recurent |
| Produs „Produs simplu” | obișnuit (fără bifa de contract) |
| Secvență de numerotare | una pentru Contract, una pentru Act adițional, una pentru Notificare, una pentru GDPR |
| Server SMTP | configurat (pentru testele de e-mail) sau e-mail în modul „outgoing” verificabil |

> **Sfat:** rulează testele pe o **bază de date de test/staging**, nu pe producție —
> semnarea consumă numere din serie și confirmă comenzi reale.

---

## 3. Matricea de acoperire

| Zonă | Scenarii | Prioritate |
|---|---|---|
| Configurare | TC-CONF-01..02 | 🔴 |
| Șabloane | TC-SABLON-01..03 | 🔴 |
| Contract vânzare | TC-VANZARE-01..06 | 🔴 |
| Variabile & PDF | TC-VAR-01..04 | 🟡 |
| E-mail | TC-MAIL-01..03 | 🟡 |
| Achiziție / OCA | TC-ACHIZ-01..03 | 🔴 |
| Blanket order | TC-CADRU-01 | 🟢 |
| Notificare / GDPR | TC-DOC-01..02 | 🟡 |
| Modificare | TC-MODIF-01..03 | 🔴 |
| Reziliere | TC-REZIL-01..02 | 🔴 |
| Retenție | TC-RETEN-01..02 | 🟡 |
| Drepturi / multi-companie | TC-SEC-01..03 | 🟡 |
| Negative / validări | TC-NEG-01..07 | 🔴 |

---

## 4. Configurare (serii și șabloane)

### TC-CONF-01 — Creare serie de numerotare (Contract Set) 🔴
**Obiectiv:** verifică crearea unei serii valide pentru contracte.
**Precondiții:** utilizator cu drepturi de configurare.
**Pași:**
1. Deschide `Contracts → Configuration → Contract Sets` → **New**.
2. Completează **Set name** = „Contracte vânzare 2026”.
3. La **Serial Sequence** creează/alege o secvență (Prefix + Padding).
4. La **Document type** alege `Contract`. Verifică **Company**.
5. **Save**.

**Rezultat așteptat:** seria se salvează; **Current Sequence** este vizibil (gol sau ultimul
număr); seria apare în listă.

### TC-CONF-02 — Serie fără secvență (validare negativă) 🟡
**Obiectiv:** verifică obligativitatea secvenței.
**Pași:** repetă TC-CONF-01 dar lasă **Serial Sequence** gol și încearcă să salvezi.
**Rezultat așteptat:** apare eroarea *„Please set a serial sequence on your contract set!”*;
înregistrarea nu se salvează.

---

## 5. Șabloane: capitole, articole, variabile

### TC-SABLON-01 — Creare șablon cu capitol și articol 🔴
**Obiectiv:** verifică structura ierarhică a unui șablon.
**Precondiții:** există o serie de tip Contract (TC-CONF-01).
**Pași:**
1. `Contracts → Configuration → Contract Templates` → **New**.
2. Completează **Name** = „Contract prestări servicii”, **Document Type** = `Contract`,
   **Contract Type** = `Custommer`, **Contract Set** = seria creată.
3. În fila **Elements** → **Add a line**: **Type** = `Chapter`, **Name** = „Obiectul
   contractului”.
4. În capitol, fila **Sub Units** → **Add a line**: **Type** = `Article`, **Name** = „Art. 1”.
5. **Save**.

**Rezultat așteptat:** capitolul primește automat **Number** `Cap.I`, articolul `Art.1`;
ierarhia capitol → articol este vizibilă.

### TC-SABLON-02 — Articol cu variabile dinamice 🔴
**Obiectiv:** verifică inserarea variabilelor în text.
**Pași:**
1. Pe articolul din TC-SABLON-01, deschide fila **Content**.
2. Scrie un text care conține: numele clientului, CUI-ul, o valoare din *Related Data* și un
   text local — folosind variabilele din manual (ex. client, CUI, `attrelation.garantie`,
   `localrelation.termen`).
3. Pentru variabila locală, în fila **Attributes** adaugă cheia (Key = „termen”, tip Text, valoare).
4. **Save**.

**Rezultat așteptat:** textul se salvează cu variabilele intacte; cheia locală apare în
**Attributes**.

### TC-SABLON-03 — Renumerotare după reordonare 🟢
**Obiectiv:** verifică recalcularea numerelor.
**Pași:** adaugă un al doilea capitol și mai multe articole, apoi schimbă ordinea prin tragere
și verifică numerotarea (pe contract se poate folosi și butonul **Re-Order**).
**Rezultat așteptat:** capitolele se renumerotează cu cifre romane (`Cap.I`, `Cap.II`),
articolele continuu (`Art.1`, `Art.2`…), alineatele cu litere (`Alin.a`…).

---

## 6. Contract de vânzare: creare → semnare

### TC-VANZARE-01 — Creare contract din comanda de vânzare 🔴
**Obiectiv:** verifică generarea contractului din ofertă.
**Precondiții:** o comandă de vânzare cu „Test Client SRL” și cel puțin o linie.
**Pași:**
1. Deschide comanda de vânzare → apasă **Creare contract** în antet.
2. Apasă butonul statistic **Contracte**.

**Rezultat așteptat:** se creează un contract **Draft** de tip *Sales* cu clientul preluat;
PDF-ul draft este atașat comenzii; butonul statistic **Contracte** apare cu contor 1.

### TC-VANZARE-02 — Populare din șablon 🔴
**Obiectiv:** verifică copierea conținutului din șablon.
**Pași:**
1. Pe contractul Draft, alege **Contract Template** = șablonul din TC-SABLON-01.
2. Completează **Legal Customer Person** (un contact al clientului).
3. Apasă **Populate** → confirmă.
4. Deschide fila **Articles**.

**Rezultat așteptat:** structura de capitole/articole din șablon apare pe contract; compania,
**Document Type**, **Contract Type** și **Contract Set** s-au preluat din șablon.

### TC-VANZARE-03 — Preluare perioadă/monedă din produse de contract 🟡
**Obiectiv:** verifică `sale_contract`.
**Precondiții:** comanda conține „Serviciu abonament” cu **Start/End Date** pe linii.
**Pași:** creează contractul (TC-VANZARE-01) și verifică **Start Date / End Date / Currency** pe
contract.
**Rezultat așteptat:** Start Date = cea mai mică dată de început, End Date = cea mai mare dată de
sfârșit, Currency = moneda comenzii.

### TC-VANZARE-04 — Fluxul de stări Ready → Sign 🔴
**Obiectiv:** verifică tranzițiile de stare și alocarea numărului.
**Pași:**
1. Pe contractul populat (Draft), apasă **Parse Variables** (dacă *Variable Count* > 0).
2. Apasă **Ready** → starea devine **Prepared**; verifică **Act Date** completat automat.
3. Completează **Sign Date** și apasă **Sign**.

**Rezultat așteptat:** starea devine **Signed**; **Document No.** apare în titlu (număr din
serie); comenzile de vânzare legate aflate în Draft/Sent sunt **confirmate automat**.

### TC-VANZARE-05 — Set Number fără semnare 🟡
**Obiectiv:** verifică alocarea numărului fără semnare.
**Pași:** pe un contract **Prepared** fără număr, apasă **Set Number**.
**Rezultat așteptat:** numărul se alocă din serie; comenzile legate Draft/Sent se confirmă;
documentul rămâne Prepared.

### TC-VANZARE-06 — Recepție și anulare 🟡
**Obiectiv:** verifică Received și Cancel.
**Pași:**
1. Pe un contract **Signed**, completează **Reception Date** → apasă **Received**.
2. Pe alt contract Signed, apasă **Cancel**.
**Rezultat așteptat:** primul trece în **Received**; al doilea trece în **Terminated** și
**comenzile de vânzare legate sunt anulate**.

---

## 7. Variabile dinamice și raportul PDF

### TC-VAR-01 — Rezolvarea variabilelor 🟡
**Obiectiv:** verifică înlocuirea variabilelor cu valori reale.
**Pași:** pe un contract populat cu variabile, notează **Variable Count**, apasă
**Parse Variables**, apoi recitește articolele.
**Rezultat așteptat:** variabilele sunt înlocuite cu valorile reale (nume client, CUI, valori);
**Variable Count** scade (ideal la 0).

### TC-VAR-02 — Variabilă nerezolvabilă 🟡
**Obiectiv:** verifică comportamentul la cheie inexistentă.
**Pași:** introdu în text o variabilă cu o cheie KV care **nu** există, salvează, apasă
**Parse Variables**.
**Rezultat așteptat:** variabila rămâne marcată ca nerezolvată (evidențiată); la tipărire
(TC-VAR-04) apare **mascată** (liniuță), nu ca text de cod.

### TC-VAR-03 — Contor variabile nerezolvate 🟢
**Obiectiv:** verifică indicatorul de variabile rămase.
**Pași:** pe un contract cu variabile nerezolvate, folosește acțiunea de afișare a variabilelor
nerezolvate (dacă e disponibilă în UI).
**Rezultat așteptat:** se listează elementele cu variabile nerezolvate.

### TC-VAR-04 — Raportul PDF al contractului 🟡
**Obiectiv:** verifică generarea PDF.
**Pași:** pe un contract semnat, din meniul de tipărire alege raportul **Contract**.
**Rezultat așteptat:** PDF cu titlul „Contract Nr. …”, structura de capitole/articole numerotate,
mențiunea de încheiere cu data semnării și blocurile **Provider** / **Client**; variabilele
nerezolvate apar mascate. Pentru Draft titlul apare ca „DRAFT …”.

---

## 8. Trimiterea pe e-mail

### TC-MAIL-01 — Trimitere individuală 🟡
**Obiectiv:** verifică asistentul *Send*.
**Precondiții:** clientul are e-mail; contract în Draft/Prepared.
**Pași:** apasă **Send** → verifică **To**, **Subject**, corpul mesajului precompletat și
atașamentul PDF → apasă **Send**.
**Rezultat așteptat:** mesajul cu PDF atașat se postează în chatter și se trimite
destinatarilor.

### TC-MAIL-02 — Trimitere în masă 🟢
**Obiectiv:** verifică trimiterea în lot.
**Pași:** în lista de contracte bifează 3 contracte → **Send** din antet → verifică rezumatul
*Contracts to send* → **Send**.
**Rezultat așteptat:** se trimit individual; contractele al căror partener **nu are e-mail**
sunt marcate „(fără email)” și sunt **sărite**.

### TC-MAIL-03 — Partener fără e-mail (validare) 🟢
**Obiectiv:** verifică tratarea lipsei e-mailului.
**Pași:** încearcă trimiterea individuală pentru un contract al cărui partener nu are e-mail.
**Rezultat așteptat:** sistemul semnalează lipsa destinatarului (nu trimite un e-mail gol).

---

## 9. Flux de achiziție → contracte recurente OCA

### TC-ACHIZ-01 — Date pe liniile comenzii de achiziție 🟡
**Obiectiv:** verifică `purchase_contract`.
**Pași:** creează o comandă de achiziție cu „Serviciu abonament”; pe linii completează
**Start Date** și **End Date**.
**Rezultat așteptat:** coloanele Start/End Date apar pe linii și se salvează.

### TC-ACHIZ-02 — Generarea contractelor recurente la semnare 🔴
**Obiectiv:** verifică automatizarea la *Sign*.
**Pași:**
1. Pe comanda de achiziție apasă **Creare contract**.
2. Pe contract: alege șablonul, **Populate**, **Parse Variables**.
3. **Ready** → completează **Sign Date** → **Sign**.
4. Verifică fila **Purchase Orders** și caută contractele recurente OCA (`contract.contract`).

**Rezultat așteptat:** la semnare se generează automat contracte recurente OCA (câte unul per
șablon de produs), cu linii, termen de plată, poziție fiscală și monedă din comandă; perioada se
aliniază la contractul inteligent; comenzile de achiziție se confirmă.

### TC-ACHIZ-03 — Blocarea facturării directe (validare) 🔴
**Obiectiv:** verifică interdicția de facturare directă.
**Pași:** încearcă să creezi factura direct dintr-o comandă de achiziție cu produse de tip
contract.
**Rezultat așteptat:** apare eroarea *„You cannot create an invoice from a purchase order that
contains contract products. Please create a contract instead.”*.

---

## 10. Comenzi cadru (Blanket Orders)

### TC-CADRU-01 — Legarea automată a comenzilor din acord 🟢
**Obiectiv:** verifică `smart_contract_blanket_order`.
**Precondiții:** modul instalat; contract inteligent semnat cu furnizorul.
**Pași:**
1. Pe un acord de achiziție completează câmpul **Smart Contract** cu contractul furnizorului.
2. Creează o comandă de achiziție **din acord**.
3. Deschide contractul, fila **Purchase Orders**.
**Rezultat așteptat:** comanda creată din acord apare automat atașată contractului (dacă firmele
coincid).

---

## 11. Notificări și documente GDPR

### TC-DOC-01 — Emiterea unei notificări 🟡
**Obiectiv:** verifică tipul de document Notificare.
**Precondiții:** serie de tip Notificare + șablon de tip Notificare.
**Pași:**
1. `Contracts → Additional Contracts` → **New** → **Document Type** = `Notificare`.
2. Alege **Contract parent**, **Contract Set** de tip Notificare, șablonul → **Populate**.
3. **Parse Variables** → **Ready** → **Sign Date** → **Sign**.
**Rezultat așteptat:** seria este obligatorie; după numerotare denumirea devine
„Notification *nr* for *nr contract părinte*”; tipărirea cu raportul **Contract** funcționează.

### TC-DOC-02 — Emiterea unui document GDPR 🟡
**Obiectiv:** verifică tipul de document GDPR.
**Pași:** identic cu TC-DOC-01, dar **Document Type** = `GDPR` și serie/șablon de tip GDPR.
**Rezultat așteptat:** seria GDPR este obligatorie; denumirea devine „GDPR *nr* for *nr contract
părinte*”; PDF-ul se generează corect.

---

## 12. Modificare abonament (upsale / downsale)

### TC-MODIF-01 — Încărcarea liniilor de abonament 🔴
**Obiectiv:** verifică *Incarca Linii*.
**Precondiții:** modul `smart_contract_upsale_downsale` instalat; contract părinte cu abonamente
OCA active.
**Pași:**
1. `Contracts → Additional Contracts` → **New** → **Document Type** = `Modificare`.
2. Alege **Contract parent** → fila **Subscription Lines** → grupul Control → **Incarca Linii**.
**Rezultat așteptat:** se încarcă liniile de abonament active (în curs, viitoare, de reînnoit)
ale contractului și actelor sale.

### TC-MODIF-02 — Upsale (creștere preț/cantitate) 🔴
**Obiectiv:** verifică execuția modificării la semnare.
**Pași:**
1. Pe o linie încărcată, bifează **Activ**, mărește **Pret unitar (vanzare)** și/sau
   **Cantitate**.
2. Verifică indicatorii **Diferenta linii curente** și **Rezultat dupa aplicare**.
3. **Ready** → **Sign Date** → **Sign**.
4. Deschide abonamentul OCA și verifică linia.
**Rezultat așteptat:** la semnare, noua cantitate/preț și data următoarei facturi se scriu în
linia de abonament; execuția este consemnată în chatter.

### TC-MODIF-03 — Downsale / oprire de linie 🟡
**Obiectiv:** verifică oprirea unei linii debifate.
**Pași:** pe o linie încărcată **debifează Activ** (sau folosește **Sterge linii**), apoi
semnează.
**Rezultat așteptat:** linia debifată este **oprită** la următoarea facturare; indicatorul
*Renuntare inainte de termen* reflectă corect situația.

---

## 13. Reziliere

### TC-REZIL-01 — Reziliere completă 🔴
**Obiectiv:** verifică încheierea contractului și a abonamentelor.
**Precondiții:** contract părinte cu abonamente active; nomenclator *Motiv Retentie* configurat.
**Pași:**
1. `Contracts → Additional Contracts` → **New** → **Document Type** = `Reziliere`.
2. Alege **Contract parent**, completează **Data Reziliere** și **Motiv Reziliere**.
3. **Ready** → **Sign Date** → **Sign**.
**Rezultat așteptat:** la semnare, toate liniile active se **opresc** la data rezilierii,
contractele recurente OCA primesc dată de sfârșit, iar contractul părinte și actele lui trec în
**Terminated**.

### TC-REZIL-02 — Reziliere fără contract părinte (validare) 🟡
**Obiectiv:** verifică obligativitatea părintelui.
**Pași:** încearcă să creezi/semnezi o reziliere fără **Contract parent**.
**Rezultat așteptat:** apare eroarea *„Un contract de reziliere trebuie sa aiba un contract
parinte!”*.

---

## 14. Retenția clienților

### TC-RETEN-01 — Retenție câștigată (ofertă acceptată) 🟡
**Obiectiv:** verifică generarea actului adițional.
**Precondiții:** nomenclator *Motiv Retentie* configurat; șablon de act adițional „de ofertă”.
**Pași:**
1. `Vânzări → Orders → Retentie → Retentii Noi` → **New**.
2. Alege **Client**, **Contract**, **Abonament**, **Motiv Reziliere**, **Data primire cerere
   reziliere**; la **Sablon Act Aditional Nou** alege șablonul de ofertă (NU unul de reziliere).
3. Scrie textul în fila **Rezolutie** → **Save** → **Finalizare**.
**Rezultat așteptat:** se creează automat un **act adițional Draft** populat din șablon, cu
liniile de abonament încărcate și rezoluția în chatter; câmpul **Act aditional rezultat** este
completat; rezultatul devine **Castigat**; **Agent** și **Agent Suport** au fost preluați automat
la creare.

### TC-RETEN-02 — Retenție pierdută (reziliere) 🟡
**Obiectiv:** verifică marcarea ca pierdut.
**Pași:** repetă TC-RETEN-01 dar alege un șablon al cărui nume conține „rezili” și completează
**Data reziliere**; **Finalizare**.
**Rezultat așteptat:** rezultatul devine **Pierdut**; *Data reziliere* este obligatorie;
înregistrarea apare în *Retentii Pierdute* și în *Raport Retentii*.

---

## 15. Drepturi de acces și multi-companie

### TC-SEC-01 — Vânzător vede doar contractele proprii 🟡
**Obiectiv:** verifică regula „Own Documents”.
**Pași:** autentificat ca vânzător cu drept doar pe documente proprii, deschide lista de
contracte.
**Rezultat așteptat:** vede doar contractele la care este **Responsible** (sau fără responsabil),
nu și ale altora.

### TC-SEC-02 — Șabloanele sunt doar citire pentru utilizatorul standard 🟢
**Obiectiv:** verifică restricția pe șabloane.
**Pași:** ca utilizator intern standard, încearcă să modifici un șablon.
**Rezultat așteptat:** șablonul este accesibil **doar pentru citire** (fără salvare).

### TC-SEC-03 — Izolare multi-companie 🟡
**Obiectiv:** verifică filtrarea pe companie.
**Precondiții:** două companii; documente în fiecare.
**Pași:** cu utilizatorul comutat pe Compania A, deschide listele și câmpurile de legătură
(șablon, serie, acord de achiziție).
**Rezultat așteptat:** se văd doar documentele/seriile/șabloanele Companiei A; câmpurile de
legătură sunt filtrate pe companie.

---

## 16. Scenarii negative și de validare

| Cod | Scenariu | Acțiune | Rezultat așteptat |
|---|---|---|---|
| TC-NEG-01 🔴 | Semnare fără Sign Date | apasă **Sign** fără *Sign Date* | eroare *„Signed date is not set!”* |
| TC-NEG-02 🔴 | Act adițional cu părinte fără număr | adu actul la Prepared când părintele nu are număr | eroare *„Please set a number for parent contract first!”* |
| TC-NEG-03 🔴 | Contract fără serie | salvează un contract de tip Contract fără *Contract Set* | eroare *„Please set a serial sequence on your contract set!”* |
| TC-NEG-04 🟡 | Reziliere fără părinte | vezi TC-REZIL-02 | eroare despre contractul părinte obligatoriu |
| TC-NEG-05 🟡 | Facturare directă PO cu produse de contract | vezi TC-ACHIZ-03 | eroare de blocare a facturării |
| TC-NEG-06 🟢 | Populate pe contract cu conținut | apasă **Populate** pe un contract cu articole existente | conținutul existent este **înlocuit** (avertizare: nu folosi pe contracte cu modificări manuale) |
| TC-NEG-07 🟢 | Reziliere fără Motiv/Data | semnează reziliere fără *Motiv Reziliere* sau *Data Reziliere* | sistemul cere completarea câmpurilor obligatorii |

---

## 17. Test de regresie rapidă (smoke test)

Set minim de verificat după fiecare actualizare/deploy (≈ 15 minute):

1. **TC-CONF-01** — se poate crea o serie.
2. **TC-SABLON-01** — se poate crea un șablon cu capitol + articol.
3. **TC-VANZARE-01 → 04** — creare contract din comandă, Populate, Parse, Ready, Sign (cu număr).
4. **TC-VAR-04** — PDF-ul contractului semnat se generează corect.
5. **TC-MAIL-01** — trimiterea pe e-mail funcționează.
6. **TC-ACHIZ-02** — semnarea unui contract de achiziție generează abonamente OCA.
7. **TC-MODIF-02** — un upsale se aplică pe abonament la semnare.
8. **TC-REZIL-01** — o reziliere oprește abonamentele și trece părintele în Terminated.

Dacă oricare eșuează, oprește promovarea și raportează defectul (cap. 18).

---

## 18. Șablon de raportare a defectelor

```
[ID defect]      DEF-____
[Scenariu]       TC-________ (codul cazului de test)
[Prioritate]     Critic / Important / Secundar
[Mediu]          Versiune Odoo / bază de date / companie / utilizator(rol)
[Precondiții]    starea inițială a datelor
[Pași de reproducere]
  1.
  2.
  3.
[Rezultat actual]      ce s-a întâmplat (cu textul exact al erorii, dacă există)
[Rezultat așteptat]    ce trebuia să se întâmple
[Atașamente]           capturi de ecran / PDF / log
[Observații]           reproductibil mereu / intermitent; soluție de ocolire
```

> **Referințe:** [`manual_utilizare.md`](manual_utilizare.md) (câmpuri și butoane),
> [`ghid_procese.md`](ghid_procese.md) (pași operaționali), [`ghid_fluxuri_business.md`](ghid_fluxuri_business.md)
> (context de business), [`ghid_tehnic.md`](ghid_tehnic.md) (detalii tehnice).
