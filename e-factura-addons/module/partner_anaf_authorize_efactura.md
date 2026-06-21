# `partner_anaf_authorize_efactura` — Autorizare ANAF prin Provider (client, nativ)

**Nume afișat:** *Partner ANAF Authorize*
**Versiune:** 19.0.0.0.1 · **Licență:** LGPL-3 · **Autor:** Dakai SOFT, NextERP Romania · **Mentenanță:** Flavia0320
**Categorie:** Localization · **Status:** Beta · **Rol:** CLIENT
**Depinde de:** `base`, `l10n_ro_efactura`

> Variantă **modernizată** a [`partner_anaf_authorize`](partner_anaf_authorize.md),
> aliniată la modulul `l10n_ro_efactura`. Vezi **secțiunea 4 din [README](../README.md)**
> pentru arhitectura client–server.

---

## 1. Scop de business

Identic ca scop cu `partner_anaf_authorize`: obținerea token-ului ANAF prin
intermediul unui **Provider**, fără ca clientul să gestioneze credențialele ANAF.
Diferența este **integrarea cu fluxul nativ** `l10n_ro_efactura`:
- folosește câmpuri prefixate consecvent `l10n_ro_edi_*`;
- se integrează în `res.config.settings` peste butoanele standard ale Odoo;
- reutilizează `_l10n_ro_edi_process_token_response` și
  `_l10n_ro_edi_refresh_access_token` din modulul de bază.

---

## 2. Ce extinde tehnic

### `res.company` (`models/res_company.py`)
Câmpuri noi (prefix `l10n_ro_edi_`):
- **`l10n_ro_edi_provider_id`** (M2O provider, required),
- **`l10n_ro_edi_licence_status`** (draft/waiting/confirmed_licence/confirmed_token/blocked),
- **`l10n_ro_edi_error_message`**, **`l10n_ro_edi_provider_licence`**.
- **`get_token_from_anaf_website()`** — redirect la `…/partner-licence/anaf-token/<licence>`.
- **`_l10n_ro_edi_refresh_access_token()`** (override) — dacă există provider, apelează
  `…/partner-licence/anaf-refresh/<licence>` (cu `timeout=10`); altfel `super()`.

### `res.config.settings` (`models/res_config_settings.py`)
- Related pe `l10n_ro_edi_provider_id`, `l10n_ro_edi_licence_status`,
  `l10n_ro_edi_provider_licence`.
- **`button_l10n_ro_edi_generate_token()`** (override) — dacă există provider, pornește
  fluxul prin provider; altfel metoda standard Odoo.
- **`get_anaf_licence()`** — POST la `…/partner-licence/register` (cu `timeout=10`),
  setează licența și statusul.

### `l10n.ro.account.anaf.sync.provider` (model nou)
- `name` (M2O `res.partner`) + `url`.

### Controller (`controller/main.py`)
- **`POST /partner-licence/update-partner-licence`** — caută compania după
  `l10n_ro_edi_provider_licence`; la `access_token` apelează
  `company._l10n_ro_edi_process_token_response(kw)` (procesare nativă) și setează
  `confirmed_token`; tratează `state` (blocked → golește token-uri), `error_msg`, și
  **`anaf_env`** (production/test → setează `l10n_ro_edi_test_env`).
- **`GET /partner-licence/callback-anaf-oauth/<provider_licence>`** — redirect la
  formularul companiei.

### Securitate (`security/ir.model.access.csv`)
- ACL pe `l10n.ro.account.anaf.sync.provider`: citire pentru account user/invoice,
  CRUD complet pentru account manager.

### View-uri
- `views/res_company.xml`, `views/l10n_ro_account_anaf_sync_view.xml`,
  `views/res_config_settings_view.xml`.

## 3. Documentație OCA inclusă
- `README.rst` + `readme/` (DESCRIPTION/USAGE/CONTRIBUTORS) generate cu
  `oca-gen-addon-readme`. Descriere: *„adds an instrument of getting ANAF token
  through ANAF Provider"*. Contribuitor menționat: Fekete Mihai (NextERP).

## 4. Observații / riscuri
- **Diferența cheie față de varianta legacy:** suportă comutarea mediului
  test/producție (`anaf_env`) și folosește procesarea nativă a token-ului.
- În `get_token_from_anaf_website` se setează `l10n_ro_edi_oauth_error` (câmp care
  pare a proveni din modulul de bază `l10n_ro_efactura`, nu e definit aici) — atenție
  la dependența implicită.
- A nu se instala împreună cu `partner_anaf_authorize` (conflict de model/endpoint).
</content>
