# Audit & Documentație — `e-factura-addons`

> Documentație tehnică și de business pentru repository-ul **`e-factura-addons`**
> (module Odoo 19.0 pentru e-Factura România / integrarea cu ANAF).
>
> Document scris pentru cititori care **nu cunosc** aceste module. Conține atât
> partea de business (ce problemă rezolvă, pentru cine, cum se folosește), cât și
> partea tehnică (cum funcționează intern, ce extinde, ce API-uri ANAF apelează).

---

## 1. Ce este acest repository

`e-factura-addons` este o colecție de **12 module Odoo** dezvoltate în principal de
**Dakai SOFT** (cu contribuții de la NextERP Romania / OCA) care extind și
îmbunătățesc funcționalitatea de **facturare electronică din România (e-Factura)**
și comunicarea cu sistemul **ANAF SPV** (Spațiul Privat Virtual).

Toate modulele sunt pentru **Odoo 19.0** (vezi câmpul `version` din fiecare
`__manifest__.py`).

### Context de business: ce este e-Factura?

În România, începând cu 2024, transmiterea facturilor în format electronic prin
sistemul **RO e-Factura** al ANAF este **obligatorie** pentru tranzacțiile B2B (și
parțial B2C). Practic:

1. Furnizorul generează factura într-un format XML standardizat (**UBL 2.1 / CIUS-RO**).
2. O **semnează / trimite** către platforma ANAF (SPV) prin API-uri REST.
3. ANAF **validează** factura (sintactic + reguli de business) și răspunde cu un
   identificator de încărcare (`index_incarcare`) și apoi cu o stare
   (`ok` / `nok` / `in prelucrare`).
4. Dacă e validă, ANAF generează o factură semnată (cu semnătura ministerului)
   pe care cumpărătorul o poate descărca.
5. Comunicarea cu ANAF necesită **autentificare OAuth2** cu un certificat
   digital calificat înregistrat în SPV.

Odoo are deja module standard care acoperă fluxul de bază:
- `l10n_ro` / `l10n_ro_config` — localizarea contabilă românească;
- `l10n_ro_edi` — modulul **nativ Odoo 19** pentru e-Factura (abordarea nouă);
- `l10n_ro_account_edi_ubl` — modul OCA mai vechi, bazat pe `account.edi.format`
  și formatul `cius_ro` (abordarea legacy).

**Acest repository nu reinventează fluxul de bază — îl extinde**, acoperind
lipsuri și cazuri reale întâlnite la clienți (erori de validare ANAF afișate prost,
PDF lipsă, atașamente "Anexa", potrivire produse la import, autorizare OAuth
delegată unui furnizor de servicii etc.).

---

## 2. Cele două „generații” tehnice (important!)

Repository-ul conține module care aparțin **a două abordări tehnice diferite**
pentru e-Factura. Este esențial de înțeles pentru a nu le confunda:

| Generație | Modul de bază Odoo | Model central | Câmpuri tipice | Module din acest repo |
|-----------|--------------------|---------------|----------------|------------------------|
| **Legacy (EDI framework)** | `l10n_ro_account_edi_ubl` (OCA) | `account.edi.format` (cod `cius_ro`), `account.edi.document` | `l10n_ro_edi_transaction`, `l10n_ro_edi_download` | `l10n_ro_edi_ubl_anaf_errors`, `l10n_ro_edi_ubl_pdf`, `l10n_ro_edi_ubl_product_invoice_line`, `dakai_inv2po` |
| **Nativ Odoo 19** | `l10n_ro_edi` (Odoo) | `l10n_ro_edi.document`, `account.edi.xml.cius_ro` | `l10n_ro_edi_index`, `l10n_ro_edi_state` | `l10n_ro_efactura_enhancements`, `e_factura_attachments` |

Modulele de **autorizare/licențiere ANAF** (grupul C de mai jos) se conectează la
ambele, în funcție de variantă.

> ⚠️ **Atenție la dependențe încrucișate:** unele module (de ex. cele din grupul
> legacy) extind metode care există în `l10n_ro_account_edi_ubl`, iar cele native
> extind `l10n_ro_edi`. Instalarea trebuie făcută conform stivei folosite de client.

---

## 3. Harta modulelor (3 grupuri funcționale)

### Grup A — Procesare e-Factura / UBL (legacy, pe `account.edi.format`)

| Modul | Rol pe scurt |
|-------|--------------|
| [`l10n_ro_edi_ubl_anaf_errors`](module/l10n_ro_edi_ubl_anaf_errors.md) | Descarcă din ANAF ZIP-ul cu erorile de validare și le afișează în chatter; permite re-trimiterea facturii. |
| [`l10n_ro_edi_ubl_pdf`](module/l10n_ro_edi_ubl_pdf.md) | La importul facturilor primite, generează PDF-ul vizual din XML folosind serviciul web ANAF de „transformare". |
| [`l10n_ro_edi_ubl_product_invoice_line`](module/l10n_ro_edi_ubl_product_invoice_line.md) | Îmbunătățește importul: potrivire produse după codul/numele furnizorului, partener nou, cont bancar IBAN, taxe 0%, denumiri linii. |
| [`dakai_inv2po`](module/dakai_inv2po.md) | Creează o comandă de achiziție (RFQ) direct dintr-o factură furnizor; corelează mișcările de stoc. |

