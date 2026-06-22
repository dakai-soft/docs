# Ghid de fluxuri de business — Contracte inteligente (Smart Contract)

**Suita de module:** `smart_contract`, `sale_contract`, `purchase_contract`, `contract_overwrite`,
`smart_contract_blanket_order`, `smart_contract_gdpr`, `smart_contract_notification`,
`smart_contract_upsale_downsale`
**Platformă:** Odoo 19.0 · **Autor module:** Dakai SOFT
**Public țintă:** management, responsabili de proces (vânzări, achiziții, juridic, facturare)

> **Cum se folosește acest ghid:** spre deosebire de [`manual_utilizare.md`](manual_utilizare.md)
> (referință de câmpuri/butoane) și [`ghid_procese.md`](ghid_procese.md) (pași concreți, pas cu
> pas), acest document descrie **fluxurile de business end-to-end**: cine acționează, când, de
> ce, ce reguli și ce rezultate de business produce fiecare etapă. Pentru detalii tehnice vezi
> [`ghid_tehnic.md`](ghid_tehnic.md).

---

## Cuprins

1. [Valoarea de business a suitei](#1-valoarea-de-business-a-suitei)
2. [Roluri și responsabilități](#2-roluri-și-responsabilități)
3. [Ciclul de viață al unui document juridic](#3-ciclul-de-viață-al-unui-document-juridic)
4. [Flux: De la ofertă la contract de vânzare semnat](#4-flux-de-la-ofertă-la-contract-de-vânzare-semnat)
5. [Flux: De la comanda de achiziție la abonamente recurente](#5-flux-de-la-comanda-de-achiziție-la-abonamente-recurente)
6. [Flux: Acorduri cadru (Blanket Orders)](#6-flux-acorduri-cadru-blanket-orders)
7. [Flux: Documente conexe (Notificare, GDPR)](#7-flux-documente-conexe-notificare-gdpr)
8. [Flux: Modificarea abonamentului (upsale / downsale)](#8-flux-modificarea-abonamentului-upsale--downsale)
9. [Flux: Rezilierea contractului](#9-flux-rezilierea-contractului)
10. [Flux: Retenția clienților](#10-flux-retenția-clienților)
11. [Reguli de business și puncte de control](#11-reguli-de-business-și-puncte-de-control)
12. [Indicatori și raportare](#12-indicatori-și-raportare)

---

## 1. Valoarea de business a suitei

Suita transformă documentele juridice dintr-un proces manual (Word + e-mail + arhivă) într-un
**proces controlat în Odoo**, integrat cu vânzările, achizițiile și facturarea recurentă:

- **Standardizare** — toate contractele pornesc din șabloane aprobate (capitole/articole),
  reducând riscul juridic și erorile.
- **Date întotdeauna corecte** — variabilele dinamice completează automat clientul, valorile,
  datele și termenii din sistem; nu se mai copiază manual.
- **Trasabilitate** — fiecare document are stare, număr de serie, istoric (chatter) și legături
  către comenzi și abonamente.
- **Continuitate financiară** — contractele de achiziție generează automat abonamente recurente
  OCA; modificările și rezilierile se reflectă imediat în facturare.
- **Retenție** — procesul de retenție documentează negocierile cu clienții care vor să plece și
  generează automat oferta (actul adițional) propusă.

---

## 2. Roluri și responsabilități

| Rol | Responsabilități în flux |
|---|---|
| **Administrator / Juridic** | Configurează seriile de numerotare și șabloanele de contract; definește acțiunile asociate și cheile de date. |
| **Vânzător (Agent)** | Creează contractul din ofertă, îl populează, îl trimite spre semnare; gestionează retenția clienților proprii. |
| **Responsabil achiziții** | Creează contractul de achiziție din comandă; semnarea generează abonamentele recurente. |
| **Facturare / Account Manager** | Vede toate contractele; administrează abonamentele recurente, conversia valutară, închiderea forțată de linii. |
| **Client / Furnizor (extern)** | Primește documentul pe e-mail / pe portal; semnează în afara sistemului; data semnării se înregistrează în Odoo. |

---

## 3. Ciclul de viață al unui document juridic

Toate documentele (contract, act adițional, notificare, GDPR, modificare, reziliere) urmează
aceeași mașină de stări:

```
Draft ─────▶ Prepared ─────▶ Signed ─────▶ Received
(Ciornă)     (Întocmit)      (Semnat)       (Recepționat)
                                  │
                                  └──────────▶ Terminated (Anulat)
```

| Stare | Semnificație de business | Eveniment declanșator |
|---|---|---|
| **Draft** | Document în lucru, complet editabil | creare/`Populate` |
| **Prepared** | Pregătit pentru semnare; se fixează data întocmirii | buton **Ready** |
| **Signed** | Semnat de părți; se alocă numărul oficial; comenzile legate se confirmă | buton **Sign** (necesită *Sign Date*) |
| **Received** | Exemplarul semnat a fost primit înapoi | buton **Received** |
| **Terminated** | Anulat; comenzile legate se anulează | buton **Cancel** |

**Reguli de business cheie ale ciclului:**

- Numărul oficial se alocă **doar** la *Set Number* sau la semnare — un document Draft nu
  consumă serie.
- La semnare/numerotare, **comenzile de vânzare legate** trec automat la confirmate.
- Un **act adițional** se poate semna doar dacă **contractul părinte are deja număr**.

---

## 4. Flux: De la ofertă la contract de vânzare semnat

**Obiectiv de business:** formalizarea unei vânzări printr-un contract corect și trasabil.

```
Ofertă/Comandă vânzare ─▶ Creare contract (Draft) ─▶ Populate din șablon
   ─▶ Parse Variables ─▶ Ready ─▶ Send (semnare client) ─▶ Sign ─▶ (Received)
```

1. **Declanșare (Vânzător).** Pe ofertă/comanda de vânzare se apasă **Creare contract**: apare
   un contract inteligent Draft de tip *Sales*, cu clientul preluat, iar PDF-ul draft se
   atașează comenzii.
2. **Construirea conținutului.** Se alege **Contract Template** și se apasă **Populate** —
   conținutul standard (capitole/articole) și termenii se copiază din șablon.
   - *Regulă (`sale_contract`):* dacă oferta conține **produse de tip contract**, perioada
     (start/end) și moneda contractului se preiau automat din liniile comenzii.
3. **Completarea datelor.** **Parse Variables** înlocuiește variabilele cu datele reale
   (client, CUI, IBAN, valori). Se verifică valorile pe fila *Details* (calculate automat din
   comenzi dacă *Fixed Value* = 0).
4. **Pregătire și trimitere.** **Ready** → documentul devine Prepared; **Send** trimite PDF-ul
   către semnatari (individual sau în masă).
5. **Semnare.** După semnarea fizică/electronică în afara sistemului, se completează **Sign
   Date** și se apasă **Sign**: se alocă numărul oficial și se confirmă comenzile legate.
6. **Recepție (opțional).** La primirea exemplarului semnat — **Received**.

**Rezultat de business:** un contract numerotat, legat de comandă, cu PDF arhivat și valoare
calculată; comenzile sunt confirmate și pot fi facturate.

---

## 5. Flux: De la comanda de achiziție la abonamente recurente

**Obiectiv de business:** formalizarea unui angajament de achiziție recurentă (servicii,
abonamente) și automatizarea facturării prin contracte recurente OCA.

```
Comandă achiziție (linii cu date start/end) ─▶ Creare contract (Draft, Purchase)
   ─▶ Populate + Parse ─▶ Ready ─▶ Sign
        └─▶ generare automată contracte recurente OCA + confirmare comenzi
```

1. **Pregătirea comenzii (Achiziții).** Pe liniile comenzii de achiziție se completează
   **Start Date** / **End Date** pentru produsele de tip contract (perioada de valabilitate).
2. **Crearea contractului.** **Creare contract** produce un contract inteligent de achiziție;
   perioada și moneda se preiau din linii.
3. **Construire și semnare.** Populate → Parse Variables → Ready → Sign.
4. **Automatizare la semnare (`purchase_contract`).** La **Sign**, pentru fiecare comandă
   legată se generează automat **contracte recurente OCA** (`contract.contract`) — câte unul per
   șablon de produs — cu liniile, termenul de plată, poziția fiscală și moneda comenzii; perioada
   se aliniază la contractul inteligent, iar comenzile se confirmă.

**Regulă de business critică:** o comandă de achiziție cu produse de tip contract **nu** poate
fi facturată direct — facturarea se face exclusiv prin abonamentul recurent generat. Acest lucru
previne facturarea dublă și păstrează facturarea recurentă ca unică sursă de adevăr.

**Rezultat de business:** angajamentul de achiziție devine un set de abonamente care facturează
automat, conform perioadei și monedei contractului.

---

## 6. Flux: Acorduri cadru (Blanket Orders)

**Obiectiv de business:** consolidarea sub un singur contract a tuturor comenzilor generate
dintr-un acord cadru cu un furnizor.

1. Pe **acordul de achiziție** (blanket order) se completează câmpul **Smart Contract** cu
   contractul inteligent semnat al furnizorului.
2. Orice comandă de achiziție creată din acord se **atașează automat** contractului inteligent
   (dacă firmele coincid), apărând în fila *Purchase Orders*.

**Rezultat de business:** vizibilitate centralizată asupra tuturor comenzilor dintr-un acord
cadru, fără legare manuală.

---

## 7. Flux: Documente conexe (Notificare, GDPR)

**Obiectiv de business:** emiterea de documente juridice auxiliare legate de un contract, cu
aceeași rigoare (serie, structură, semnare).

- **Notificare** (`smart_contract_notification`) — comunicare juridică formală (ex. somație,
  înștiințare) legată de un contract părinte.
- **GDPR** (`smart_contract_gdpr`) — anexă de prelucrare a datelor pe lângă un contract.

Ambele urmează **același ciclu** (Draft → Prepared → Signed → Received), necesită o **serie
dedicată** (de tip Notificare, respectiv GDPR) și se construiesc din șabloane proprii. Denumirea
se generează automat (ex. „Notification *nr* for *nr contract părinte*”).

**Rezultat de business:** documente conexe numerotate și trasabile, legate de contractul de bază.

---

## 8. Flux: Modificarea abonamentului (upsale / downsale)

**Obiectiv de business:** creșterea (upsale) sau reducerea (downsale) valorii unui abonament
existent, prin act adițional, cu efect imediat în facturarea recurentă.

```
Act „Modificare” ─▶ Incarca Linii (abonamente active) ─▶ ajustare preț/cantitate
   ─▶ verificare indicatori ─▶ Ready ─▶ Sign ─▶ execuție pe liniile de abonament
```

1. Se creează un document de tip **Modificare**, legat de contractul părinte.
2. **Incarca Linii** aduce liniile de abonament active (în curs, viitoare, de reînnoit).
3. Pentru fiecare linie vizată: se bifează **Activ**, se ajustează **prețul de vânzare**,
   **discountul** și/sau **cantitatea**.
4. Indicatorii de decizie: **Diferenta linii curente**, **Rezultat dupa aplicare** și
   **Renuntare inainte de termen** (dacă se renunță înainte de finalul perioadei contractuale).
5. La **Sign**, liniile bifate se execută — noile valori se scriu în abonament; liniile debifate
   se **opresc** la următoarea facturare. Fiecare execuție se consemnează în chatter.

**Rezultat de business:** valoarea recurentă a clientului se modifică controlat și auditabil,
fără intervenție manuală în abonamente.

---

## 9. Flux: Rezilierea contractului

**Obiectiv de business:** încheierea formală a unui contract și oprirea facturării recurente, cu
motiv documentat.

1. Se creează un document de tip **Reziliere**, **obligatoriu** legat de un contract părinte.
2. Se completează **Data Reziliere** și **Motiv Reziliere** (din nomenclator).
3. La **Sign**: toate liniile de abonament active se **opresc** la data rezilierii, contractele
   recurente OCA primesc dată de sfârșit, iar contractul părinte și actele lui trec în
   **Terminated**.

**Rezultat de business:** facturarea recurentă se oprește corect la data rezilierii, cu trasabilitate
asupra motivului — bază pentru analiza churn-ului.

---

## 10. Flux: Retenția clienților

**Obiectiv de business:** gestionarea structurată a clienților care cer rezilierea, maximizând
șansele de a-i păstra și măsurând rezultatele.

```
Cerere reziliere ─▶ Retentie Noua ─▶ negociere (rezoluție + ofertă) ─▶ Finalizare
        ├─▶ Castigat: act adițional generat automat din șablonul de ofertă
        └─▶ Pierdut: (șablon de reziliere) — necesită Data reziliere
```

1. La primirea unei cereri de reziliere, agentul deschide o **Retentie Noua**: la creare se
   preiau automat **Agentul** și **Agentul Suport** ai clientului.
2. Se documentează: contractul, abonamentul, **motivul**, **data primirii cererii** și se
   propune un **șablon de act adițional** (oferta de retenție).
3. În fila **Rezolutie** se notează negocierea.
4. **Finalizare**:
   - dacă există un șablon de ofertă, se creează automat un **act adițional Draft** populat din
     șablon, cu liniile de abonament încărcate și rezoluția în chatter;
   - rezultatul devine **Pierdut** dacă șablonul este unul de reziliere (numele conține
     „rezili” — atunci *Data reziliere* e obligatorie), altfel **Castigat**;
   - se completează data închiderii și se notifică agentul / contractul / fișa clientului.

**Rezultat de business:** fiecare tentativă de plecare este documentată și măsurabilă;
rapoartele de retenție (pe motiv și rezultat) susțin deciziile comerciale.

---

## 11. Reguli de business și puncte de control

| Regulă | De ce contează | Unde apare |
|---|---|---|
| Seria de numerotare este obligatorie | Documentele oficiale trebuie numerotate unitar | la salvarea/semnarea contractului |
| Numărul se alocă doar la *Set Number* / *Sign* | Draft-urile nu consumă serie | tranziția de stare |
| *Sign Date* obligatorie la semnare | Validitatea juridică depinde de data semnării | buton *Sign* |
| Actul adițional cere părinte numerotat | Coerența referințelor între documente | semnarea actului |
| Comenzile cu produse de tip contract nu se facturează direct | Evită dubla facturare; forțează abonamentul | facturarea PO |
| Rezilierea cere contract părinte + motiv + dată | Trasabilitatea churn-ului | semnarea rezilierii |
| `Populate` înlocuiește conținutul existent | Evită amestecul de versiuni | doar pe contracte noi |
| Multi-companie izolează documentele | Confidențialitate și conformitate | reguli de înregistrare |

---

## 12. Indicatori și raportare

- **Valoarea contractului** (`valoare` / `valoare_fara_tva`) — calculată automat din comenzile
  legate (cu conversie valutară) sau din valoarea fixă.
- **Total Opened Lines Amount** (pe abonamentul OCA) — valoarea liniilor deschise, cu filtre
  rapide *Has Opened Lines* / *Zero Opened Lines* pentru urmărirea veniturilor recurente.
- **Indicatorii de modificare** — *Diferenta linii curente* / *Rezultat dupa aplicare* pentru
  impactul financiar al unui upsale/downsale.
- **Rapoarte de retenție** — *Retentii Castigate*, *Retentii Pierdute* și *Raport Retentii*
  (listă/pivot/grafice grupate pe motiv) pentru analiza churn-ului și a eficienței retenției.
- **Variabile nerezolvate** — contorul de variabile semnalează documentele incomplete înainte
  de tipărire/semnare (control de calitate al documentului).

> Pentru pașii operaționali detaliați ai fiecărui flux, consultați [`ghid_procese.md`](ghid_procese.md);
> pentru semnificația fiecărui câmp/buton, [`manual_utilizare.md`](manual_utilizare.md).
