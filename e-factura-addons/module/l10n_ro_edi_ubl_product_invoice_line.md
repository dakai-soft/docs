# `l10n_ro_edi_ubl_product_invoice_line` — Import îmbunătățit linii/produse

**Nume afișat:** *E-factura Product Names*
**Versiune:** 19.0.0.0 · **Licență:** OPL-1 · **Autor:** Dakai SOFT SRL · **Mentenanță:** adrian-dks
**Categorie:** Accounting · **Generație:** Legacy (`account.edi.xml.cius_ro`)
**Depinde de:** `l10n_ro_account_edi_ubl`

---

## 1. Scop de business

La **importul facturilor furnizor** primite prin e-Factura, modulul standard
lasă multe câmpuri necompletate (produs necunoscut, cont bancar lipsă, taxe greșite,
denumiri de linii sărace). Acest modul **îmbogățește importul** astfel încât factura
primită să fie cât mai completă și „gata de validat", reducând munca manuală:

- **potrivește automat produsul** după codul sau numele folosit de furnizor
  (`seller_ids` — lista de furnizori de pe produs);
- **creează partenerul** dacă nu există (cu VAT, țară, email, telefon) și rulează
  validarea VAT românească;
- **completează contul bancar (IBAN)** al furnizorului din XML, creându-l dacă lipsește;
- aplică o **taxă 0%** când categoria de TVA din XML este `O`/`E`/`Z` (neimpozabil/scutit/zero);
- compune **denumirea liniei** din `Description` + `Name`;
- setează **contul de stoc** corect pentru produsele cu evaluare în timp real.

Beneficiu: facturile de la furnizori recurenți se importă aproape complet automat.

---

## 2. Ce extinde tehnic

### `account.edi.xml.cius_ro` (`models/account_edi_xml_cius_ro.py`)

- **`_l10n_ro_search_iban(iban, bank, partner)`** — caută/creează `res.partner.bank`
  pentru IBAN-ul din XML; leagă banca (`res.bank`) după nume; marchează
  `l10n_ro_print_report=True`.
- **`_l10n_ro_import_retrieve_partner_bank(tree, invoice)`** — extrage IBAN-ul din
  `PaymentMeans/PayeeFinancialAccount` și îl pune pe `partner_bank_id`.
- **`_import_fill_invoice_form(...)`** (override) — apelează logica de cont bancar
  doar pentru jurnale de tip `purchase`.
- **`_import_retrieve_and_fill_partner(...)`** (override) — dacă nu s-a găsit partener,
  **creează** unul nou (`is_company=True`, nume/VAT/email/telefon/țară) și apelează
  `ro_vat_change()` pentru normalizarea VAT-ului românesc.
- **`_import_fill_invoice_line_form(...)`** (override) — dacă există o singură
  categorie de taxă cu cod `O`/`E`/`Z`, aplică prima taxă 0% potrivită tipului jurnalului.
- **`_import_fill_invoice_line_values(...)`** (override) — compune `name` din
  `Description - Name`; dacă linia n-are produs (jurnal `purchase`), încearcă să
  potrivească produsul prin harta de import.
- **`_l10n_ro_line_product(invoice_line, product)`** — setează produsul pe linie și,
  dacă produsul are evaluare `real_time`, setează contul `stock_valuation`.
- **`_import_retrieve_product_map(company)`** — întoarce o **hartă de strategii** de
  potrivire ordonate pe priorități:
  - `10` → potrivire după **codul** furnizorului (`seller_ids.product_code`);
  - `30` → potrivire după **numele** furnizorului (`seller_ids.product_name`);
  - filtrate pe `company_id` și `partner_id`.
- **`_import_retrieve_info_from_map(...)`** — iterează strategiile în ordinea cheilor
  și întoarce primul produs găsit.

### `account.edi.format` (`models/account_edi_format.py`)
- **`_find_value(...)`** (override) — încearcă mai întâi metoda standard; dacă aruncă
  excepție, **reîncearcă cu un set explicit de namespace-uri UBL** (cac, cbc, qdt,
  udt, ccts, xsi). Robustețe pentru XML-uri cu namespace-uri diferite.

### `account.move.line` (`models/account_move_line.py`)
- **`_compute_price_unit`** și **`_compute_name`** (override) — pentru facturile
  provenite din ANAF (`move_id.l10n_ro_edi_download` setat), **păstrează** prețul și
  denumirea importate, împiedicând recalcularea automată Odoo să le suprascrie.

---

## 3. Observații / riscuri
- **Bug probabil** în `_import_retrieve_and_fill_partner`: `new_partner` primește un
  `id` (int), apoi se apelează `new_partner.ro_vat_change()` pe un int — ar arunca
  eroare. De verificat.
- Căutarea taxei folosește `('amount','=','0')` (string) — depinde de normalizarea
  ORM; de preferat `0` (numeric).
- Strategiile de potrivire produs combină `Name`/`Description`/`SellersItemIdentification`
  într-un singur `in` — pot apărea potriviri false dacă datele furnizorului se suprapun.
- Fără view-uri / fără ACL (CSV gol).
</content>
