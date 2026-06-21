# `e_factura_attachments` — Atașamente („Anexa") în e-Factura

**Nume afișat:** *E factura attachments*
**Versiune:** 19.0.0.0.0 · **Licență:** AGPL-3 · **Autor:** Dakai Soft
**Categorie:** Accounting · **Generație:** Nativ Odoo 19
**Depinde de:** `account`, `l10n_ro`, `l10n_ro_edi`, `account_edi_ubl_cii`

---

## 1. Scop de business

Unele facturi au **documente anexate** (de ex. situații de lucrări, devize,
specificații) pe care furnizorul vrea să le trimită **împreună** cu factura
electronică. Acest modul permite ca PDF-urile a căror denumire începe cu **„Anexa"**
să fie:

1. **încorporate în XML-ul UBL** trimis la ANAF (ca `AdditionalDocumentReference`
   cu conținut binar embed), și
2. **îmbinate (merge) în PDF-ul facturii** generat de Odoo, astfel încât PDF-ul final
   conține factura + toate anexele.

Beneficiu: anexele călătoresc odată cu factura, atât în fluxul ANAF cât și în PDF-ul
pus la dispoziția clientului.

---

## 2. Ce extinde tehnic

### `account.move.send` (`models/account_move_send.py`) — model abstract

- **`_postprocess_invoice_ubl_xml(invoice, invoice_data)`** (override) —
  după generarea XML-ului UBL standard:
  - caută atașamentele PDF ale facturii cu numele `ilike 'Anexa%'` (excluzând PDF-ul
    principal al facturii);
  - dacă există, inserează pentru fiecare un nod
    **`cac:AdditionalDocumentReference`** în XML, înainte de
    `AccountingSupplierParty`, cu PDF-ul codat base64 în
    `cbc:EmbeddedDocumentBinaryObject` (cu `mimeCode` și `filename`);
  - folosește `dict_to_xml`, nsmap-ul builder-ului EDI și `cleanup_xml_node` pentru
    serializare corectă.

- **`_hook_invoice_document_after_pdf_report_render(invoice, invoice_data)`** (override) —
  după randarea PDF-ului facturii:
  - caută aceleași anexe „Anexa%";
  - folosește `OdooPdfFileReader`/`OdooPdfFileWriter` pentru a **concatena** PDF-ul
    principal cu fiecare anexă;
  - scrie PDF-ul îmbinat înapoi în `pdf_attachment` și în `invoice_pdf_report_id`;
  - eșecurile de merge sunt prinse și logate (factura nu e blocată).

---

## 3. Convenții & dependențe
- **Convenția de denumire „Anexa…"** este mecanismul de selecție — orice atașament
  PDF al facturii care începe cu „Anexa" este tratat ca anexă.
- Necesită `account_edi_ubl_cii` (infrastructura UBL/CII Odoo) pentru hook-urile de
  post-procesare XML.

## 4. Observații / riscuri
- `manifest` are `data: []` gol și **nu** declară `installable` explicit `True` în
  partea de view (deși are `installable: True`).
- Selecția pe nume `'Anexa%'` este sensibilă la diacritice/majuscule și la convenția
  de denumire — fragilă dacă utilizatorii nu respectă convenția.
- Merge-ul PDF poate eșua silențios (doar log `error`) — anexele ar putea lipsi din
  PDF fără avertisment vizibil pentru utilizator.
- Fără modele/câmpuri noi, fără view-uri, fără ACL.
</content>
