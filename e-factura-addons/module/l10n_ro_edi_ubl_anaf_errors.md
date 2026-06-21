# `l10n_ro_edi_ubl_anaf_errors` — Erori de validare ANAF

**Nume afișat:** *E-factura anaf errors*
**Versiune:** 19.0.0.0 · **Licență:** OPL-1 · **Autor:** Dakai SOFT SRL · **Mentenanță:** adrian-dks
**Categorie:** Account · **Generație:** Legacy (`account.edi.format` / `cius_ro`)
**Depinde de:** `l10n_ro_account_edi_ubl`

---

## 1. Scop de business

Când o factură este respinsă de ANAF (stare `nok` / blocking `error`), Odoo standard
nu afișează clar **de ce** a fost respinsă. Acest modul:

- **descarcă automat din ANAF arhiva ZIP** cu erorile de validare,
- **extrage și afișează mesajele de eroare** în chatter-ul facturii (în română:
  „Erori validare ANAF: …"),
- permite operatorului să **reprogrameze re-trimiterea** facturii corectate
  (buton „Schedule Resend"), resetând identificatorii de tranzacție.

Beneficiu: contabilul vede exact ce reguli ANAF au fost încălcate, fără a se loga
manual în SPV.

---

## 2. Ce extinde tehnic

### `account.edi.document` (`models/account_edi_document.py`)
- Adaugă câmpul boolean **`l10n_ro_reload_einvoice`**.
- Metoda **`l10n_ro_set_bloching_level_error()`** — marchează documentul pentru
  re-trimitere: setează flag-ul și șterge `l10n_ro_edi_transaction` și
  `l10n_ro_edi_download` de pe `move_id` (factura), astfel încât următoarea
  trimitere să o ia de la capăt.

### `account.edi.format` (`models/account_edi_format.py`)
Conține logica principală de comunicare cu ANAF:

- **`l10n_ro_get_idDescarcare(invoice)`** — dacă nu avem `id_descarcare`, îl caută
  interogând mesajele ANAF (`_l10n_ro_get_anaf_efactura_messages`) pe filtrele
  `E` (erori) și `T` (toate) într-un interval de timp în jurul datei documentului,
  apoi corelează după `id_solicitare == l10n_ro_edi_transaction`.
- **`l10n_ro_download_zip_anaf(invoice, anaf_config=False)`** — apelează endpoint-ul
  ANAF **`/descarcare`** (`anaf_config._l10n_ro_einvoice_call`) și întoarce
  `{success, zip_content}` sau `{success: False, error}`.
- **`l10n_ro_check_anaf_error_xml(zip_content, transaction)`** — deschide ZIP-ul în
  memorie, găsește fișierul `<transaction>.xml`, îl decodează prin formatul
  `cius_ro` și extrage toate tag-urile `Error` (`errorMessage`) într-un mesaj HTML.
- **`_l10n_ro_post_invoice_step_2(...)`** (override) — după pasul 2 standard, dacă
  starea e `error`, descarcă ZIP-ul, extrage erorile, le postează în chatter și,
  dacă documentul e marcat pentru reload, resetează `l10n_ro_edi_transaction`.
- **`_l10n_ro_anaf_call(...)`** — metodă utilitar (marcată „To be merged") care
  parsează răspunsul XML ANAF pe namespace-urile `respUploadFisier` (pasul 1) și
  `stareMesajFactura` (pasul 2) și normalizează stările (`in prelucrare`, `nok`,
  `ok`, `XML cu erori nepreluat de sistem`) în dicționare de rezultat.

### `res.company` (`models/res_company.py`)
- Fișier prezent dar **fără logică** (doar import-uri). Placeholder.

---

## 3. View-uri

`views/account_edi_doocument.xml` (numele fișierului are typo: „doocument"):
- Adaugă în lista `edi_document_ids` de pe formularul facturii butonul
  **„Schedule Resend"** (`l10n_ro_set_bloching_level_error`), vizibil doar pentru
  grupul tehnic `base.group_no_one` și ascuns când starea e `cancel`/`sent`.

## 4. Securitate
`security/ir.model.access.csv` este **gol** (comentat și în manifest). Nu introduce
modele noi care necesită ACL.

---

## 5. Endpoint-uri ANAF folosite
- `GET /descarcare?id=<id_descarcare>` — descărcarea ZIP-ului (răspuns semnat sau erori).
- mesaje SPV prin `_l10n_ro_get_anaf_efactura_messages` (filtre `E`, `T`).

## 6. Observații / riscuri
- **Cod depreciat** mare lăsat comentat (`_post_invoice_edi`) — fost flux complet,
  păstrat ca referință de migrare.
- În `l10n_ro_get_idDescarcare`, `start`/`end` par inversate (start = create_date − 60s,
  end = create_date − 2 zile) — de verificat semantica intervalului cerut de ANAF.
- Comparațiile pe `status_code` amestecă string (`"400"`) și int (`200`) — fragil.
- Typo în numele fișierului view (`account_edi_doocument.xml`) — cosmetic.
</content>