### Grup B — Îmbunătățiri e-Factura nativ (pe `l10n_ro_edi`)

| Modul | Rol pe scurt |
|-------|--------------|
| [`l10n_ro_efactura_enhancements`](module/l10n_ro_efactura_enhancements.md) | Salvează XML-ul pe document, descarcă PDF-ul de la ANAF, configurează nr. de zile de sincronizare, override la fetch facturi/achiziții, reset to draft. |
| [`e_factura_attachments`](module/e_factura_attachments.md) | Atașează PDF-uri „Anexa…" în XML-ul UBL (`AdditionalDocumentReference`) și le îmbină în PDF-ul facturii. |

### Grup C — Autorizare ANAF OAuth & server de licențe

Aici există o **arhitectură client–server** (vezi secțiunea 4):

| Modul | Parte | Rol pe scurt |
|-------|-------|--------------|
| [`partner_anaf_authorize`](module/partner_anaf_authorize.md) | **Client** (legacy) | Obține token-ul ANAF prin „Provider", scriind în câmpurile `l10n_ro_edi_*` ale companiei. |
| [`partner_anaf_authorize_efactura`](module/partner_anaf_authorize_efactura.md) | **Client** (nativ) | Variantă rafinată pentru `l10n_ro_efactura`, cu câmpuri prefixate `l10n_ro_edi_*` și `res.config.settings`. |
| [`partner_authorize`](module/partner_authorize.md) | **Client** (schelet) | Schelet/început de modul generic de autorizare (parțial implementat). |
| [`partner_licence`](module/partner_licence.md) | **Server** | Serverul de licențe: înrolează clienți, generează coduri de licență, împinge date către clienți. |
| [`partner_licence_anaf`](module/partner_licence_anaf.md) | **Server** | Extinde serverul cu fluxul complet OAuth2 ANAF (authorize/token/refresh/revoke) și un cron de reînnoire token-uri. |

### Modul stub

| Modul | Stare |
|-------|-------|
| [`l10n_ro_account_invoice_pdf_fix`](module/l10n_ro_account_invoice_pdf_fix.md) | **Gol** — doar `__manifest__.py`, fără cod. Placeholder. |

---

## 4. Arhitectura „Provider de licențe" (grupul C) — cea mai importantă piesă de business

Aceasta este cea mai interesantă decizie de arhitectură din repo și merită
explicată pentru cineva fără context.

### Problema de business

Pentru a comunica cu ANAF, fiecare companie are nevoie de **credențiale OAuth2**
(`client_id` + `client_secret`) obținute de la ANAF, plus un flux de autorizare cu
certificat digital. Acest lucru este complicat pentru clientul final:
- trebuie să-și înregistreze aplicația OAuth la ANAF;
- trebuie să gestioneze reînnoirea token-urilor (access token valabil scurt,
  refresh token valabil ~3 ani / max. 64 reînnoiri);
- callback-ul OAuth trebuie să fie pe un URL public HTTPS.

### Soluția

Dakai/partenerul rulează un **server central** (instanță Odoo cu modulele
`partner_licence` + `partner_licence_anaf`) care:
1. deține **credențialele OAuth ANAF** (`partner.licence.anaf_oauth`);
2. **înrolează** fiecare client și îi emite un **cod de licență** unic;
3. execută fluxul OAuth cu ANAF **în numele clientului** și apoi **împinge
   token-urile** înapoi către instanța Odoo a clientului.

Instanța clientului rulează modulele **client** (`partner_anaf_authorize` sau
`partner_anaf_authorize_efactura`), care:
1. cer o licență de la server (`/partner-licence/register`);
2. redirecționează utilizatorul către server pentru autorizare ANAF;
3. primesc token-urile prin endpoint-ul `/partner-licence/update-partner-licence`
   și le salvează pe `res.company`.

```
   ┌─────────────────────────┐         ┌──────────────────────────────┐         ┌──────────┐
   │  CLIENT (Odoo client)   │         │  SERVER PROVIDER (Odoo)      │         │  ANAF    │
   │  partner_anaf_authorize │         │  partner_licence +           │         │  OAuth2  │
   │  (_efactura)            │         │  partner_licence_anaf        │         │  SPV     │
   └───────────┬─────────────┘         └──────────────┬───────────────┘         └────┬─────┘
               │  1. register (vat, db, url)          │                              │
               │ ───────────────────────────────────►│                              │
               │            cod licență               │                              │
               │ ◄───────────────────────────────────│                              │
               │  2. redirect anaf-token/<licence>    │                              │
               │ ───────────────────────────────────►│  3. authorize ──────────────►│
               │                                      │  4. ◄──────── code ──────────│
               │                                      │  5. token ──────────────────►│
               │                                      │  6. ◄──── access/refresh ────│
               │  7. push token (update-partner-…)    │                              │
               │ ◄───────────────────────────────────│                              │
               │  (salvează pe res.company)           │  (cron refresh la 1 zi)      │
```

