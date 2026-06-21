# `l10n_ro_edi_ubl_pdf` — Generare PDF din XML (ANAF)

**Nume afișat:** *E-factura Render PDF*
**Versiune:** 19.0.0.0 · **Licență:** OPL-1 · **Autor:** Dakai SOFT SRL · **Mentenanță:** adrian-dks
**Categorie:** Account · **Generație:** Legacy (`account.edi.xml.cius_ro`)
**Depinde de:** `l10n_ro_account_edi_ubl`

---

## 1. Scop de business

Facturile primite prin e-Factura sosesc ca **XML** (UBL), nu ca PDF. Pentru
contabili, un XML brut este greu de citit. Acest modul, la **importul** unei
facturi, **generează automat reprezentarea PDF vizuală** a facturii folosind
serviciul oficial ANAF de „transformare" XML→PDF, și o atașează facturii în Odoo.

Dacă serviciul ANAF nu poate produce PDF-ul (sau lipsește XML-ul), modulul cade pe
o variantă de rezervă: generează PDF-ul standard Odoo al facturii.

Beneficiu: fiecare factură importată are imediat un PDF lizibil în atașamente/chatter.

---

## 2. Ce extinde tehnic

### `account.edi.xml.cius_ro` (`models/account_edi_xml_cius_ro.py`)

- **`_import_invoice(...)`** (override) — după importul standard, verifică dacă XML-ul
  conține deja `AdditionalDocumentReference` (adică un PDF embed). Dacă **nu**:
  1. încearcă `l10n_ro_renderAnafPdf(invoice)` (PDF de la ANAF);
  2. dacă eșuează, generează PDF-ul Odoo prin raportul
     `account.account_invoices_without_payment` și îl atașează.

- **`l10n_ro_renderAnafPdf(invoice)`** — găsește atașamentul XML al facturii
  (`<l10n_ro_edi_transaction>.xml`), îl decodează și face un **POST** către serviciul
  ANAF de transformare:
  - URL: `https://webservicesp.anaf.ro/prod/FCTEL/rest/transformare/{tip}/{DA}`
  - `tip` = `FCN` pentru note de credit (refund), altfel `FACT1`;
  - al doilea parametru `DA` cere validarea.
  - **Fallback la respingere:** dacă răspunsul conține „The requested URL was
    rejected", elimină atributul `xsi:schemaLocation` din XML și reîncearcă (workaround
    pentru WAF-ul ANAF).
  - PDF-ul rezultat este atașat prin `l10n_ro_addPDF_from_att`.

- **`l10n_ro_addPDF_from_att(invoice, pdf)`** — creează un `ir.attachment` de tip PDF
  legat de factură și postează un mesaj în chatter cu atașamentul.
  - Notă: aplică un „fix de padding" base64 (`pdf + b'=' * (len(pdf) % 3)`).

---

## 3. Endpoint-uri ANAF folosite
- `POST https://webservicesp.anaf.ro/prod/FCTEL/rest/transformare/{FACT1|FCN}/DA`
  — transformare XML → PDF (serviciu public ANAF, fără autentificare).

## 4. Securitate / View-uri
- Fără view-uri. `security/ir.model.access.csv` gol (comentat în manifest).

## 5. Observații / riscuri
- `requests.post` **fără `timeout`** — poate bloca la indisponibilitatea ANAF.
- Excepțiile sunt înghițite (`except Exception: return False`) — eșecurile sunt
  silențioase (doar fallback).
- „Fixul" de padding base64 (`% 3` în loc de `% 4`) este suspect matematic — de
  reverificat dacă chiar corectează padding-ul corect.
- URL hardcodat pe mediul **`prod`** — nu există variantă de test.
</content>
