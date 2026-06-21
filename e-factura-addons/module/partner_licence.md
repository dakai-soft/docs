# `partner_licence` — Server de licențe (bază)

**Nume afișat:** *Partner licence*
**Versiune:** 19.0.0.0 · **Licență:** OPL-1 · **Autor:** Dakai SOFT SRL · **Mentenanță:** adrian-dks
**Categorie:** Licence · **Rol:** SERVER (instalat la partener/provider)
**Depinde de:** `sale_management`

> Acesta este **serverul** din arhitectura client–server. Vezi **secțiunea 4 din
> [README](../README.md)**.

---

## 1. Scop de business

Implementează **registrul central de licențe** pe care îl rulează partenerul
(Dakai/provider). Rolurile principale:

- **înrolarea clienților** (un client = o instanță Odoo identificată prin VAT + DB + URL);
- **generarea unui cod de licență unic** per client;
- **împingerea de date** (token-uri, stare) înapoi către instanța clientului prin
  HTTP;
- gestionarea **stării licenței** (open/closed) ca mecanism comercial;
- un mecanism de **notificare a expirării** licenței.

Partea specifică ANAF (OAuth) este adăugată de modulul
[`partner_licence_anaf`](partner_licence_anaf.md).

---

## 2. Ce conține tehnic

### `res.partner.licence` (`models/customer_licence.py`) — modelul central
Câmpuri: `name` (calculat), `scope`, `partner_id`, `vat` (compute/inverse pe partener),
`json_data` (Json — datele împinse spre client, inclusiv token-uri), `username`,
`licence` (codul), `database`, `url`, `state` (open/closed), `to_update` +
`to_update_datetime` (lock pentru request-uri concurente), `anaf_env` (test/production).

Metode cheie:
- **`enrole_customer(**kw)`** — punctul de intrare al înrolării:
  - validează prezența `username/database/url/vat`;
  - dacă licența există → o întoarce; dacă există mai multe → le închide (eroare);
  - dacă nu există → **creează** una nouă cu cod generat și o întoarce;
  - răspuns structurat `{error, message, response_data:{licence, vat, state}}`.
- **`_generateLicenceString()`** — generează un cod de forma `XXXX-XXXX-XXXX-XXXX`
  (16 caractere alfanumerice), verificând unicitatea în DB (recursiv).
- **`getLicences` / `getActiveLicences` / `_getLicences`** — căutare după `licence`
  sau (`url` + `vat`); „active" = `state == open`.
- **`_send_data_to_customer()`** — pentru fiecare licență, face **POST** la
  `{client_base_url}/partner-licence/update-partner-licence?db=…&licence=…` cu
  `json_data`; logează răspunsul în chatter. Declanșat automat la
  **`write()`** când se modifică `json_data`.
- **`_getBaseUrl` / `_getCustomerUrl`** — construiesc URL-ul clientului din `url`.
- **`_checkPermission(...)`** — control de concurență/securitate: închide
  request-urile „orfane" (lock mai vechi de `_MAX_TIME=300s`); refuză dacă există un
  request în curs („Busy…"), dacă lipsește licența („Restricted! No licence"), sau
  dacă `referer` nu corespunde `url`-ului licenței („Wrong request origin").
- **`_prepare_licence_values(values)`** — hook de extensie (folosit de modulul ANAF).

### `notification.client` (`models/notification_client.py`)
- **`start_notification_loop()`** — buclă `while True` cu `time.sleep(interval*60)`
  care apelează `show_notification()`.
- **`get_notification_interval()`** — POST la `/partner-licence/get_interval`.
- **`show_notification()`** — notificare „Licența ta a expirat!".

### Controller (`controller/main.py`)
- **`POST /partner-licence/register`** (`auth=public`, `csrf=False`) — apelează
  `res.partner.licence.enrole_customer(**kw)`. Acesta este endpoint-ul pe care îl
  apelează clientul pentru a obține o licență.

### Cron (`data/data.xml`)
- `ir.cron` „Notificare licenta" — rulează `model.start_notification_loop()` la fiecare
  1 minut.

### Securitate
- ACL CRUD pe `res.partner.licence` și `notification.client` pentru `base.group_user`.

### View-uri
- `views/customer_licence.xml` — interfața de administrare a licențelor.

---

## 3. Observații / riscuri
- ⚠️ **`start_notification_loop()` cu `while True` rulat din cron** este periculos:
  cron-ul ar bloca un worker la nesfârșit. Probabil necesită regândire (mai degrabă
  o singură verificare per execuție de cron).
- **Bug:** în `_send_data_to_customer` se compară `s.state == "close"`, dar valorile
  selecției sunt `open`/`closed` → condiția nu se activează niciodată corect.
- `requests.post` **fără `timeout`** în `_send_data_to_customer` și
  `get_notification_interval`.
- `get_notification_interval` apelează un URL **relativ** (`/partner-licence/get_interval`)
  cu `requests.post` — nu va funcționa fără host absolut.
- ACL `base.group_user` pe registrul de licențe este **prea permisiv** pentru un
  registru sensibil (token-uri în `json_data`).
</content>