**Avantaj de business:** clientul nu trebuie să gestioneze credențiale ANAF sau
reînnoirea token-urilor — totul este externalizat la partener (model „licență ca
serviciu"). Codul de licență acționează și ca mecanism de
activare/dezactivare comercială (`state` open/closed, `licence_status` blocked).

---

## 5. Glosar de termeni

| Termen | Explicație |
|--------|------------|
| **ANAF** | Agenția Națională de Administrare Fiscală (autoritatea fiscală din România). |
| **SPV** | Spațiul Privat Virtual — platforma ANAF prin care se trimit/primesc documente. |
| **e-Factura (RO e-Factura)** | Sistemul național de facturare electronică obligatoriu. |
| **UBL 2.1 / CIUS-RO** | Formatul XML standard pentru facturi (Universal Business Language, profilul de conformitate românesc). |
| **`index_incarcare` / `id_solicitare`** | ID-ul de încărcare returnat de ANAF după trimiterea unei facturi (pasul 1). |
| **`id_descarcare`** | ID-ul de descărcare al răspunsului semnat / al ZIP-ului cu erori (pasul 2). |
| **`cius_ro`** | Codul formatului EDI românesc în Odoo (`account.edi.format` / `account.edi.xml.cius_ro`). |
| **OAuth2 / token** | Mecanismul de autentificare la ANAF (access token + refresh token). |
| **Provider / Licență** | Serverul partenerului care brokerează autorizarea ANAF; codul de licență identifică un client. |
| **„Anexa"** | Convenție de denumire pentru PDF-urile suplimentare atașate facturii. |
| **`transformare`** | Serviciul web ANAF care convertește XML-ul în PDF vizual. |

---

## 6. Observații generale din audit (calitate cod & riscuri)

Aspecte identificate în timpul auditului, utile pentru mentenanță. Detalii în
fișierele pe module.

- **Două generații coexistă** (legacy `account.edi.format` vs. nativ
  `l10n_ro_edi`). Trebuie ales coerent setul de module pentru fiecare client; nu
  toate sunt compatibile între ele.
- **Cod comentat / depreciat** rămas în surse (ex. `_post_invoice_edi` în
  `l10n_ro_edi_ubl_anaf_errors`, blocuri `# scope = …`). Indicator de migrare în curs.
- **Apeluri HTTP fără tratare robustă** în câteva locuri (ex. `requests.post` fără
  timeout în `partner_licence`, `partner_anaf_authorize`).
- **Endpoint-uri `auth="public"` cu `csrf=False`** pe serverul de licențe — corecte
  funcțional (API server-to-server), dar trebuie protejate prin verificarea
  licenței/`referer` (parțial implementată în `_checkPermission`).
- **`notification_client.start_notification_loop()` conține un `while True` cu
  `time.sleep`** rulat dintr-un cron — potențial blocant (vezi modulul
  `partner_licence`).
- **`l10n_ro_account_invoice_pdf_fix` este gol** — fie de eliminat, fie de completat.
- **Bug minor**: în `partner_licence._send_data_to_customer` comparația este
  `s.state == "close"` în loc de `"closed"` (valoarea reală a selecției).
- **`dakai_inv2po`** scrie `standard_price` pe produs la crearea liniilor PO
  (efect secundar asupra costului produsului) — comportament intenționat dar de
  semnalat.

---

## 7. Cum este organizată documentația

```
e-factura-addons/
├── README.md                      ← acest fișier (overview + business + arhitectură)
└── module/
    ├── l10n_ro_edi_ubl_anaf_errors.md
    ├── l10n_ro_edi_ubl_pdf.md
    ├── l10n_ro_edi_ubl_product_invoice_line.md
    ├── dakai_inv2po.md
    ├── l10n_ro_efactura_enhancements.md
    ├── e_factura_attachments.md
    ├── partner_anaf_authorize.md
    ├── partner_anaf_authorize_efactura.md
    ├── partner_authorize.md
    ├── partner_licence.md
    ├── partner_licence_anaf.md
    └── l10n_ro_account_invoice_pdf_fix.md
```

Fiecare fișier de modul conține: **scop de business**, **dependențe**,
**ce extinde tehnic**, **fluxul detaliat**, **modele/câmpuri noi**,
**endpoint-uri / view-uri** și **observații/riscuri**.
</content>
