# `l10n_ro_efactura_enhancements` — Îmbunătățiri e-Factura (nativ)

**Nume afișat:** *Romania - E-factura Enhancements*
**Versiune:** 19.0.1.0.1 · **Licență:** AGPL-3 · **Autor:** DakaiSoft, OCA · **Mentenanță:** Flavia0320
**Categorie:** Localization · **Status:** Beta · **Generație:** Nativ Odoo 19 (`l10n_ro_edi`)
**Depinde de:** `base`, `l10n_ro_config`, `l10n_ro_edi`

---

## 1. Scop de business

Extinde modulul nativ Odoo 19 `l10n_ro_edi` cu funcționalități practice cerute de
clienți la sincronizarea cu ANAF:

- **salvează XML-ul** efectiv al fiecărei facturi pe documentul EDI (pentru
  consultare/arhivare ulterioară);
- permite **descărcarea PDF-ului** generat de ANAF din XML, direct dintr-un buton;
- face **configurabil numărul de zile** pe care le acoperă sincronizarea facturilor
  primite (0–60 zile);
- **rescrie fluxurile de fetch** pentru facturi trimise și facturi/achiziții primite,
  astfel încât XML-ul răspunsului ANAF să fie reținut;
- permite **resetarea la „draft"** a facturilor de intrare descărcate din SPV.

Beneficiu: vizibilitate completă asupra XML-ului și PDF-ului oficial, plus control
asupra ferestrei de sincronizare.

---

## 2. Ce extinde tehnic

### `l10n_ro_edi.document` (`models/l10n_ro_edi_document.py`)
- Adaugă câmpul **`xml_attachment`** (Binary, readonly) — stochează XML-ul facturii.
- **`action_l10n_ro_edi_download_attachment()`** — decodează XML-ul, detectează dacă
  e notă de credit (`<CreditNote` → `FCN`, altfel `FACT1`) și apelează serviciul ANAF
  **`/transformare`** pentru a obține PDF-ul:
  - tratează cazul `"stare":"nok"` (eroare ANAF) → `UserError` cu mesajul;
  - la succes, creează un `ir.attachment` PDF numit `<index>.pdf` și întoarce o
    acțiune de tip URL pentru descărcare imediată.

### `account.move` (`models/account_move.py`)
- **`_l10n_ro_edi_process_bill_messages(...)`** (override) — la procesarea facturilor
  **primite** (bills), salvează XML-ul răspunsului pe documentele EDI ale facturii
  corelate prin `l10n_ro_edi_index`.
- **`_l10n_ro_edi_fetch_invoice_sent_documents()`** (override complet) — pentru
  facturile **trimise** (`invoice_sent`): face fetch status la SPV; dacă nu e gata,
  postează mesaj; dacă e respinsă (`nok`) creează document `invoice_refused` cu
  motivul; dacă e acceptată creează `invoice_validated`. **Salvează și `xml_attachment`**
  (diferența față de standard). Folosește utilitarele din `l10n_ro_edi`
  (`_request_ciusro_fetch_status`, `_request_ciusro_download_answer`).
- **`_l10n_ro_edi_send_invoice(xml_data)`** (override) — după trimitere, salvează
  XML-ul trimis pe documentele EDI.
- **`_l10n_ro_edi_fetch_invoices()`** (override) — variantă a metodei Odoo care
  folosește **`l10n_ro_download_einvoices_days`** ca fereastră (în loc de 1 zi);
  procesează mesajele acceptate/refuzate/primite și marchează ca refuzate facturile
  neindexate mai vechi de `HOLDING_DAYS`.
- **`_compute_show_reset_to_draft_button()`** (override) — afișează butonul „Reset to
  Draft" pentru facturile de intrare (`in_invoice`) ne-draft (utile pentru cele
  descărcate din SPV).

### `res.company` (`models/res_company.py`)
- Câmp **`l10n_ro_download_einvoices_days`** (Integer, default 60) cu constrângere
  `0 ≤ valoare ≤ 60`.

### `res.config.settings` (`models/res_config_settings.py`)
- Expune `l10n_ro_download_einvoices_days` (related) în Setări.

---

## 3. View-uri
`views/res_config_settings_views.xml` — adaugă câmpul „Maximum number of days to
download e-invoices" în secțiunea e-Factura din Setări (după
`l10n_ro_edi_anaf_imported_inv_journal_id`).

## 4. Endpoint-uri ANAF folosite
- `POST https://webservicesp.anaf.ro/prod/FCTEL/rest/transformare/{FACT1|FCN}`
  (cu `timeout=25`).
- Sincronizare/fetch/download prin utilitarele `l10n_ro_edi.models.utils`.

## 5. Observații / riscuri
- `README.rst` este **gol**.
- În `_l10n_ro_edi_fetch_invoice_sent_documents`, la eroarea de download se loghează
  `result['error']` în loc de `download_data['error']` (mesaj de eroare posibil incorect).
- `_logger.error("PDF CONTENT ...")` loghează conținutul PDF ca eroare — zgomot/risc
  de log-uri mari; ar trebui `debug`.
- Constrângerea limitează la max 60 de zile (limită impusă de API-ul ANAF de sincronizare).
</content>
