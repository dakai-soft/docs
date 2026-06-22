# Ghid de testare — Contracte inteligente (Smart Contract)

**Suita de module:** `smart_contract`, `sale_contract`, `purchase_contract`, `contract_overwrite`,
`smart_contract_blanket_order`, `smart_contract_gdpr`, `smart_contract_notification`,
`smart_contract_upsale_downsale`
**Platformă:** Odoo 19.0 · **Autor module:** Dakai SOFT
**Public țintă:** dezvoltatori, QA, ingineri de mentenanță

> **Documente conexe:** [`ghid_tehnic.md`](ghid_tehnic.md) (arhitectură și cod),
> [`ghid_fluxuri_business.md`](ghid_fluxuri_business.md) (fluxuri de business),
> [`manual_utilizare.md`](manual_utilizare.md) și [`ghid_procese.md`](ghid_procese.md)
> (utilizare). Acest ghid descrie **strategia de testare**, **suita existentă**, **cum se
> rulează testele local și în CI** și **cum se scriu teste noi**.

---

## Cuprins

1. [Strategia de testare](#1-strategia-de-testare)
2. [Tipuri de teste folosite](#2-tipuri-de-teste-folosite)
3. [Harta suitei de teste](#3-harta-suitei-de-teste)
4. [Cum rulezi testele local](#4-cum-rulezi-testele-local)
5. [Integrarea continuă (GitHub Actions)](#5-integrarea-continuă-github-actions)
6. [Pre-commit și linting](#6-pre-commit-și-linting)
7. [Detaliu pe fișiere de test](#7-detaliu-pe-fișiere-de-test)
8. [Acoperire și lacune cunoscute](#8-acoperire-și-lacune-cunoscute)
9. [Cum scrii un test nou](#9-cum-scrii-un-test-nou)
10. [Checklist înainte de merge](#10-checklist-înainte-de-merge)
11. [Depanarea testelor](#11-depanarea-testelor)

---

## 1. Strategia de testare

Suita aplică o piramidă de testare adaptată la Odoo:

| Nivel | Ce verifică | Unde |
|---|---|---|
| **Unit „pur” (fără Odoo)** | Logică izolată, fără ORM și fără bază de date | `smart_contract/tools/test_legacy_migration.py` |
| **Integrare ORM (`TransactionCase`)** | Modele, câmpuri calculate, constrângeri, metode de business, parser | `*/tests/test_*.py` |
| **End-to-end / CI** | Instalarea modulelor + rularea tuturor testelor pe o bază reală PostgreSQL | GitHub Actions (`.github/workflows/test.yml`) |

Principii:

- **Determinism.** Testele de parser fixează parametrii de configurare în `setUpClass`
  (adâncime maximă, allowlist, blocklist), ca rezultatele să nu depindă de starea bazei.
- **Izolare prin tranzacții.** `TransactionCase` face rollback după fiecare test, deci
  datele de test nu persistă.
- **Etichetare.** Testele de parser folosesc `@tagged('post_install', '-at_install')`, ca să
  ruleze **după** instalarea modulului (au nevoie de schema completă).
- **CI selectiv.** CI detectează automat doar modulele care **au** directoare `tests/` cu
  fișiere `test_*.py` și rulează testele lor — evită rularea inutilă a întregului Odoo.

---

## 2. Tipuri de teste folosite

### 2.1 `unittest.TestCase` (Python pur)

Folosit pentru cod care **nu** depinde de ORM-ul Odoo — de exemplu convertorul de sintaxă
veche `${...}` → `<span class="sc-var">`. Avantaj: rulează în milisecunde, fără bază de date,
poate fi rulat și în afara mediului Odoo.

### 2.2 `odoo.tests.TransactionCase`

Clasa de bază standard Odoo. Fiecare metodă rulează într-o tranzacție anulată la final.
Folosită pentru:

- crearea de înregistrări (`res.partner`, `smart.contract`, `smart.contract.element`);
- verificarea câmpurilor calculate (`_compute_*`);
- verificarea constrângerilor și a metodelor `action_*` / business;
- testarea parserului de variabile (rezolvare, blocare, KV, status).

### 2.3 Etichete (`@tagged`)

`@tagged('post_install', '-at_install')` spune lui Odoo să ruleze testul **după** instalarea
modulelor (`post_install`) și **nu** în timpul instalării (`-at_install`). Este obligatoriu
pentru testele care creează `smart.contract`, deoarece au nevoie de schema completă.

---

## 3. Harta suitei de teste

Statistici globale (la momentul redactării):

| Indicator | Valoare |
|---|---|
| Fișiere de test | 5 |
| Clase de test | 5 |
| Metode de test | 45 |
| Teste `TransactionCase` (cu ORM) | 26 |
| Teste `unittest` (Python pur) | 19 |
| Module **cu** teste | 3 (`smart_contract`, `smart_contract_gdpr`, `smart_contract_notification`) |
| Module **fără** teste | 5 (`sale_contract`, `purchase_contract`, `contract_overwrite`, `smart_contract_blanket_order`, `smart_contract_upsale_downsale`) |

| Fișier | Clasă | Bază | Metode | Rol |
|---|---|---|---:|---|
| `smart_contract/tests/test_parser.py` | `TestParser` | `TransactionCase` `@tagged` | 19 | Parserul de variabile `sc-var` |
| `smart_contract/tests/test_sale.py` | `TestSale` | `TransactionCase` | 4 | Integrarea cu comenzile de vânzare |
| `smart_contract/tools/test_legacy_migration.py` | `TestConvertLegacyText` | `unittest.TestCase` | 19 | Convertor sintaxă veche `${...}` |
| `smart_contract_gdpr/tests/test_smart_contract_gdpr.py` | `TestSmartContractGdpr` | `TransactionCase` | 2 | Tipul de document GDPR |
| `smart_contract_notification/tests/test_smart_contract_notification.py` | `TestSmartContractNotification` | `TransactionCase` | 2 | Tipul de document Notificare |

`tests/__init__.py` ale modulelor importă fișierele de mai sus, astfel încât Odoo să le
descopere automat.

---

## 4. Cum rulezi testele local

### 4.1 Cerințe

- Odoo 19.0 (versiunea este fixată în fișierul `.odoo-version`).
- PostgreSQL.
- Modulul OCA `contract` disponibil în `addons-path` (dependență a `contract_overwrite` și a
  fluxurilor de abonament).

### 4.2 Testele Python pure (cele mai rapide, fără Odoo)

Convertorul de sintaxă veche nu are nevoie de Odoo:

```bash
cd /path/to/sales-advance
python3 -m unittest smart_contract.tools.test_legacy_migration -v
# sau direct:
python3 smart_contract/tools/test_legacy_migration.py
```

### 4.3 Testele Odoo (`TransactionCase`) via `odoo-bin`

Modul recomandat, identic cu CI:

```bash
python /opt/odoo/odoo-bin \
  --addons-path=/opt/odoo/addons,/opt/dakai-soft/contract_oca,. \
  --db_host=localhost --db_port=5432 \
  --db_user=odoo --db_password=odoo \
  -d test_db \
  -i smart_contract,smart_contract_gdpr,smart_contract_notification \
  --test-enable \
  --test-tags /smart_contract \
  --stop-after-init \
  --log-level=test \
  --max-cron-threads=0
```

- `-i` instalează modulele (la prima rulare); ulterior poți folosi `-u` pentru a le actualiza.
- `--test-enable` activează rularea testelor la instalare/actualizare.
- `--test-tags /smart_contract` rulează doar testele modulului `smart_contract` (prefixul `/`
  filtrează pe modul). Schimbă numele pentru alt modul.
- `--stop-after-init` oprește serverul după rulare (util în scripturi/CI).
- `--max-cron-threads=0` dezactivează cron-urile în timpul testelor.

Pentru a rula **un singur fișier/clasă/metodă**, folosește o etichetă mai specifică, de ex.:

```bash
--test-tags /smart_contract:TestParser
--test-tags /smart_contract:TestParser.test_depth_limit
```

### 4.4 Notă despre pytest

`DEVELOPER_GUIDE.md` menționează `pytest` (`python -m pytest smart_contract/tests/test_parser.py -v`).
Aceasta funcționează **doar** dacă mediul are `pytest-odoo` configurat și un Odoo disponibil
în `PYTHONPATH`. Pentru reproducerea exactă a CI, folosește comanda `odoo-bin` de la 4.3.

---

## 5. Integrarea continuă (GitHub Actions)

Există două workflow-uri în `.github/workflows/`.

### 5.1 `ci.yml` — lint (pe orice push/PR)

```
Job: lint
  - Python 3.12
  - pip install ruff
  - ruff check . --select E722,F811,F821,F632
```

Rulează la **fiecare** push și PR. Nu rulează teste, doar verificări statice (vezi §6).

### 5.2 `test.yml` — teste unitare Odoo

- **Declanșare:** push și PR pe ramuri care se potrivesc cu `*.0` (ex. `19.0`).
- **Concurență:** rulările anterioare pe aceeași ramură sunt anulate (`cancel-in-progress`).
- **Servicii:** PostgreSQL 15 (`odoo`/`odoo`).
- **Python:** 3.11.
- **Pași:**
  1. Citește versiunea din `.odoo-version`.
  2. Clonează Odoo la tag-ul versiunii (`git clone --branch 19.0 ... /opt/odoo`).
  3. Instalează dependențele de sistem (`libldap2-dev`, `libsasl2-dev`, `libssl-dev`,
     `build-essential`) și `pip install -r /opt/odoo/requirements.txt`.
  4. **Auto-detectează modulele cu teste**: caută fișiere `__manifest__.py` și verifică dacă
     directorul `tests/` conține `test_*.py`; produce o listă JSON de module.
  5. Rulează `odoo-bin` cu `--test-enable --test-tags /<modul> --stop-after-init`.

> **Important pentru contributori:** un modul nou intră automat în CI doar dacă are
> `tests/test_*.py`. Dacă adaugi teste într-un modul care nu avea, asigură-te că există
> `tests/__init__.py` care le importă, altfel nu sunt descoperite.

> **Important pentru ramuri:** workflow-ul de teste rulează **doar** pe ramuri `*.0`. Pe o
> ramură de feature (ex. `claude/...`) rulează doar lint-ul. Pentru a vedea testele rulând,
> ținta PR-ului trebuie să fie o ramură `*.0` (ex. `19.0`).

---

## 6. Pre-commit și linting

Fișierul `.pre-commit-config.yaml` configurează:

- `trailing-whitespace`, `end-of-file-fixer` — igienă de fișiere;
- `check-yaml`, `check-xml` — validare YAML/XML (manifest, vederi, date);
- `debug-statements` — blochează `pdb`/`breakpoint` rămase în cod;
- `ruff` cu regulile **E722** (bare `except`), **F811** (redefinire), **F821** (nume
  nedefinit), **F632** (`is` cu literal).

Instalare și rulare:

```bash
pip install pre-commit
pre-commit install          # rulează automat la fiecare commit
pre-commit run --all-files  # rulare manuală pe tot proiectul
```

Aceleași reguli ruff rulează și în `ci.yml`, deci rularea locală a pre-commit previne
eșecurile de lint în CI.

---

## 7. Detaliu pe fișiere de test

### 7.1 `smart_contract/tests/test_parser.py` — `TestParser` (19 teste)

Testează inima funcțională a suitei: **parserul de variabile** bazat pe span-uri
`<span class="sc-var">`. `setUpClass` fixează parametrii de configurare și creează un partener
(`Acme Test SRL`) și un contract de tip vânzare. Helper-ul `_span()` construiește marcaje HTML,
iar `_make_element()` creează un `smart.contract.element` legat de contractul de test.

| Test | Ce verifică |
|---|---|
| `test_pull_contract_field` | Rezolvarea `contract.partner_id.name` → span devine `sc-var--resolved`, atributele `data-*` sunt eliminate |
| `test_pull_nested_relation` | `_resolve_var()` traversează relații imbricate |
| `test_blocked_env` | `ParsePathError` la accesul `contract.env` (câmp blocat) |
| `test_blocked_cr` | `ParsePathError` la accesul `contract._cr` |
| `test_blocked_via_parse_marks_unresolved` | Accesul blocat marchează span-ul `sc-var--unresolved`, păstrează `data-*`; eroarea apare în `_resolve_status()` |
| `test_add_on_contract_forbidden` | `ParsePathError` pentru sensul `add` pe câmpuri de contract (doar KV permis) |
| `test_method_not_in_allowlist` | `ParsePathError` dacă metoda nu e în allowlist |
| `test_method_allowed_after_param` | O metodă din allowlist (`set_name`) se apelează fără eroare |
| `test_depth_limit` | `ParsePathError` la depășirea adâncimii maxime |
| `test_field_blocklist` | `ParsePathError` pentru câmp aflat în blocklist |
| `test_kv_pull_attrelation` | Citirea KV `attrelation.rent` la nivel de contract |
| `test_kv_pull_localrelation` | Citirea KV `localrelation.note` la nivel de element |
| `test_kv_missing_returns_none` | Cheie KV inexistentă → `None` |
| `test_sync_add_placeholders_attrelation` | `_sync_add_placeholders()` creează KV de contract din span-uri `add`; idempotent |
| `test_sync_add_placeholders_localrelation` | Idem la nivel de element, cu maparea corectă a tipului |
| `test_status_counts` | `_resolve_status()` numără corect rezolvate/nerezolvate |
| `test_count_sc_var_spans` | `_count_sc_var_spans()` numără span-urile, ignoră `${}` vechi |
| `test_resolved_vs_unresolved_attrs` | Span-urile rezolvate pierd `data-*`, cele nerezolvate le păstrează |
| `test_contract_unresolved_count_compute` | `_compute_variable_status()` + `action_show_unresolved_variables()` |

### 7.2 `smart_contract/tests/test_sale.py` — `TestSale` (4 teste)

`setUpClass` creează un partener și o `sale.order`.

| Test | Ce verifică |
|---|---|
| `test_sm_action_create_contract` | `sm_action_create_contract()` creează un `smart.contract` legat |
| `test_action_open_smart_contract` | `action_open_smart_contract()` întoarce acțiunea cu domeniul corect |
| `test_action_add_attachment` | Stub (placeholder) — întoarce `True` |
| `test_action_send_contract_via_message_compose` | Stub (placeholder) — întoarce `True` |

> Ultimele două sunt **stub-uri** — locuri rezervate pentru teste viitoare. Recomandare:
> înlocuiește-le cu verificări reale (generarea atașamentului PDF, deschiderea wizard-ului).

### 7.3 `smart_contract/tools/test_legacy_migration.py` — `TestConvertLegacyText` (19 teste)

`unittest.TestCase` **fără** Odoo. Testează `convert_legacy_text()` / `convert_legacy_expr()`,
care transformă sintaxa veche `${...}` în span-uri `sc-var`. Acoperă:

- câmpuri de contract simple și imbricate (`${contract.partner_id.name}`);
- extragerea numelor de metode (`${contract.get_total()}`), inclusiv deduplicarea;
- maparea coloanelor pentru `attrelation`/`localrelation` la `data-type`
  (`number`→monetary, `number_int`→integer, `number_float`→float, `data`→datetime,
  `data_date`→date, necunoscut→text);
- cazuri „noop”: namespace necunoscut, text simplu, `${contract}` fără cale;
- intrări goale/`None`.

### 7.4 `smart_contract_gdpr` și `smart_contract_notification` (câte 2 teste)

Ambele creează un `smart.contract` cu `document_type` specific (`gdpr`, respectiv `notificare`)
și verifică cele două câmpuri calculate care impun seria obligatorie:

- `_compute_show_regulation()` → `True`;
- `_get_regulation_required_compute()` → `True`.

---

## 8. Acoperire și lacune cunoscute

**Bine acoperit:**

- Parserul de variabile (rezolvare, securitate: blocklist/allowlist/adâncime, KV, status).
- Migrarea sintaxei vechi `${...}`.
- Crearea contractului din comanda de vânzare (parțial).
- Câmpurile de control ale tipurilor GDPR/Notificare.

**Lacune (oportunități de teste noi):**

1. **`purchase_contract`** — generarea contractelor recurente OCA la semnare și blocarea
   facturării directe **nu** au teste. (Risc de business ridicat.)
2. **`smart_contract_upsale_downsale`** — execuția liniilor de abonament (Modificare),
   rezilierea și fluxul de Retenție **nu** au teste.
3. **`contract_overwrite`** — conversia valutară la facturare și „Total Opened Lines Amount”
   **nu** au teste.
4. **`smart_contract_blanket_order`** — atașarea automată a comenzilor din acord **nu** are teste.
5. **Fluxul de stări** (`Ready`/`Set Number`/`Sign`/`Cancel`) și **alocarea numărului** din
   serie nu au teste end-to-end dedicate.
6. **Raportul PDF** și **trimiterea pe e-mail** nu sunt testate (stub-uri în `test_sale.py`).

---

## 9. Cum scrii un test nou

### 9.1 Structura unui modul de test

```
smart_contract_upsale_downsale/
└── tests/
    ├── __init__.py          # from . import test_modificare
    └── test_modificare.py
```

`__init__.py`:

```python
from . import test_modificare
```

### 9.2 Șablon de test `TransactionCase`

```python
# -*- coding: utf-8 -*-
from odoo.tests import TransactionCase, tagged


@tagged('post_install', '-at_install')
class TestModificare(TransactionCase):

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.partner = cls.env['res.partner'].create({'name': 'Client Test SRL'})
        cls.parent = cls.env['smart.contract'].create({
            'partner_id': cls.partner.id,
            'contract_type': 'sale',
            'type': 'custommer',
        })

    def test_incarca_linii_abonament(self):
        act = self.env['smart.contract'].create({
            'partner_id': self.partner.id,
            'type': 'modificare',
            'parent_id': self.parent.id,
        })
        act.action_incarca_linii()           # metoda de business reală
        self.assertTrue(act.subscription_line_ids,
                        "Liniile de abonament ar trebui încărcate")
```

### 9.3 Bune practici (specifice acestui proiect)

- **Folosește `setUpClass`** pentru date partajate (mai rapid decât `setUp`).
- **Etichetează cu `post_install`** orice test care creează `smart.contract`.
- **Fixează configurarea** dacă testezi parserul: setează
  `smart_contract.parser.max_depth` / `method_allowlist` / `field_blocklist` în `setUpClass`,
  ca în `test_parser.py`.
- **Verifică efectul, nu doar lipsa erorii** — evită stub-urile care `return True`.
- **Mesaje de aserțiune** descriptive (al doilea argument la `assertTrue/assertEqual`).
- **Adaugă `tests/__init__.py`** ca testele să intre automat în detecția CI.
- Pentru parser, refolosește helper-ele `_span()` și `_make_element()` ca model.

### 9.4 Rularea testului nou

```bash
python /opt/odoo/odoo-bin --addons-path=...,. -d test_db \
  -u smart_contract_upsale_downsale \
  --test-enable --test-tags /smart_contract_upsale_downsale \
  --stop-after-init --log-level=test
```

---

## 10. Checklist înainte de merge

- [ ] `pre-commit run --all-files` trece (ruff + verificări XML/YAML).
- [ ] Toate cele 45 de teste existente trec.
- [ ] Testele noi sunt importate în `tests/__init__.py`.
- [ ] Testele noi sunt etichetate `post_install` dacă folosesc ORM-ul.
- [ ] PR-ul țintește o ramură `*.0` dacă vrei ca workflow-ul de teste să ruleze.
- [ ] Nu există stub-uri `return True` noi.
- [ ] Modulul OCA `contract` este disponibil în `addons-path` la rularea locală.

---

## 11. Depanarea testelor

| Simptom | Cauză probabilă | Soluție |
|---|---|---|
| `ParsePathError` neașteptat în teste | Allowlist/blocklist/adâncime moștenite din baza de date | Fixează parametrii în `setUpClass` (vezi `test_parser.py`) |
| Testul nu rulează în CI | Modulul nu are `tests/test_*.py` sau lipsește `__init__.py` | Adaugă fișierul de test și importul; verifică detecția din `test.yml` |
| `KeyError: 'smart.contract'` la `at_install` | Test rulat în timpul instalării | Adaugă `@tagged('post_install', '-at_install')` |
| Eroare la import `contract.*` | Modulul OCA `contract` lipsește din `addons-path` | Adaugă calea către `contract_oca` (vezi `--addons-path` din CI) |
| `_parse_status AttributeError` | Cod care folosește vechiul atribut eliminat | Folosește metoda `_resolve_status()` (vezi `PARSER_FIXES.md`) |
| Workflow-ul de teste nu pornește | Ramura nu se potrivește cu `*.0` | Țintește un PR către `19.0` sau altă ramură `*.0` |
| Lint eșuează în CI dar local nu | `pre-commit` nu este instalat local | `pip install pre-commit && pre-commit install` |

> **Referințe suplimentare:** `DEVELOPER_GUIDE.md` (rezumatul corecțiilor de parser),
> `smart_contract/PARSER_FIXES.md` (detaliu pe fiecare corecție și comenzi de verificare),
> `smart_contract/tests/test_parser.py` (exemple de utilizare a parserului).
