# `partner_licence_anaf` — Server de licențe + OAuth ANAF

**Nume afișat:** *Partner Licence Anaf*
**Versiune:** 19.0.0.1 · **Licență:** OPL-1 · **Autor:** Dakai SOFT SRL · **Mentenanță:** adrian-dks
**Categorie:** Licence for Anaf · **Rol:** SERVER
**Depinde de:** `partner_licence` · **Dependențe Python:** `PyJWT`

> Extinde [`partner_licence`](partner_licence.md) cu **fluxul complet OAuth2 cu ANAF**.
> Vezi **secțiunea 4 din [README](../README.md)** pentru arhitectura de ansamblu.

---

## 1. Scop de business

Acesta este „motorul" care face ca providerul să poată **autoriza la ANAF în numele
clienților**. Adaugă serverului de licențe:

- stocarea **credențialelor OAuth ANAF** (`client_id`, `client_secret`, URL-uri);
- fluxul OAuth2 complet: **authorize → code → token → refresh → revoke**;
- **reînnoirea automată** a token-urilor printr-un cron zilnic;
- gestionarea valabilității token-urilor (access token scurt, refresh token ~3 ani,
  max. 64 reînnoiri — limita ANAF).

Token-urile obținute sunt salvate în `json_data`-ul licenței și **împinse automat**
către clientul respectiv (prin mecanismul moștenit din `partner_licence`).

---

## 2. Ce conține tehnic

### `partner.licence.anaf_oauth` (`models/oauth_anaf.py`) — config OAuth
Câmpuri: `name`, `anaf_oauth_url` (default `https://logincert.anaf.ro/anaf-oauth2/v1`),
`anaf_callback_url` (calculat = `…/partner-licence/anaf_oauth/<id>`), `client_id`,
`client_secret` (tracked), `last_request_datetime`, `response_secret`, `licence_ids`.

Metode:
- **`_activeLicence()`** — licența curentă „în curs de actualizare" (`to_update=True`).
- **`saveData2Licence(data)`** — scrie `json_data` pe licența activă (incluzând
  `licence` și `state`), ceea ce declanșează (prin `write` moștenit) împingerea
  datelor către client.

### `res.partner.licence` (`models/customer_licence.py`) — extindere
Câmpuri noi: `oauth_id` (M2O config OAuth), `to_update`, `to_update_datetime`, `anaf_env`.

Metode:
- **`_refreshToken()`** — POST la `{anaf_oauth_url}/token` cu `grant_type=refresh_token`;
  la succes decodează JWT-ul (PyJWT, `verify_signature=False`) pentru a extrage `exp`,
  actualizează token-urile și incrementează `refresh_token_used`; salvează prin
  `saveData2Licence`. (lock cu `_freez_transactions` / `_release_transactions`).
- **`_revokeToken()`** — golește token-urile și le salvează.
- **`cronRefreshTokens()`** — pentru fiecare licență `open`, decide pe baza
  valabilităților din `json_data`:
  - dacă `client_token_valability` expirat **și** `refresh_token_used == 64` → **revoke**;
  - dacă `refresh_token_valability` expirat → **revoke**;
  - dacă doar `client_token_valability` expirat → **refresh**.
- **`setAnafEnvProduction` / `setAnafEnvTest`** — comută mediul.
- **`write(vals)`** (override) — sincronizează `state`/`anaf_env` în `json_data`.
- **`_freez_transactions` / `_release_transactions`** — lock-ul de concurență.
- **`prepare_redirect(message)`** — construiește URL-ul de revenire la client
  (`…/partner-licence/callback-anaf-oauth/<licence>?message=…`).
- **`_prepare_licence_values`** (override) — atașează `oauth_id` la crearea licenței.

### Controller (`controller/main.py`)
Fluxul OAuth2 complet (toate `auth=public`, `csrf=False`):
- **`GET /partner-licence/anaf-token/<licence_code>`** — pornește autorizarea:
  verifică permisiunile, blochează tranzacțiile, validează `client_id/secret`,
  impune **max. 1 request/minut**, generează `response_secret` și redirecționează la
  pagina ANAF `/authorize` (`response_type=code`, `token_content_type=jwt`).
- **`GET /partner-licence/anaf_oauth/<config_id>`** — **callback-ul de la ANAF**:
  primește `code`, face POST la `{anaf_oauth_url}/token` (`grant_type=authorization_code`),
  decodează JWT-ul access token-ului pentru `exp`, construiește `json_dump` cu
  access/refresh token + valabilități (refresh ~1095 zile) și îl salvează pe licență
  (→ împins la client). Apoi redirect la client.
- **`GET /partner-licence/anaf-revoke/<licence_code>`** — revocă token-ul.
- **`GET /partner-licence/anaf-refresh/<licence_code>`** — reînnoiește token-ul.
- **`redirect_anaf`, `redirectFromError`, `_checkPermissions`** — utilitare (permisiuni,
  redirect-uri, gestionarea erorilor cu mesaj salvat în `json_data`).

### Cron (`data/data.xml`)
- `ir.cron` „Refresh anaf token" — rulează `model.cronRefreshTokens()` zilnic la 04:00.

### View-uri (`views/oauth_anaf.xml`, `customer_licence.xml`, `l10n_ro_account_anaf_sync.xml`)
- Formular/listă pentru `partner.licence.anaf_oauth` (cu `client_id/secret`,
  callback URL, lista de licențe) + meniu „ANAF config Sync".

### Securitate
- ACL CRUD pe `partner.licence.anaf_oauth` pentru `base.group_user`.

---

## 3. Endpoint-uri ANAF folosite
- `GET {anaf_oauth_url}/authorize` — pagina de autorizare ANAF (logincert).
- `POST {anaf_oauth_url}/token` — schimb `code`/`refresh_token` pe token-uri.

## 4. Observații / riscuri
- JWT-ul ANAF este decodat cu **`verify_signature=False`** (doar pentru a citi `exp`) —
  acceptabil aici (token-ul vine direct de la ANAF), dar de notat.
- `timeout=1.5s` la schimbul de token în callback este **foarte mic** — poate eșua la
  latențe normale ANAF (în `_refreshToken` e `timeout=80`, inconsecvent).
- ACL `base.group_user` pe configul OAuth (care conține `client_secret`) este **prea
  permisiv** — secrete sensibile expuse oricărui user intern.
- Limita de **64 reînnoiri** și valabilitatea refresh-ului (1095 zile) sunt
  reguli ANAF; logica de revoke/refresh din cron le respectă.
- Câmpurile de scope (`e-factura`/`e-transport`) sunt **comentate** — sugerează suport
  multi-scope planificat dar neactivat.
</content>
