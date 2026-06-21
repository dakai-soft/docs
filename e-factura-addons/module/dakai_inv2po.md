# `dakai_inv2po` — Comandă de achiziție din factură (Invoice → PO)

**Nume afișat:** *Dakai inv2po*
**Versiune:** 19.0.0.0 · **Licență:** OPL-1 · **Autor:** Dakai SOFT SRL · **Mentenanță:** adrian-dks
**Categorie:** Accounting · **Generație:** Legacy
**Depinde de:** `l10n_ro_account_edi_ubl`, `purchase_enterprise`

---

## 1. Scop de business

Multe companii primesc factura furnizorului **înainte** de a avea o comandă de
achiziție în sistem (mai ales facturile sosite automat prin e-Factura). Acest modul
permite, dintr-o **factură furnizor postată**, crearea cu un click a unei
**comenzi de achiziție (RFQ)** corespunzătoare, corelând cantitățile și prețurile.

În plus, corectează **corelarea cu mișcările de stoc** astfel încât evaluarea
stocului (intrarea de marfă de la furnizor) să fie legată corect de factură.

Beneficiu: trasabilitate completă factură ↔ comandă ↔ recepție, fără introducerea
manuală a comenzii.

---

## 2. Ce extinde tehnic

### `account.move` (`models/account_move.py`)

- **`createPo()`** — pentru fiecare factură:
  - validează că toate liniile de tip `product` au produs setat (altfel `UserError`);
  - creează un `purchase.order` cu `partner_id`, `date_approve`, `date_planned`
    luate din factură;
  - apelează `createPOLines` pentru a genera liniile.
- **`_poValues(line)`** — construiește valorile liniei PO din linia de factură;
  **setează `standard_price` al produsului = `price_unit`** din factură (efect
  secundar: actualizează costul produsului); leagă `invoice_lines` și taxele.
- **`createPOLines(pOrder, invlines)`** — creează liniile `purchase.order.line`.
- **`_stock_account_get_last_step_stock_moves()`** (override din `stock_account`) —
  adaugă la mișcările de stoc relevante pentru factură mișcările provenite din
  comanda de achiziție:
  - pentru `in_invoice`: mișcări `done` cu sursa = locație furnizor;
  - pentru `in_refund`: mișcări `done` cu destinația = locație furnizor.
  Asigură evaluarea corectă a costului la facturile de intrare/retur.

### `purchase.order.line` (`models/purchase.py`)
- Override-uri „pass-through" pe `_prepare_stock_move_vals` și `_create_stock_moves`
  (apelează `super` și întorc rezultatul) — momentan **fără logică suplimentară**
  (puncte de extensie pregătite).

---

## 3. View-uri

`views/account_invoice.xml`:
- Adaugă pe formularul facturii (după `button_set_checked`) butonul
  **„Create RFQ"** (`createPo`), vizibil doar când: `move_type == 'in_invoice'`,
  `purchase_order_count == 0` și `state == 'posted'`.

## 4. Observații / riscuri
- **Efect secundar important:** `_poValues` modifică `standard_price` al produsului —
  poate altera costurile standard neașteptat dacă prețul de pe factură diferă.
- `createPo` setează `date_approve`/`date_planned` dar nu pune liniile la creare
  (linia comentată `order_line`), liniile fiind adăugate ulterior — comanda e creată
  posibil în stare `draft` fără confirmare.
- `purchase_enterprise` este dependență **Enterprise** — modulul nu rulează pe
  Community.
- Fără ACL (CSV gol).
</content>
