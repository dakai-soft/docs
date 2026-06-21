# `partner_anaf_authorize` — Autorizare ANAF prin Provider (client, legacy)

**Nume afișat:** *Partner ANAF Authorize*
**Versiune:** 19.0.0.2 · **Autor:** Dakai SOFT · **Mentenanță:** Flavia0320
**Categorie:** Localization · **Rol:** CLIENT (partea instalată la client)
**Depinde de:** `base`, `l10n_ro_edi`

> Vezi întâi **secțiunea 4 din [README](../README.md)** pentru arhitectura
> client–server „Provider de licențe".

---

## 1. Scop de business

Permite unei companii (clientul) să obțină **token-ul OAuth ANAF** fără a-și gestiona
singură credențialele ANAF, delegând autorizarea către un **Provider** (serverul
partenerului). Compania:

1. cere o **licență** de la provider (pe baza VAT-ului);
2. este redirecționată către provider pentru a face autorizarea ANAF;
3. primește înapoi token-urile, salvate în câmpurile `l10n_ro_edi_*` ale companiei.

Statusul licenței (draft → waiting → confirmed_licence → confirmed_token → blocked)
funcționează și ca **mecanism comercial de activare/dezactivare**.

---

## 2. Ce extinde tehnic

### `res.company` (`models/res_company.py`)
Câmpuri noi:
- **`provider_id`** (M2O `l10n.ro.account.anaf.sync.provider`, required) — providerul.
- **`licence_status`** (selecție: draft / waiting_licence / confirmed_licence /
  confirmed_token / blocked).
- **`err_message`**, **`provider_licence`**, **`code`** (codul OAuth ANAF).
- **`_l10n_ro_get_anaf_sync(scope)`** (override) — filtrează configurațiile de sync
  ANAF la cele cu `licence_status == 'confirmed_token'` (doar cu token valid).
- **`get_token_from_anaf_website()`** — redirecționează către
  `{provider.url}/partner-licence/anaf-token/{provider_licence}`.

### `res.config.settings` (`models/l10n_ro_account_anaf_sync.py`)
- Expune `provider_id`, `licence_status`, `err_message`, `provider_licence`, `code`.
- **`get_anaf_licence()`** — POST la `{provider.url}/partner-licence/register` cu
  `{username, database, url, vat}`; setează `provider_licence` și `licence_status`.
- **`get_token_from_anaf_website()`**, **`revoke_access_token()`**,
  **`refresh_access_token()`** — apeluri GET către provider
  (`anaf-token` / `anaf-revoke` / `anaf-refresh`).
- **`button_l10n_ro_edi_generate_token_auth()`** — dacă există provider, pornește
  fluxul prin provider, altfel cade pe metoda standard.

### `l10n.ro.account.anaf.sync.provider` (model nou)
- `name` (M2O `res.partner`) + `url` (URL-ul de autentificare al providerului).

### Controller (`controller/main.py`)
- **`POST /partner-licence/update-partner-licence`** (`auth=public`, `csrf=False`) —
  **endpoint apelat de server** pentru a împinge token-urile/starea către client:
  - caută compania după `provider_licence`;
  - actualizează `licence_status` (open→confirmed, altfel blocked + golește token-urile);
  - scrie câmpurile permise (`code`, `l10n_ro_edi_access_token`,
    `..._refresh_token`, `..._expiry_date`, `client_id`, `client_secret`);
  - tratează `error_msg`.
- **`GET /partner-licence/callback-anaf-oauth/<provider_licence>`** — redirect la
  formularul companiei după autorizare.

### Data (`data/data.xml`) — *comentat în manifest*
- `ir.rule` care limitează configurarea ANAF la companii cu țară `RO`.

## 3. Observații / riscuri
- `requests.post` în `get_anaf_licence` **fără `timeout`**.
- Endpoint-ul `update-partner-licence` este `auth=public` și scrie token-uri — se
  bazează pe secretul `provider_licence` ca autentificare (de protejat).
- Lista albă `_permited` previne scrierea câmpurilor neașteptate (bun).
- Coexistă cu `partner_anaf_authorize_efactura` (variantă mai nouă) — **nu** instala
  ambele simultan (definesc același model provider și endpoint).
</content>
