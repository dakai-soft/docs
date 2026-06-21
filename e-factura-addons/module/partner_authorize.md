# `partner_authorize` — Autorizare partener (schelet)

**Nume afișat:** *Partner authorize*
**Versiune:** 19.0.0.0 · **Licență:** OPL-1 · **Autor:** Dakai SOFT SRL · **Mentenanță:** adrian-dks
**Categorie:** Licence · **Rol:** CLIENT (incomplet)
**Depinde de:** `sale_management`, `l10n_ro_config`

---

## 1. Scop de business (intenționat)

Pare a fi un **schelet / început** de modul generic de autorizare a clientului față
de serverul de licențe (varianta de bază, fără partea ANAF). În forma actuală este
**incomplet** și nu adaugă funcționalitate utilizabilă de sine stătătoare.

---

## 2. Conținut tehnic

### `controller/main.py`
- Clasa `PartnerLicenceAuthorize(http.Controller)`:
  - **`get_licence(self)`** — metodă **ne-rutată** (nu are decorator `@http.route`) și
    care folosește `self.company_id`/`self.provider_id` (atribute inexistente pe un
    controller) → **cod nefuncțional/copiat** din modulele client. Pare lăsat ca
    referință.
  - **`POST /partner-licence/get_interval`** (`auth=public`, `csrf=False`) — apelează
    `res.partner.licence.getActiveLicences(**kwargs)`; dacă nu există, întoarce
    `{'interval': 60}`. Folosit (teoretic) de mecanismul de notificare a licenței.

### `models/__init__.py`
- **Gol** — niciun model definit.

### `security/ir.model.access.csv`
- Doar antetul, **fără reguli**.

---

## 3. Observații / riscuri
- Modul **incomplet**: `get_licence` nu este expus și ar arunca erori dacă ar fi
  apelat (atribute lipsă). Probabil rest dintr-o refactorizare.
- Singura piesă funcțională este endpoint-ul `get_interval`, care depinde de
  `res.partner.licence` (definit în `partner_licence`) — deși **`partner_licence` nu
  este în `depends`**, ceea ce poate provoca erori dacă modelul lipsește.
- Recomandare: de clarificat dacă modulul trebuie finalizat, fuzionat cu
  `partner_anaf_authorize`, sau eliminat.
</content>
