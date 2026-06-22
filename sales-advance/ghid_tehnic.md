# Ghid tehnic — Contracte inteligente (Smart Contract)

**Suita de module:** `smart_contract`, `sale_contract`, `purchase_contract`, `contract_overwrite`,
`smart_contract_blanket_order`, `smart_contract_gdpr`, `smart_contract_notification`,
`smart_contract_upsale_downsale`
**Platformă:** Odoo 19.0 · **Autor module:** Dakai SOFT · **Licență:** OPL-1
**Public țintă:** dezvoltatori, integratori, ingineri de mentenanță

> **Documente conexe:** [`ghid_fluxuri_business.md`](ghid_fluxuri_business.md) (fluxuri de
> business), [`ghid_testare.md`](ghid_testare.md) (testare), [`manual_utilizare.md`](manual_utilizare.md)
> și [`ghid_procese.md`](ghid_procese.md) (utilizare), `DEVELOPER_GUIDE.md` și
> `smart_contract/PARSER_FIXES.md` (parser).

---

## Cuprins

1. [Privire de ansamblu asupra arhitecturii](#1-privire-de-ansamblu-asupra-arhitecturii)
2. [Harta modulelor și dependențele](#2-harta-modulelor-și-dependențele)
3. [Modelul de date al nucleului `smart_contract`](#3-modelul-de-date-al-nucleului-smart_contract)
4. [Parserul declarativ de variabile (`sc-var`)](#4-parserul-declarativ-de-variabile-sc-var)
5. [Mașina de stări și numerotarea](#5-mașina-de-stări-și-numerotarea)
6. [Integrarea cu Vânzări / Achiziții / OCA](#6-integrarea-cu-vânzări--achiziții--oca)
7. [Modulele de extensie](#7-modulele-de-extensie)
8. [Securitate, drepturi și multi-companie](#8-securitate-drepturi-și-multi-companie)
9. [Parametri de configurare](#9-parametri-de-configurare)
10. [Vederi, rapoarte, wizard-uri și assets](#10-vederi-rapoarte-wizard-uri-și-assets)
11. [Puncte de extensie și bune practici](#11-puncte-de-extensie-și-bune-practici)
12. [Referință rapidă: clase și metode cheie](#12-referință-rapidă-clase-și-metode-cheie)

---

## 1. Privire de ansamblu asupra arhitecturii

Suita implementează un sistem de **documente juridice structurate** (contracte, acte
adiționale, notificări, documente GDPR, acte de modificare/reziliere) generate din **șabloane
ierarhice** (capitole → articole → alineate) cu **variabile dinamice** rezolvate din datele
contractului.

Piloni arhitecturali:

- **Nucleul `smart_contract`** — modelele de bază (`smart.contract`, `smart.contract.element`,
  șabloane, serii de numerotare, KV metadata, execuții), parserul de variabile, raportul PDF,
  trimiterea pe e-mail, integrarea cu comenzile.
- **Module de flux** (`sale_contract`, `purchase_contract`, `contract_overwrite`) — leagă
  contractul inteligent de comenzi și de contractele recurente OCA (`contract.contract`).
- **Module de tip document** (`smart_contract_gdpr`, `smart_contract_notification`) — extind
  selecția `document_type`.
- **Module de proces** (`smart_contract_upsale_downsale`, `smart_contract_blanket_order`) —
  modificări de abonament, reziliere, retenție și legarea comenzilor cadru.

Convenții de cod observate în proiect:

- Codul sursă este în limba engleză, dar **etichetele UI sunt mixte ro/en** (există `ro.po`).
- Modelele nucleului stau în `smart_contract/model/` (atenție: director `model/`, nu `models/`).
- Modulele de extensie folosesc directorul standard `models/`.
- Stările folosesc valori în română (`draft`, `intocmit`, `semnat`, `receptionat`, `anulat`).

---

## 2. Harta modulelor și dependențele

| Modul | Versiune | Depinde de | Aplicație | Rol |
|---|---|---|:---:|---|
| `smart_contract` | 19.0.1.0.6 | base, sale, website, purchase, account | da | Nucleul |
| `sale_contract` | 19.0.0.0.0 | contract_overwrite, smart_contract | — | Preia perioada/moneda din comanda de vânzare |
| `purchase_contract` | 19.0.0.0.0 | contract_overwrite, smart_contract | — | Generează contracte recurente OCA la semnare |
| `contract_overwrite` | 19.0.0.0.1 | product_contract (OCA) | — | Ajustări pe `contract.contract` (valută, total linii) |
| `smart_contract_blanket_order` | 19.0.0.0.0 | purchase, smart_contract, purchase_requisition | — | Leagă acordurile cadru |
| `smart_contract_gdpr` | 19.0.0.0.1 | base, smart_contract | — | Tip document GDPR |
| `smart_contract_notification` | 19.0.0.0.1 | base, smart_contract | — | Tip document Notificare |
| `smart_contract_upsale_downsale` | 19.0.0.0.1 | smart_contract, contract (OCA) | — | Modificare/Reziliere/Retenție |

Graf de dependențe (sageata = „depinde de”):

```
                    ┌─ base, sale, purchase, account, website
smart_contract ─────┤
                    └─ (nucleu)
contract_overwrite ── product_contract (OCA)
sale_contract ──────── contract_overwrite, smart_contract
purchase_contract ──── contract_overwrite, smart_contract
smart_contract_gdpr ── base, smart_contract
smart_contract_notification ── base, smart_contract
smart_contract_blanket_order ── purchase, purchase_requisition, smart_contract
smart_contract_upsale_downsale ── smart_contract, contract (OCA)
```

> **Dependență externă cheie:** modulele OCA `product_contract` / `contract` trebuie să fie
> disponibile în `addons-path` (în CI: `/opt/dakai-soft/contract_oca`).

---

## 3. Modelul de date al nucleului `smart_contract`

Fișiere: `smart_contract/model/` — `contract_contract.py`, `contract_template.py`,
`doc_related.py`, `execution.py`, `numbers.py`, `partner.py`, `sale.py`, `purchase.py`,
`template_doc_related.py`, `utils.py`.

### 3.1 `smart.contract` — documentul juridic
Fișier: `contract_contract.py`.

Câmpuri principale (selecție):

| Câmp | Tip | Observații |
|---|---|---|
| `name` | Char (computed, `set_name()`) | Generat din `document_type`, `nr_document`, dată |
| `partner_id` | M2O `res.partner` | Clientul/furnizorul |
| `legal_partner_id` | M2O `res.partner` | Persoana semnatară |
| `company_id` | M2O `res.company` (required) | Multi-companie |
| `type` | Selection | `custommer` / `supplier` / `undefined` |
| `contract_type` | Selection | `sale` / `purchase` / `additional` (determină meniul) |
| `document_type` | Selection | `contract` / `act` (+ extensii: gdpr, notificare, modificare, reziliere) |
| `status` | Selection | `draft → intocmit → semnat → receptionat` / `anulat` |
| `nr_document` | Char (computed) | Alocat din `regulation_id`/`regulation_adition_id` |
| `regulation_id` / `regulation_adition_id` | M2O `smart.contract.regulation` | Seria contractului / a actelor adiționale |
| `parent_id` / `child_ids` | M2O / O2M `smart.contract` | Ierarhia act adițional ↔ părinte |
| `elemente_ids` | O2M `smart.contract.element` | Capitole/articole/alineate |
| `related_data_ids` | O2M `smart.contract.related.data` | KV `attrelation` |
| `related_object_ids` | O2M `smart.contract.related` | Acțiuni asociate |
| `execution_ids` | O2M `smart.contract.execution` | Documente țintă |
| `sale_ids` / `purchase_ids` | O2M | Comenzi legate |
| `valoare_fixa` / `valoare` / `valoare_fara_tva` | Monetary | Valori (computed din comenzi sau fix) |
| `variable_count` / `variable_unresolved_count` / `variable_status_json` | computed | Diagnostic variabile |

Metode de business cheie:

- `populate_with_data()` — instanțiază contractul din șablon (copiază elemente, copii, acțiuni;
  apelează `reNumber()` și `sync_add_placeholders()`).
- `intocmire()` → `draft → intocmit` (setează `data_intocmirii`).
- `setNumber()` — alocă `nr_document` din serie; confirmă comenzile legate.
- `set_signed()` — validează `signed_date`, alocă număr dacă lipsește, trece în `semnat`.
  **Este suprascrisă** de `purchase_contract` și `smart_contract_upsale_downsale`.
- `set_recive()` / `cancel()` — recepție / anulare (anulează comenzile legate).
- `_computeValues()` — calculează valorile cu conversie valutară.
- `ParseVariable()` → `elemente_ids.parseAll()`; `reNumber()` → `elemente_ids.reNumber()`.
- `_compute_variable_status()` / `action_show_unresolved_variables()` — diagnostic variabile.

### 3.2 `smart.contract.element` — unitatea de text
Fișier: `contract_contract.py` (aceeași clasă găzduiește parserul).

- Câmpuri: `name`, `numar` (computed), `tip` (`capitol`/`articol`/`alineat`),
  `text` (Html, **`sanitize_attributes=False`**), `cleartext` (computed, mascat pentru PDF),
  `parent_id`/`child_ids`, `contract_rel_id` (rădăcina O2M), `contract_id` (cascade delete),
  `attribute_ids` (KV `localrelation`), `order`.
- Metode de parser (vezi cap. 4): `_parse()`, `_resolve_var()`, `_resolve_contract()`,
  `_resolve_kv()`, `_format_value()`, `_resolve_status()`, `_sync_add_placeholders()`,
  `_count_sc_var_spans()`, `parseAll()`/`parseRacursive()`, `reNumber()`.

> **`sanitize_attributes=False`** este **critic**: fără el, atributele `data-*` ale span-urilor
> `sc-var` sunt eliminate de sanitizatorul HTML la salvare (vezi `PARSER_FIXES.md`, Fix #1).

### 3.3 Șabloane: `smart.contract.template` și `smart.contract.template.element`
Fișier: `contract_template.py`. Oglindesc structura contractului (`elemente_ids`, ierarhie,
`document_type`, `type`, KV pe șablon) și se instanțiază în contract la `populate_with_data()`.
Au aceeași sintaxă `sc-var` și aceleași generatoare de numerotare (`CapCr`/`ArtCr`/`AlinCr`).

### 3.4 KV metadata: `smart.contract.related.data` (DataSet)
Fișier: `doc_related.py`. Stochează perechile cheie-valoare folosite de variabilele
`attrelation` (la nivel de contract, prin `contract_id`) și `localrelation` (la nivel de
element, prin `element_id`).

- Tipuri: `text` / `float` / `datetime` / `currency`; câmpuri valoare: `text`, `number`
  (Monetary), `number_float`, `number_int` (computed), `data` (Datetime), `data_date` (computed).
- `contract_id` este **scriptabil direct** (nu computed) — necesar pentru fluxul de creare KV
  din span-urile `add`; `create()`/`write()` auto-derivă `contract_id` din `related_id` pentru
  compatibilitate (vezi `PARSER_FIXES.md`, Fix #2).
- `getData(key, domain)` filtrează **întâi după nume** apoi după domeniu (Fix #4);
  `getDatas(keys)` întoarce recordset + dicționar.

### 3.5 Numerotare și execuții
- `smart.contract.regulation` (în `contract_contract.py`) — leagă o `ir.sequence` (`serial_id`)
  de companie și `document_type`; `getNext()` produce numărul.
- `smart.contract.execution` (`execution.py`) — audit al documentelor pe care contractul „se
  execută”; `add()`/`drop()`/`check()` (anti-duplicat).
- `smart.contract.related` (`doc_related.py`) — acțiuni server asociate (manual sau cron
  `RelatedExecute()` pentru cele cu `autoexec`).
- `numbers.py` — clasele `ToRoman`/`ToArabic` pentru numerotarea capitolelor.

---

## 4. Parserul declarativ de variabile (`sc-var`)

### 4.1 Sintaxa
Variabilele sunt marcaje HTML `<span class="sc-var">` cu metadate în atribute `data-*`:

```html
<!-- Pull (doar citire) -->
<span class="sc-var sc-var--pull" data-sens="pull"
      data-name="contract.partner_id.name" data-type="text">[Nume]</span>

<!-- Add (KV editabil, creat automat) -->
<span class="sc-var sc-var--add" data-sens="add"
      data-name="attrelation.garantie" data-type="monetary">[___]</span>
```

> **Sintaxa veche `${...}`** este migrată automat în span-uri prin
> `smart_contract/tools/` (`convert_legacy_text`). Vezi `ghid_testare.md` §7.3.

### 4.2 Spații de nume
- `contract.*` — câmpuri/metode directe pe `smart.contract` (traversare recursivă a relațiilor).
- `attrelation.*` — KV la nivel de contract (`related_data_ids`).
- `localrelation.*` — KV la nivel de element (`attribute_ids`).

### 4.3 Fluxul de rezolvare
```
element.text (HTML cu span-uri sc-var)
  → ContractElement._parse()
     pentru fiecare span:
       _resolve_var(sens, name, vtype)
         ├─ contract.*       → _resolve_contract()  (depth/blocklist/allowlist)
         ├─ attrelation.*    → _resolve_kv('attrelation', ...)
         └─ localrelation.*  → _resolve_kv('localrelation', ...)
       _format_value(value, vtype)   (dată/monedă/număr, după limba userului)
       mutație span:
         succes  → text înlocuit, class `sc-var--resolved`, data-* eliminate
         eșec    → class `sc-var--unresolved`, data-* păstrate (debug)
```

### 4.4 Modelul de securitate al parserului
- **Adâncime maximă** de traversare (`smart_contract.parser.max_depth`, implicit 10).
- **Blocklist de câmpuri** (`field_blocklist`) — împiedică accesul la `env`, `_cr`, `_uid` etc.
- **Allowlist de metode** (`method_allowlist`) — apelurile `contract.metoda()` necesită
  înscriere explicită.
- **Restricție de sens** — sensul `add` este interzis pe `contract.*` (doar KV).
- O variabilă nerezolvabilă **rămâne** marcată `unresolved`; în PDF apare **mascată** (`cleartext`).

Erorile sunt semnalate prin excepția `ParsePathError`
(`odoo.addons.smart_contract.model.contract_contract`).

### 4.5 Sincronizarea placeholder-elor `add`
`_sync_add_placeholders()` scanează `text` după span-uri `data-sens="add"` și creează automat
înregistrări KV (`attrelation` pe contract / `localrelation` pe element), idempotent (sare
cheile existente). Maparea tip HTML → tip KV:

| `data-type` HTML | `type` KV | câmp valoare |
|---|---|---|
| text | text | `text` |
| integer / float | float | `number_float` |
| monetary | currency | `number` |
| date / datetime | datetime | `data` |

### 4.6 Stilizare
`smart_contract/static/src/scss/parser.scss` colorează span-urile: `pull` (albastru),
`add` (verde), `resolved` (transparent), `unresolved` (roșu).

---

## 5. Mașina de stări și numerotarea

### 5.1 Ciclul de viață
```
draft ──intocmire()──▶ intocmit ──set_signed()──▶ semnat ──set_recive()──▶ receptionat
                                       │
                                       └────────── cancel() ──▶ anulat (anulează comenzile)
```
- `set_signed()` cere `signed_date`; alocă `nr_document` dacă lipsește; confirmă comenzile
  Draft/Sent legate.
- Actele adiționale pornesc în `draft` indiferent de starea părintelui; pot fi semnate
  independent (părintele trebuie să aibă deja număr).

### 5.2 Numerotarea documentelor
```
setNumber():
  document_type == 'contract' → nr_document = regulation_id.getNext()
  document_type == 'act'      → regulation_adition_id.getNext()
                                 sau index secvențial în child_ids
```

### 5.3 Numerotarea elementelor
`reNumber()` parcurge elementele în `order` și atribuie `numar` în funcție de `tip`:
capitol → cifre romane (`Cap.I`), articol → arab continuu (`Art.1`), alineat → litere
(`Alin.a`), recursiv prin `child_ids` cu generatoare partajate.

---

## 6. Integrarea cu Vânzări / Achiziții / OCA

### 6.1 Nucleu (`smart_contract`)
- `sale.py` / `purchase.py` adaugă `contract_id` pe `sale.order` / `purchase.order` și metodele
  `sm_action_create_contract()`, `action_open_smart_contract()`, `action_add_attachment()`,
  `action_send_contract_via_message_compose_util()`.
- `utils.py` conține utilitarele partajate (`action_create_contract_util`, etc.).
- `partner.py` adaugă `smart_contract_ids` / `smart_contract_count` și metodele bancare
  `first_bank()` / `first_account()` (folosite în variabile).

### 6.2 `sale_contract`
Suprascrie `sm_action_create_contract()` ca să preia `date_start` (min), `date_end` (max) și
`currency_id` din liniile comenzii cu produse de tip contract.

### 6.3 `purchase_contract`
- Adaugă `date_start`/`date_end` pe `purchase.order.line`.
- Suprascrie `set_signed()`: pentru contractele de achiziție generează, prin
  `action_create_contract()`, contracte recurente OCA (`contract.contract`) — câte unul per
  șablon de produs — apoi confirmă comenzile.
- Blochează facturarea directă a comenzilor cu produse de tip contract.

### 6.4 `contract_overwrite`
Ajustează modulul OCA `contract.contract`:
- `total_opened_lines_amount` (computed, stored) + planificator de recalculare;
- `currency_id` related pe `contract.line`;
- conversie valutară la facturare (`_prepare_invoice_line`) când moneda contractului diferă de
  cea a companiei;
- flag pe `res.company` pentru crearea automată a contractelor la confirmarea comenzii;
- ascunde butoanele OCA standard pe comanda de vânzare.

---

## 7. Modulele de extensie

### 7.1 `smart_contract_gdpr` / `smart_contract_notification`
Pattern identic: extind `document_type` (cu `gdpr`, respectiv `notificare`) pe `smart.contract`,
`smart.contract.regulation` și `smart.contract.template`; suprascriu `_compute_show_regulation()`,
`_get_regulation_required_compute()` (serie obligatorie) și `set_name()` (denumire dedicată).

### 7.2 `smart_contract_blanket_order`
- Adaugă `purchase_requisition_ids` pe `smart.contract` și `smart_contract_id` pe
  `purchase.requisition`.
- Suprascrie `purchase.order.create()`: comenzile create dintr-un acord legat de un contract
  inteligent sunt atașate automat (dacă firmele coincid).
- Hook post-init `_activate_blanket_orders` activează funcționalitatea de acorduri.

### 7.3 `smart_contract_upsale_downsale`
Cel mai amplu modul de proces. Modele:
- `smart.contract` (inherit): `subscription_line_ids`, indicatorii `subs_mod`/`subs_new`/
  `renuntare` (`_newExec`), `document_type` += `modificare`/`reziliere`, câmpuri de reziliere;
  metode `loadSubscription()`, `dropSubscription()`, `_do_reziliere()`, `set_signed()` (execută
  modificările sau rezilierea).
- `smart.contract.line` — linie de modificare a abonamentului; `execute()` aplică schimbările pe
  `contract.line`.
- `smart.contract.retentie` (+ `mail.thread`) — procesul de retenție; `Executa()`/`getState()`
  generează actul adițional rezultat și stabilesc `castigat`/`pierdut`.
- `smart.contract.retentie.category` — nomenclator de motive.
- `contract.line` (inherit) — `SetCancel()`/`SetUnCancel()`, gestionarea reluării la mijloc de
  lună (`retro_date`), `backup_date_end`.
- `res.partner` (inherit) — `support_agent_id`.

---

## 8. Securitate, drepturi și multi-companie

Fișiere: `smart_contract/security/ir.model.access.csv` și `contract_security.xml`.

- **`base.group_user`** — citire/scriere pe contracte, elemente, KV, serii, execuții;
  șabloanele sunt **doar citire**.
- **`account.group_account_manager`** — CRUD complet (inclusiv șabloane); singurul rol care
  vede `Force Close` pe liniile de abonament.
- **`account.group_erp_manager`** — CRUD complet, administrare serii/acțiuni/execuții.

Reguli de înregistrare:
- multi-companie pe `smart.contract`, `smart.contract.template`, `smart.contract.regulation`
  (filtrare după `company_id`);
- „See All Contracts” pentru manageri de vânzări/contabilitate;
- „See Own Contracts” pentru agenți (`user_id` = utilizatorul curent sau gol).

---

## 9. Parametri de configurare

Parametri de sistem (`ir.config_parameter`), prefix `smart_contract.parser.`:

| Parametru | Tip | Implicit | Rol |
|---|---|---|---|
| `smart_contract.parser.max_depth` | Integer | 10 | Adâncimea maximă de traversare pe `contract.*` |
| `smart_contract.parser.field_blocklist` | CSV | (gol) | Câmpuri interzise (ex. `env,_cr,_uid`) |
| `smart_contract.parser.method_allowlist` | CSV | (gol) | Metode permise pentru `contract.metoda()` |

```xml
<record id="param_max_depth" model="ir.config_parameter">
  <field name="key">smart_contract.parser.max_depth</field>
  <field name="value">10</field>
</record>
```

---

## 10. Vederi, rapoarte, wizard-uri și assets

- **Vederi** (`smart_contract/view/`): `contract_contract.xml` (form/tree/kanban + șablon
  portal), `contract_template.xml`, `contract_regulation.xml`, `doc_related.xml`,
  `template_doc_related.xml`, `execution.xml`, `smart_contract_data_views.xml`,
  `smart_contract_template_data_views.xml`, `res_partner_views.xml`, `sale.xml`, `purchase.xml`,
  `contract_report.xml`.
- **Raport PDF**: definit în `view/contract_report.xml` (titlu, structură, mențiune de
  încheiere, blocuri Provider/Client; variabile nerezolvate mascate prin `cleartext`).
- **Wizard-uri** (`smart_contract/wizard/`): `contract.contract.send.wizard` (trimitere
  individuală, `action_send_and_print()`) și `contract.contract.send.batch.wizard` (trimitere
  în masă, `action_send_batch()`, marchează contractele fără e-mail).
- **Date / șabloane e-mail** (`smart_contract/data/`): `data.xml`, `add_attachment.xml`,
  `mail_smart_contract.xml`, `mail_template_data_sale.xml`, `mail_template_data_purchase.xml`.
- **Assets**: `web.assets_backend` → `static/src/scss/parser.scss`.

---

## 11. Puncte de extensie și bune practici

- **Tip nou de document**: urmează pattern-ul GDPR/Notificare — `selection_add` pe
  `smart.contract`, `smart.contract.regulation`, `smart.contract.template`; suprascrie
  `set_name()` și, dacă seria e obligatorie, `_get_regulation_required_compute()`.
- **Logică nouă la semnare**: suprascrie `set_signed()` apelând `super()` (vezi
  `purchase_contract` și `upsale_downsale` ca exemple de compunere multiplă).
- **Variabile / metode noi**: expune metode pe `smart.contract` și înscrie-le în
  `method_allowlist`; nu apela direct ORM-ul în text — folosește spațiile de nume.
- **Nu dezactiva** `sanitize_attributes=False` pe câmpurile `text` — rupe parserul.
- **KV**: folosește `getData()`/`getDatas()` pentru lookup; nu căuta direct prin `related_data_ids`.
- **Numerotare**: nu seta `numar` manual — apelează `reNumber()`.
- Atenție la directorul `model/` (singular) în nucleu vs. `models/` în extensii.

---

## 12. Referință rapidă: clase și metode cheie

| Clasă | Fișier | Metodă | Scop |
|---|---|---|---|
| `smart.contract` | `model/contract_contract.py` | `populate_with_data()` | Instanțiere din șablon |
| | | `set_signed()` | Semnare + finalizare |
| | | `setNumber()` | Alocare număr |
| | | `_computeValues()` | Calcul valoare (cu valută) |
| | | `ParseVariable()` | Rezolvă variabilele |
| `smart.contract.element` | `model/contract_contract.py` | `_parse()` | Rezolvă span-urile `sc-var` |
| | | `_resolve_status()` | Diagnostic (fără mutație) |
| | | `_sync_add_placeholders()` | Creează KV din span-uri `add` |
| | | `reNumber()` | Numerotare automată |
| `smart.contract.template` | `model/contract_template.py` | `reNumber()` | Numerotare șablon |
| `smart.contract.related.data` | `model/doc_related.py` | `getData()` / `getDatas()` | Lookup KV |
| `smart.contract.regulation` | `model/contract_contract.py` | `getNext()` | Următorul număr |
| `smart.contract.execution` | `model/execution.py` | `add()` / `drop()` | Documente țintă |
| `smart.contract.related` | `model/doc_related.py` | `ExecRelated()` / `RelatedExecute()` | Acțiuni server (manual/cron) |
| Send wizard | `wizard/contract_contract_send_wizard.py` | `action_send_and_print()` | Trimitere individuală |
| Batch wizard | `wizard/contract_contract_send_batch_wizard.py` | `action_send_batch()` | Trimitere în masă |
| `smart.contract` (upsale) | `smart_contract_upsale_downsale/models/smart_contract.py` | `loadSubscription()` / `set_signed()` / `_do_reziliere()` | Modificare/Reziliere |
| `smart.contract.line` | `…/models/smart_contract_line.py` | `execute()` | Aplică schimbările pe linie |
| `smart.contract.retentie` | `…/models/smart_contract_retentie.py` | `Executa()` / `getState()` | Proces de retenție |

> **Referințe de cod cu detalii suplimentare:** `DEVELOPER_GUIDE.md` (rezumat parser) și
> `smart_contract/PARSER_FIXES.md` (cele 4 corecții, cu fișiere și linii).
