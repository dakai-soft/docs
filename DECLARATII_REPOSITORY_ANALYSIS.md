# 📋 DECLARATII REPOSITORY - ANALIZA COMPLETĂ
## Romanian Accounting & Tax Declarations System for Odoo 19

---

## 📑 CUPRINS

1. [Prezentare Generală](#prezentare-generală)
2. [Arhitectura Tehnică](#arhitectura-tehnică)
3. [Flowuri de Business](#flowuri-de-business)
4. [Module Detaliate](#module-detaliate)
5. [Fluxuri Tehnice Principale](#fluxuri-tehnice-principale)
6. [Dependențe și Integrări](#dependențe-și-integrări)
7. [Procese de Date](#procese-de-date)

---

## PREZENTARE GENERALĂ

### Ce este?
**declaratii** este o suită completă de **27 module Odoo 19** pentru gestionarea declarațiilor contabile și fiscale românești, develop de **Dakai SOFT SRL** și **adrian-dks**.

### Scopul Affacerilor
Permite companiile românești să:
- ✅ Genereze declarații fiscale (D100, D101, D112, D300, D390, D394)
- ✅ Gestioneze contabilitate corespundent
- ✅ Raporteze după standarde SAFT/E-Factura
- ✅ Automatizeze procesele contabile
- ✅ Integreze cu sisteme guvernamentale (REGES)

### Stack Tehnologic
```
┌─────────────────────────────────────────┐
│         ODOO 19 ERP FRAMEWORK           │
├─────────────────────────────────────────┤
│  Backend:  Python 3.8+ (SQLAlchemy ORM) │
│  Frontend: OWL Components (JS)          │
│  Database: PostgreSQL                   │
│  Reports:  XML Templates + PDF Engine   │
│  API:      REST (Odoo native)           │
└─────────────────────────────────────────┘
```

---

## ARHITECTURA TEHNICĂ

### Piramida Modulelor

```
┌─────────────────────────────────────────────────────────────┐
│ LAYER 4: RAPOARTE & RAPORTĂRI (Export XML, PDF, Web)       │
│  • dakai_declarations_to_xml                                │
│  • l10n_ro_export_saga                                      │
│  • dakai_account_report_journal                             │
├─────────────────────────────────────────────────────────────┤
│ LAYER 3: DECLARAȚII FISCALE SPECIALE (D100, D300, D394)    │
│  • dakai_d100 (VAT Return - Declarația 100)                 │
│  • dakai_d112 (Payroll - Declarația 112)                    │
│  • dakai_d300 (Product Accounting)                          │
│  • dakai_d390 (Multi-Operation)                             │
│  • dakai_d394 (Invoice List - Declarația 394)               │
├─────────────────────────────────────────────────────────────┤
│ LAYER 2: LOCALIZĂRI ȘI EXTENSII (Romanian GAAP)            │
│  • l10n_ro_corresponding_account  ⭐ (Cont Corespondent)    │
│  • l10n_ro_account_asset          ⭐ (Active Imobilizate)   │
│  • l10n_ro_currency_revaluation   ⭐ (Revaluări valute)     │
│  • l10n_ro_saft_fix               (Rapoarte SAFT)          │
├─────────────────────────────────────────────────────────────┤
│ LAYER 1: FUNDAȚIE COMUNĂ (Models Base, Config, Security)    │
│  • dakai_declarations_common (Model Base)                   │
│  • l10n_ro_config (Romanian Localization Base)              │
└─────────────────────────────────────────────────────────────┘
```

### Structura unui Module Tipic

```
MODULE_NAME/
├── __manifest__.py           # Metadate (versiune, dependen., descriere)
├── __init__.py              # Import-uri module
├── models/
│   ├── __init__.py
│   ├── model_principal.py   # Model declarație
│   └── ...alte modele
├── views/
│   ├── model_form.xml       # Formular de editare
│   ├── model_list.xml       # Vedere listă
│   └── ...alte views
├── data/
│   ├── data.xml             # Date inițiale
│   └── nomenclator.csv      # Tabele de referință
├── security/
│   ├── ir.model.access.csv  # ACL (Access Control Lists)
│   └── security.xml         # Reguli de securitate
├── report/
│   ├── report_action.xml    # Definiție raport
│   └── report_template.xml  # Template PDF/HTML
├── wizard/
│   ├── wizard_model.py      # Workflow interactiv
│   └── wizard_view.xml      # UI wizard
├── static/src/
│   ├── js/                  # OWL Components (JavaScript)
│   ├── scss/                # Stiluri
│   └── xml/                 # Template-uri OWL
├── migrations/
│   └── [version]/pre-migration.py  # Migrare date
├── tests/
│   └── test_*.py            # Unit & integration tests
└── i18n/
    └── *.po                 # Traduceri
```

---

## FLOWURI DE BUSINESS

### FLUX PRINCIPAL: Generare Declarație Fiscală

```
1️⃣  INTRARE
    └─ Utilizator deschide modul declarație (ex: D394)
       
2️⃣  CITIRE TRANZACȚII
    └─ Sistem citește din:
       • account.move (Jurnale contabile)
       • account.move.line (Înregistrări individuale)
       • product.category (Clasificare produse)
       • partner.invoice (Furnizori/Clienți)
       
3️⃣  PROCESARE ȘI CLASIFICARE
    ├─ Filtrare tranzacții după perioadă
    ├─ Clasificare după tip operație
    │  • D100: Clasificare VAT (Intracomunitar, Import, etc.)
    │  • D394: Clasificare după tip factură (Emise/Primite)
    │  • D300: Clasificare produse după categorie
    └─ Validare și reconciliare
    
4️⃣  CALCUL ȘI AGREGARE
    ├─ Sumări per clasă/tip
    ├─ Calcul TVA (bază imponibilă vs. impozit)
    ├─ Reconciliare cu conturile analitice
    └─ Aplicare reguli speciale (neexigibilitate, etc.)
    
5️⃣  STOCARE REZULTATE
    └─ Salvare în:
       • dakai_d100 (D100.line_ids)
       • dakai_d394 (D394.facturi_ids, D394.lista_ids)
       • dakai_d300 (D300.line_ids)
       
6️⃣  EXPORT
    ├─ OPȚIUNE 1: Export XML (pentru trimitere ANAF)
    │  • Transform din model Odoo → XML SAGA
    │  • Download comprescat
    ├─ OPȚIUNE 2: Tipar PDF (pentru documente)
    │  • Render template raport
    │  • Tipar/Save PDF
    └─ OPȚIUNE 3: Vedere Web (pentru revizuire)
       • Afișare date în tabele OWL
       • Editare linie cu linie
```

### FLUX SECUNDAR: Contabilitate Corespundent (l10n_ro_corresponding_account)

```
👤 UTILIZATOR INTRA JURNAL CONTABIL
         ↓
📝 CREAZĂ ÎNREGISTRARE CONTABILĂ (account.move)
    ├─ Debit: 401 (Furnizor)
    ├─ Credit: 512 (Bancă)
    └─ Înregistrare salvată
         ↓
🤖 SISTEM AUTO-CALCUL
    ├─ Detectează: "Înregistrare simplă 1xN sau Nx1"
    ├─ CALCUL INTELIGENT → Distribuție pe conturi corespondente
    │  Algoritm:
    │  1. Identifică conturile correspondenți (403, 4111, 404 etc.)
    │  2. Calculează factori de distribuție
    │  3. Distribuie proporțional pe lini
    └─ Salvează distribuție automat
         ↓
👁️ AFIȘARE & VALIDARE
    ├─ OWL Widget (responsive, editable)
    │  ├─ Tabel cu distribuție
    │  ├─ Valori editabile inline
    │  └─ Validare real-time
    └─ Avertizări sistemate:
       Level 1: Filtru (ascunde anumite înregistrări)
       Level 2: Banner (avertisment vizibil)
       Level 3: Chatter (mesaj comentariu)
         ↓
📊 RAPORT
    ├─ Raport Web: "Jurnal cu Corespondent"
    │  └─ Afișare dinamică pe ecran
    └─ Raport PDF: "Fișă de Cont"
       └─ Tipar pentru arhivă
```

### FLUX TERȚIAR: Gestionare Active Imobilizate

```
➕ ADAUGARE ACTIV
    └─ Crează record în account.asset
       ├─ Descriere, cost de achiziție
       ├─ Durată de depreciere (Lege fiscală RO)
       └─ Cont de depreciere
         ↓
📅 CALCUL AMORTIZARE
    ├─ Sistem cron job (lunar/anual)
    ├─ Calcul automat conform normelor RO
    └─ Generare înregistrări contabile
         ↓
📋 RAPORT
    └─ Raport: "Active Imobilizate"
       ├─ Valoare brută
       ├─ Amortizare cumulată
       └─ Valoare netă
```

---

## MODULE DETALIATE

### 🔴 LAYER 1: FUNDAȚIE COMUNĂ

#### 1. **dakai_declarations_common** (v19.0.0.0.1)
**Rol**: Bază comună pentru toate declarațiile

**Modele Principale**:
```python
class CommonDeclaration(models.Model):
    """Model base pentru toate declarațiile"""
    _name = 'dakai.common.declaration'
    
    company_id          # Companie
    fiscal_year_id      # An fiscal
    declaration_type    # Tip declarație (D100, D300, etc.)
    state               # Status (draft, submitted, validated)
    declaration_date    # Data declarației
    xml_content         # Conținut XML generat

class CountryState(models.Model):
    """State-uri administrative românești"""
    _name = 'res.country.state'
    # Contains: 41 județe + București
    
class CompanyDeclarationConfig(models.Model):
    """Configurație per companie pentru declarații"""
    company_id
    vat_regime          # Regim TVA (Normal, Apel, etc.)
    decimal_places      # Precizie calcule
```

**Fluxuri**:
```
1. Creare înregistrare în dakai.common.declaration
2. Setare parametri declarație (perioadă, companie)
3. Linkare la modul specific (D100, D300, etc.)
4. Validare și blocaj modificări după trimitere
```

**Fișiere Cheie**:
- `models/common_declaration.py` - Model base
- `data/states.csv` - 41 județe + București
- `security/ir.model.access.csv` - Permiții de acces

---

### 🟡 LAYER 2: LOCALIZĂRI ȘI EXTENSII SPECIALE

#### 2. **l10n_ro_corresponding_account** ⭐ (v19.0.1.0.0)
**Rol**: Implementează conceptul românesc de "Cont Corespondent"

**Concept de Business**:
> În contabilitatea românească, pentru o înregistrare simplă cu mai mult conturi de debit/credit, se necesită un "cont corespondent" care arată repartiția fondurilor între furnizori/clienți individuali.

**Modele Principale**:
```python
class AccountMoveLineCorrespondentDist(models.Model):
    """Distribuție cont corespondent"""
    _name = 'account.move.line.correspondent.dist'
    
    account_move_id     # Înregistrare contabilă
    partner_id          # Furnizor/Client
    amount              # Valoare distribuită
    percentage          # % din total
    
class AccountCorrespondentDistributionRule(models.Model):
    """Reguli de distribuție"""
    company_id
    distribution_type   # 'proportional', 'manual'
    auto_calculate      # Calcul automat?
```

**Fluxuri**:
```
CALCUL AUTOMAT:
1. Utilizator salvează journal entry
2. System trigger detectează: entry.line_count == (1, N) sau (N, 1)
3. Dacă enabled: CALCUL AUTOMAT
   ├─ Identifică conturi 40x (Furnizori)
   ├─ Calculează proporție per partener
   └─ Crează înregistrări în correspondent_dist
4. Display în OWL widget cu opțiune de editare

VALIDARE:
├─ Suma distribuții = Total înregistrare
├─ Toate liniile au distribuție
└─ Nu există valori negative (dacă enabled)

RAPORTARE:
├─ Raport Web: Jurnal cu Corespondent
│  └─ Afișare dynamic cu filtru dată
└─ Raport PDF: Fișă de Cont
   └─ Export per jurnal cu paginare
```

**Fișiere Importante**:
- `models/correspondent.py` - Model distribuție
- `static/src/js/corresponding_account_widget.js` - OWL widget
- `report/report_fisa_cont_template.xml` - Template PDF
- `doc/business_requirements.md` - Specificații detaliate

**Dependențe**:
```
account → account.move, account.move.line
product → product.category
```

---

#### 3. **l10n_ro_account_asset** (v19.0.1.0.1)
**Rol**: Gestionarea activelor imobilizate după GAAP român

**Concept**:
- Activele cu durata de amortizare specifică legii: 3, 5, 10, 20+ ani
- Reevaluări anuale pe bază de indici INSTATEC

**Modele Principale**:
```python
class AccountAsset(models.Model):
    """Active imobilizate - extensie"""
    category_id         # Categorie activ (Cladiri, Mașini, etc.)
    asset_model_id      # Model normalizat RO
    acquisition_date    # Data achiziție
    gross_value         # Valoare brută
    cumulative_deprec   # Amortizare cumulată
    salvage_value       # Valoare reziduală
    useful_life_years   # Anos utili
```

**Categorie de Active (din CSV)**:
```
Cladiri si constructii (20 ani)
Masini si utilaje (10 ani)
Computere (3 ani)
Mijloace de transport (5 ani)
Mobilier si aparatura (5 ani)
```

**Fluxuri**:
```
1. Creare activ → Selectare categorie → Auto-set durata
2. Cron job lunar:
   ├─ Calcul amortizare lunară
   ├─ Creare automat journal entry
   │  ├─ Debit: 6813 (Cheltuieli cu amortizare)
   │  └─ Credit: 280x (Amortizare cumulată)
   └─ Update cumulative depreciation
3. Raport anual: Fișă active (brută, amortizare, netă)
```

**Integrări cu Contabilitate**:
```
account.asset → account.move (auto-generat)
           ↓
    acru_expense → reconciliare automată
```

---

#### 4. **l10n_ro_currency_revaluation** (v19.0.1.0.0)
**Rol**: Revaluare conturi în mai valute pe bază de partner

**Concept Business**:
- Conform IAS 21: Diferente curs apar la datoriile în valute
- Particularitate RO: Revaluare PER PARTENER, nu global

**Modele**:
```python
class AccountMulticurrencyRevaluationWizard(models.Model):
    """Wizard revaluare multi-valută"""
    currency_id         # Valuta de revaluare
    rate_date          # Data curs
    rate                # Curs de schimb
    partner_ids         # Parteneri afectați (401, 4111, etc.)
```

**Fluxuri**:
```
1. User: Deschide Wizard "Revaluare Multi-valută"
2. Selectare:
   ├─ Valuta
   ├─ Data curs
   ├─ Parteneri (Class 4: 401, 4111, 4121, etc.)
   └─ Curs de schimb nou
3. System calcul:
   ├─ Pentru FIECARE partener:
   │  ├─ Sold actual în valută
   │  ├─ Sold actual în RON (curs vechi)
   │  ├─ Sold nou în RON (curs nou)
   │  └─ Diferență = Diferență de curs
   └─ Creare journal entries PE PARTENER (nu global!)
       ├─ Debit/Credit: 401 (Furnizor)
       ├─ Credit/Debit: 765 (Câștiguri curs) / 665 (Pierderi curs)
       └─ Ref: Partner + Valuta
4. Export: Jurnal cu repartiție pe parteneri
```

**Diferență vs. Odoo Standard**: 
❌ Standard: 1 entry global
✅ Noi: N entries (una per partener) → Audit trail detaliat

---

#### 5. **l10n_ro_account_move_line_negative** (v19.0.0.0.0)
**Rol**: Validare lini cu valori negative

**Logică**:
```
Permitere linii negative numai în contexte specifice:
✅ Storn facturi
✅ Ajustări contabile autorizate
❌ În caz opus: Avertisment + Blocare (config)
```

**Implementare**:
```python
def validate_negative_line(self):
    if self.amount < 0:
        if not self.move_id.allow_negative:
            raise ValidationError("Lini negative nu sunt permise!")
```

---

#### 6. **l10n_ro_saft_fix** (v19.0.0.0.0)
**Rol**: Corecții raport SAFT (Standard Audit File for Tax)

**Care sunt problemele SAFT în Odoo standard?**
```
❌ Clasificare incorrect pe operații
❌ Excludere documente stornate
❌ Agregare greșită per tip cont
❌ Lipsă referință jurnale

FIXES IMPLEMENTATE:
✅ Reclasificare pe move_type (out_invoice vs. in_invoice)
✅ Excludere complet stornate (state = 'cancel')
✅ Agregare corect per jurnal + tip cont
✅ Paginare și filtrare per perioadă
```

**Format SAFT: XML cu structură**
```xml
<AuditFile>
  <Header>...</Header>
  <MasterFiles>
    <Customers/> <Suppliers/>
  </MasterFiles>
  <GeneralLedgerEntries>
    <Journal>
      <Transaction>
        <Line/>
      </Transaction>
    </Journal>
  </GeneralLedgerEntries>
</AuditFile>
```

---

### 🟢 LAYER 3: DECLARAȚII FISCALE SPECIALE

#### 7. **dakai_d100** - D100 Declaration (VAT Return)
**Rol**: Generare Declarație 100 (Declarația privind taxa pe valoarea adăugată)

**Concept Business**:
- Raport lunar/trimestrial de TVA
- Clasificare operații: Intracomunitar, Import, Operații taxabile, etc.
- Calcul TVA: Bază imponibilă × Cot (19%, 9%, 5%, 0%)

**Modele**:
```python
class DakaiD100(models.Model):
    """Declarația 100"""
    company_id
    fiscal_period       # Luna/Trimestru
    fiscal_year
    
    d100_nomenclator_ids  # Rânduri clasificare TVA
    
class DakaiD100Nomenclator(models.Model):
    """Linie TVA per clasificare"""
    d100_id
    classification      # Ex: "Intraco", "Import", "Operații taxabile"
    basis              # Bază imponibilă
    vat_rate           # Cot TVA (19%, 9%, etc.)
    vat_amount         # TVA = basis × rate
    total             # Basis + VAT
```

**Fluxuri**:
```
1. User: Creare D100 pentru perioada (ex: Februarie 2024)
2. System: Auto-citire din account.move
   ├─ Filter by date range
   ├─ Filter by invoice type (customer vs. supplier)
   └─ Group by VAT classification
3. Processing:
   ├─ Pentru FIECARE clasificare:
   │  ├─ Sum basis amount
   │  ├─ Sum VAT
   │  └─ Create D100.nomenclator line
   └─ Cross-check: Total VAT = Jurnal contabil
4. UI: Tabel cu rânduri clasificare
   ├─ Edit permis pe basis/rate
   ├─ VAT auto-recalculat
   └─ Validare: Total ≤ Actual journal amount
5. Export:
   ├─ XML SAGA format (ANAF)
   ├─ PDF Raport
   └─ CSV pentru excel
```

**Data Flow**:
```
account.move (Invoices)
         ↓
   l10n_ro_move_tax (Tax lines)
         ↓
  Clasificare automat per tipo
         ↓
   D100.nomenclator (Agregat)
         ↓
  Export XML/PDF
```

---

#### 8. **dakai_d112** - Payroll Declarations
**Rol**: Declarații salariale (contribuții, impozit venit)

**Concept**:
- Raport lunar de salarii + contribuții (CASS, CAS, AJFOR)
- Impozit pe venit: 10%

**Modele**:
```python
class DakaiD112(models.Model):
    """Declarație salarii"""
    employee_ids        # Angajați
    salary_month
    gross_salary        # Salariu brut
    contributions       # Calcul CASS (8%), CAS (25%), AJFOR (0.15%)
    income_tax          # 10% din impozabil
    net_salary          # Salariu net
```

**Integrare cu HR**:
```
hr.employee ← Link cu D112
      ↓
salary.structure ← Definiție salaru
      ↓
hr.payslip (Fișă salariu) ← Auto-generat
      ↓
D112 (Raport ANAF) ← Auto-agregat
```

---

#### 9. **dakai_d300** - Product Accounting Report
**Rol**: Raport contabil per categorie produs

**Concept**:
- Analiză vânzări/achiziții per categorie
- Valorizare stoc (FIFO/LIFO/medie)

**Modele**:
```python
class DakaiD300(models.Model):
    """Raport D300"""
    company_id
    fiscal_period
    
    line_ids            # Per categorie produs
    
class DakaiD300Line(models.Model):
    """Linie per produs"""
    product_category_id  # Categoria
    opening_balance      # Stoc inițial
    purchases            # Achiziții
    sales                # Vânzări
    closing_balance      # Stoc final
```

---

#### 10. **dakai_d390** - Multi-Operation Declaration
**Rol**: Declarație cu mai mult tipuri de operații (Intraco, Import, etc.)

**Complex**: ✅ Cel mai complex dintre D*
- Clasificare automată pe 8+ tipuri operații
- Calcul TVA diferențiat per tip
- Reconciliare cu jurnale

---

#### 11. **dakai_d394** ⭐ (MOST COMPLEX) - Invoice List & VAT Summary
**Rol**: Lista facturilor emise și primite + Rezumat TVA

**Concept Business**:
- Lista detaliată de FIECARE factură (nr., dată, partener, valoare, TVA)
- Sumar TVA pe clasificări
- Obligator pentru export ANAF

**Modele Principale** (40 fișiere, 6 test-uri):
```python
class DakaiD394(models.Model):
    """Declarația 394 - Master record"""
    company_id
    fiscal_period       # Luna/Trimestru
    fiscal_year
    
    # Relații
    facturi_ids        # RO: Facturile emise ← RelOne2Many
    facturi_primite_ids  # Facturile primite
    lista_ids          # Lista detaliată
    operation_ids      # Operații per linie
    rezumat_ids        # Sumar TVA
    
class DakaiD394Facturi(models.Model):
    """Factură EMISĂ"""
    d394_id
    invoice_id          # Link la account.move
    number              # Nr. factură
    issue_date          # Data emitere
    partner_id          # Client
    gross_amount        # Valoare brută
    vat_amount          # TVA
    net_amount          # Valoare netă
    
class DakaiD394FacturiPrimite(models.Model):
    """Factură PRIMITĂ"""
    # Similar cu DakaiD394Facturi
    
class DakaiD394Lista(models.Model):
    """Linie în lista detaliată"""
    d394_id
    invoice_id
    sequence_no
    # Datele de facturi
    
class DakaiD394Operation(models.Model):
    """Operație (tip: Intraco, Import, etc.)"""
    d394_id
    operation_type      # Ex: "Intraco-EU"
    operation_desc
    basis_amount
    vat_rate
    vat_amount
    
class DakaiD394Rezumat(models.Model):
    """Rezumat TVA per clasificare"""
    d394_id
    classification
    total_basis
    total_vat
```

**Fluxuri Complex**:
```
STEP 1: CREARE D394
  └─ User selectează perioada (ex: Februarie 2024)
  
STEP 2: AUTO-CITIRE FACTURI
  ├─ Query: account.move WHERE (move_type IN ('out_invoice', 'in_invoice')
  │                      AND date >= period_start
  │                      AND date <= period_end)
  ├─ Pentru FIECARE factură:
  │  ├─ Creare DakaiD394Facturi/FacturiPrimite record
  │  ├─ Copy: nr., dată, partener, valori
  │  └─ Link: reference la account.move
  └─ Total: 50-500+ factură procesate
  
STEP 3: CLASIFICARE OPERAȚII
  ├─ Sistem detectează automat pe bază:
  │  ├─ Curs de schimb (Intraco vs. Extraco)
  │  ├─ Cod INTRASTAT (produs vs. serviciu)
  │  ├─ Cot TVA (19%, 9%, 5%, 0%)
  │  └─ Origine partener (EU vs. Non-EU)
  │
  └─ Creare DakaiD394Operation record per clasificare
  
STEP 4: AGREGARE ȘI SUMARE
  ├─ Group BY operation_type
  ├─ SUM basis, vat_amount
  └─ Creare DakaiD394Rezumat records
  
STEP 5: VALIDĂRI
  ├─ Check 1: Σ(D394.facturi) = Jurnal contabil (account_move)
  ├─ Check 2: Σ(vat) = Declarație TVA (D100)
  ├─ Check 3: Nu lipsesc factură
  └─ Avertisment dacă discrepanțe
  
STEP 6: UI EDITARE
  ├─ Tab 1: Facturi Emise
  │  └─ Tabel cu toate facturile cu opțiune edit
  ├─ Tab 2: Facturi Primite
  │  └─ Similar
  ├─ Tab 3: Lista Detaliată
  │  └─ View din D394Lista records
  ├─ Tab 4: Operații
  │  └─ Reclasificare manual dacă trebuie
  └─ Tab 5: Rezumat
     └─ Sumar TVA per operație
     
STEP 7: EXPORT
  ├─ Controller: dakai_d394/controller/
  ├─ XML SAGA generation
  ├─ Compresare + signing (optional)
  └─ Download sau email ANAF
```

**Fișiere Cheie**:
```
dakai_d394/
├── models/
│   ├── dakai_d394.py (Master)
│   ├── dakai_d394_facturi.py (Emise)
│   ├── dakai_d394_facturi_primite.py (Primite)
│   ├── dakai_d394_lista.py (Detaliu)
│   ├── dakai_d394_operation.py (Clasificare)
│   └── dakai_d394_rezumat.py (Sumar)
├── views/
│   ├── dakai_d394_form.xml (Form principal)
│   ├── dakai_d394_facturi_tree.xml (Facturi list)
│   └── ...
├── controller/
│   └── main.py (XML export logic)
└── tests/
    ├── test_d394.py
    ├── test_d394_facturi.py
    ├── test_d394_operations.py
    └── test_d394_export.py
```

---

### 🔵 LAYER 4: RAPOARTE & EXPORT

#### 12. **dakai_declarations_to_xml**
**Rol**: Export declarații în format XML SAGA

**Controller Endpoints**:
```python
class DakaiDeclarationsXmlController(http.Controller):
    
    @http.route('/dakai/d394/export/xml', type='http')
    def d394_export_xml(self, d394_id, **post):
        """Export D394 to XML"""
        d394 = request.env['dakai.d394'].browse(d394_id)
        xml_content = d394._generate_xml_saga()
        return request.make_response(
            xml_content,
            headers=[('Content-Disposition', 'attachment; filename=D394.xml')]
        )
```

**Transformări Date → XML**:
```
D394 Model → XML Element
├─ d394.facturi_ids → <Facturi>...</Facturi>
├─ d394.rezumat_ids → <Rezumat>...</Rezumat>
└─ d394.operation_ids → <Operatii>...</Operatii>

Encoding: UTF-8
Signing: Optional certificate-based
Compression: ZIP (optional)
```

---

### 🟣 ALTE MODULE IMPORTANTE

#### 13. **dakai_partner_ledger**
**Rol**: Raport partener detaliat cu extensii RO

**Features**:
- OWL Widget: Search bar cu auto-complete
- Sumar pe perioadă
- Export CSV/Excel

---

#### 14. **dakai_reges**
**Rol**: Integrare cu sistem REGES (governo RO - payroll)

**REGES = Register Electronic al Salariaților**
- Transmisie automată date angajați
- Calcul contribuții
- Integrare CAEN

**Wizard**: `reges_export_wizard` → Click → Export la REGES

---

#### 15. **dakai_neexigibility**
**Rol**: Operații fără obligație de TVA (neexigibile)

**Concept**:
- Export: 0% TVA (conform Cod Fiscal)
- Agricultori sub limită: Neexigibil
- Servicii financiare: Neexigibil

---

---

## FLUXURI TEHNICE PRINCIPALE

### 🔄 FLUX TEHNIC #1: Generare D394 din Journal

```python
# TRIGGER: Utilizator click "Generate D394"
def action_generate_d394():
    
    # STEP 1: Citire factură din account.move
    moves = env['account.move'].search([
        ('move_type', 'in', ['out_invoice', 'in_invoice']),
        ('state', '=', 'posted'),
        ('date', '>=', d394.period_start),
        ('date', '<=', d394.period_end),
        ('company_id', '=', d394.company_id.id),
    ])
    # Result: 100+ înregistrări
    
    # STEP 2: Procesare per factură
    for move in moves:
        
        # STEP 2a: Creare Facturi record
        facturi_record = env['dakai.d394.facturi'].create({
            'd394_id': d394.id,
            'invoice_id': move.id,
            'number': move.name,
            'issue_date': move.date,
            'partner_id': move.partner_id.id,
            'gross_amount': sum(line.debit - line.credit for line in move.line_ids),
            'vat_amount': move._get_vat_amount(),  # Suma tax lines
            'net_amount': move.amount_untaxed,
        })
        
        # STEP 2b: Criere Operation record per clasificare
        for tax_line in move.tax_line_ids:
            operation = env['dakai.d394.operation'].create({
                'd394_id': d394.id,
                'facturi_id': facturi_record.id,
                'operation_type': tax_line._compute_operation_type(),
                'basis_amount': tax_line.base_amount,
                'vat_rate': tax_line.tax_id.amount,
                'vat_amount': tax_line.amount,
            })
    
    # STEP 3: Agregare Rezumat
    grouped = moves.read_group([
        ('d394_id', '=', d394.id),
    ], ['operation_type', 'basis_amount:sum', 'vat_amount:sum'], 
    ['operation_type'])
    
    for group in grouped:
        env['dakai.d394.rezumat'].create({
            'd394_id': d394.id,
            'operation_type': group['operation_type'],
            'total_basis': group['basis_amount'],
            'total_vat': group['vat_amount'],
        })
    
    # STEP 4: Validări
    d394._validate_reconciliation()  # Verifică ≈ jurnal
    
    return d394
```

---

### 🔄 FLUX TEHNIC #2: Auto-Calcul Cont Corespondent

```javascript
// File: corresponding_account_new_widget.js
// OWL Component (Odoo 19)

class CorrespondingAccountWidget extends Component {
    
    setup() {
        this.state = useState({
            distributions: [],
            total: 0,
            editMode: false,
        });
    }
    
    // TRIGGER: Creare sau edit journal entry
    async onMoveChange() {
        const moveData = await this.fetchMoveData(this.props.move_id);
        
        // STEP 1: Detectare tip înregistrare
        const lineCount = moveData.line_ids.length;
        
        if (this._isSimpleEntry(lineCount)) {
            // STEP 2: Calcul automat distribuție
            const distributions = this._calculateDistribution(moveData);
            
            // STEP 3: Criere înregistrări în bază (via RPC call)
            await this.rpc({
                model: 'account.move.line.correspondent.dist',
                method: 'create_from_move',
                args: [this.props.move_id, distributions],
            });
            
            // STEP 4: Display în widget
            this.state.distributions = distributions;
            this.state.total = this._sumDistributions(distributions);
        }
    }
    
    _calculateDistribution(moveData) {
        // ALGORITM: Distribuție proporțională
        
        const suppliers = moveData.line_ids.filter(
            l => l.account_id.code.startsWith('40')
        ); // Class 4: Furnizori
        
        const totalAmount = moveData.amount_total;
        
        return suppliers.map(supplier => ({
            partner_id: supplier.partner_id,
            amount: supplier.balance,  // Simplified
            percentage: (supplier.balance / totalAmount * 100).toFixed(2),
        }));
    }
    
    onDistributionEdit(rowIndex, newAmount) {
        // STEP 5: Edit inline (validare real-time)
        
        const row = this.state.distributions[rowIndex];
        row.amount = newAmount;
        row.percentage = (newAmount / this.state.total * 100).toFixed(2);
        
        // Validare: Suma = Total
        const sumCheck = this._sumDistributions(this.state.distributions);
        
        if (Math.abs(sumCheck - this.state.total) < 0.01) {
            // ✅ OK
            this._markValid();
        } else {
            // ❌ ERROR
            this._markError(`Total ${sumCheck.toFixed(2)} != ${this.state.total}`);
        }
    }
}
```

---

### 🔄 FLUX TEHNIC #3: XML Export cu Compresare

```python
# File: dakai_d394/controller/main.py

class D394XmlController(http.Controller):
    
    @http.route('/dakai/d394/<int:d394_id>/export/xml', type='http')
    def export_d394_xml(self, d394_id):
        """Export D394 to XML SAGA"""
        
        d394 = request.env['dakai.d394'].browse(d394_id)
        
        # STEP 1: Validare
        if d394.state != 'validated':
            return json({'error': 'D394 must be validated first'})
        
        # STEP 2: Generare XML
        xml_content = self._build_xml_saga(d394)
        
        # STEP 3: Compresare (optional)
        if d394.company_id.compress_xml_export:
            import zipfile, io
            zip_buffer = io.BytesIO()
            with zipfile.ZipFile(zip_buffer, 'w') as zf:
                zf.writestr('D394.xml', xml_content.encode('utf-8'))
            xml_bytes = zip_buffer.getvalue()
            filename = 'D394.zip'
        else:
            xml_bytes = xml_content.encode('utf-8')
            filename = 'D394.xml'
        
        # STEP 4: Signing (optional certificate)
        if d394.company_id.sign_xml_export:
            xml_bytes = self._sign_xml(xml_bytes, d394.company_id.certificate)
        
        # STEP 5: Response
        return request.make_response(
            xml_bytes,
            headers=[
                ('Content-Type', 'application/xml' if filename.endswith('.xml') else 'application/zip'),
                ('Content-Disposition', f'attachment; filename={filename}'),
            ]
        )
    
    def _build_xml_saga(self, d394):
        """Build XML structure"""
        
        root = etree.Element('DeclaratiaSAGA')
        
        # Header
        header = etree.SubElement(root, 'Header')
        etree.SubElement(header, 'CIF').text = d394.company_id.vat
        etree.SubElement(header, 'Period').text = d394.fiscal_period
        etree.SubElement(header, 'Year').text = str(d394.fiscal_year)
        
        # Facturi Emise
        facturi_elem = etree.SubElement(root, 'FacturiEmise')
        for factura in d394.facturi_ids:
            f_elem = etree.SubElement(facturi_elem, 'Factura')
            etree.SubElement(f_elem, 'Number').text = factura.number
            etree.SubElement(f_elem, 'Date').text = str(factura.issue_date)
            etree.SubElement(f_elem, 'Partner').text = factura.partner_id.name
            etree.SubElement(f_elem, 'GrossAmount').text = str(factura.gross_amount)
            etree.SubElement(f_elem, 'VATAmount').text = str(factura.vat_amount)
        
        # Rezumat
        rezumat_elem = etree.SubElement(root, 'Rezumat')
        for rezumat in d394.rezumat_ids:
            r_elem = etree.SubElement(rezumat_elem, 'Operation')
            etree.SubElement(r_elem, 'Type').text = rezumat.operation_type
            etree.SubElement(r_elem, 'TotalBasis').text = str(rezumat.total_basis)
            etree.SubElement(r_elem, 'TotalVAT').text = str(rezumat.total_vat)
        
        return etree.tostring(root, pretty_print=True, encoding='unicode')
```

---

## DEPENDENȚE ȘI INTEGRĂRI

### 📊 Grafic Dependențe Module

```
dakai_declarations_common (BASE)
    ↓
    ├─→ dakai_d100 (VAT)
    ├─→ dakai_d101
    ├─→ dakai_d112 (Payroll)
    ├─→ dakai_d300 (Products)
    ├─→ dakai_d390 (Multi-Op)
    └─→ dakai_d394 ★ (MASTER)
         ├─→ dakai_declarations_to_xml (Export)
         └─→ dakai_account_report_journal
         
l10n_ro_config (Base RO localization)
    ↓
    ├─→ l10n_ro_corresponding_account ★★ (COMPLEX)
    ├─→ l10n_ro_account_asset
    ├─→ l10n_ro_currency_revaluation
    ├─→ l10n_ro_saft_fix
    ├─→ l10n_ro_stock_agging
    ├─→ l10n_ro_deferred
    └─→ l10n_ro_efactura_enhancements
    
Odoo Core Modules
    ├─→ account (Contabilitate)
    ├─→ account_asset (Active)
    ├─→ account_reports (Rapoarte)
    ├─→ account_stock (Stocuri)
    ├─→ product (Produse)
    ├─→ hr (Resurse Umane)
    ├─→ stock (Inventar)
    ├─→ analytic (Conturi Analitice)
    └─→ web (Framework OWL/JS)
```

### 🔌 Integrări Externe

```
┌──────────────────────────────────────────┐
│          DECLARATII REPOSITORY           │
├──────────────────────────────────────────┤
│  ↓ Export XML                            │
│  ANAF (Romanian Tax Authority)           │
│  └─ URL: https://www.anaf.ro/           │
│                                          │
│  ↓ REGES Export                          │
│  REGES Portal (Payroll System)           │
│  └─ Romanian HR government system        │
│                                          │
│  ↓ E-factura                             │
│  E-Faturat Platform (Electronic Invoice) │
│  └─ Romanian e-invoice mandate           │
│                                          │
│  ↓ SAFT Reports                          │
│  Audit Trail Files (for auditors)        │
│  └─ Standard format: XML                 │
└──────────────────────────────────────────┘
```

---

## PROCESE DE DATE

### 💾 Fluxul Datelor: Journal → Declarație → Export

```
┌──────────────────────────────────────────────────────────────────────┐
│  NIVEL 1: INTRARE DATE (Data Input Layer)                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Utilizator crează:                                                 │
│  ├─ account.move (Jurnal contabil)                                  │
│  │  └─ Linie: Debit 401 (Furnizor) vs. Credit 512 (Bancă)          │
│  │                                                                   │
│  ├─ account.invoice (Factură client/furnizor)                       │
│  │  └─ Include: Lini produs + Tax lines (TVA 19%, 9%, etc.)        │
│  │                                                                   │
│  └─ hr.payslip (Fișă salariu)                                       │
│     └─ Include: Salariu + Contribuții (CASS, CAS, etc.)            │
│                                                                       │
│  [Postare în bază]                                                  │
└──────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────┐
│  NIVEL 2: PROCESARE & TRANSFORMARE (Processing Layer)               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  System trigger (cron job):                                         │
│  ├─ Citire account.move + related lines                             │
│  ├─ Clasificare automat:                                             │
│  │  ├─ Per tip operație (Intraco, Import, etc.)                    │
│  │  ├─ Per cot TVA (19%, 9%, 5%, 0%)                               │
│  │  └─ Per categorie produs (dacă D300)                            │
│  │                                                                   │
│  ├─ Calcul agregat:                                                 │
│  │  ├─ Σ Basis = Total bază imponibilă                             │
│  │  ├─ Σ VAT = Total TVA                                           │
│  │  └─ Σ Amount = Total înregistrare                               │
│  │                                                                   │
│  └─ Aplicare reguli speciale:                                        │
│     ├─ Neexigibilitate (0% TVA)                                     │
│     ├─ Deductibilitate (partial/full)                               │
│     └─ Diferență curs (multi-currency)                              │
│                                                                       │
│  [Insert în tabele D100_*, D300_*, D394_*]                         │
└──────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────┐
│  NIVEL 3: STOCARE & VALIDARE (Storage Layer)                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Declarație record creată:                                          │
│  ├─ dakai_d394 (Master record)                                      │
│  │  ├─ facturi_ids: 150 înregistrări (Facturi emise)               │
│  │  ├─ facturi_primite_ids: 80 înregistrări (Facturi primite)      │
│  │  ├─ lista_ids: 230 înregistrări (Detaliu)                       │
│  │  ├─ operation_ids: 450 înregistrări (Clasificare)               │
│  │  └─ rezumat_ids: 15 înregistrări (Sumar)                        │
│  │                                                                   │
│  Validări:                                                          │
│  ├─ Reconciliare: Σ D394 = Σ Jurnal? → ✅ PASS                    │
│  ├─ TVA integrity: Σ Tax = Σ D394 VAT? → ✅ PASS                  │
│  ├─ Duplicate check: Nr. factură unic? → ✅ PASS                  │
│  └─ Completeness: Toate facturile incluse? → ⚠️ WARNING (2 lipsă) │
│                                                                       │
│  [Mark as "validated" or "draft"]                                   │
└──────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────┐
│  NIVEL 4: EXPORT & TRANSMISIE (Export Layer)                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  User: Click "Export to XML"                                        │
│  ├─ Serialization: D394 → XML SAGA format                           │
│  ├─ Compression: XML → ZIP (optional)                               │
│  ├─ Signing: ZIP → Signed ZIP (optional certificate)               │
│  │                                                                   │
│  └─ Download sau Email:                                             │
│     ├─ Option 1: Download direct (browser)                          │
│     ├─ Option 2: Email to ANAF (automated)                          │
│     └─ Option 3: FTP upload to gov server                           │
│                                                                       │
│  File output:                                                       │
│  └─ D394_2024_02.xml (sau .zip)                                    │
│     ├─ Size: 500KB - 5MB (depending on transactions)               │
│     ├─ Format: UTF-8 XML                                            │
│     └─ Signature: Optional SHA-256                                  │
│                                                                       │
│  [File sent to ANAF or saved locally]                              │
└──────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────┐
│  NIVEL 5: CONFIRMAȚIE & AUDIT TRAIL (Confirmation Layer)            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ANAF Response:                                                     │
│  ├─ ✅ Accepted: File stored in ANAF database                       │
│  ├─ ⚠️ Warning: Minor issues (non-blocking)                        │
│  └─ ❌ Rejected: Critical errors → Correct & re-submit             │
│                                                                       │
│  Odoo Record Update:                                                │
│  ├─ dakai_d394.state = 'submitted'                                  │
│  ├─ dakai_d394.submission_date = NOW                                │
│  ├─ dakai_d394.anaf_response = XML (saved in attachment)           │
│  └─ chatter message: "Submitted to ANAF on 2024-02-28"             │
│                                                                       │
│  Audit Trail:                                                       │
│  └─ log_ids: [                                                      │
│       {timestamp: '2024-02-15', action: 'created'},                │
│       {timestamp: '2024-02-20', action: 'validated'},              │
│       {timestamp: '2024-02-28', action: 'submitted_anaf'},         │
│     ]                                                               │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### 🔄 Ciclul Vieții unei Declarații

```
LIFECYCLE STATES:

1. DRAFT (Proiect)
   └─ Utilizator: Edit, Delete, Recalculate
   
2. READY_FOR_SUBMISSION (Gata pentru trimitere)
   └─ Sistem: Validări passed, Proofreading possible
   
3. VALIDATED (Validat)
   └─ Utilizator: Lock changes, Proceed to export
   
4. SUBMITTED (Trimis)
   └─ Export → ANAF
   └─ State changes: NEVER back (audit trail)
   
5. CONFIRMED (Confirmat)
   └─ ANAF: Acknowledement received
   └─ Mark as final in Odoo
   
6. REJECTED (Respins)
   └─ ANAF: Errors found
   └─ Action: Correct & Re-submit
   
7. ARCHIVED (Arhivat)
   └─ End of fiscal year
   └─ Read-only
```

---

## 🎯 REZUMAT FINAL

### Ce Realizează Sistemul?

| Funcție | Module | Status |
|---------|--------|--------|
| 🛡️ Declarații TVA (D100) | dakai_d100 | ✅ Complet |
| 📋 Declarații Salariale (D112) | dakai_d112 | ✅ Complet |
| 🧮 Rapoarte Produse (D300) | dakai_d300 | ✅ Complet |
| 📊 Liste Facturi (D394) | dakai_d394 | ✅ SUPER Complet |
| 🏛️ Contul Corespondent | l10n_ro_corresponding_account | ✅ SUPER Complet |
| 💰 Revaluări Multivalute | l10n_ro_currency_revaluation | ✅ Complet |
| 🏢 Active Imobilizate | l10n_ro_account_asset | ✅ Complet |
| 📤 Export XML (ANAF) | dakai_declarations_to_xml | ✅ Complet |
| 🔗 Integrare REGES | dakai_reges | ✅ Complet |

### Stack Tehnologic Utilizat

```
Backend:       Python 3.8+ (ORM SQLAlchemy)
Frontend:      OWL Components (JavaScript)
Database:      PostgreSQL
Web Framework: Odoo 19 native
Reports:       XML + PDF rendering
Testing:       unittest + Odoo test framework
Security:      Role-based ACL per model
i18n:          Multi-language translations
```

### KPI-uri de Proiect

- **27 Module** implementate
- **212 Fișiere Python** (.py)
- **66 XML Views** (.xml)
- **26 Test Files** (Unit + Integration)
- **~15,000 linii cod** (estimare)
- **Compatibil**: Odoo 19.0 (Python 3.8-3.10)

---

## 📚 DOCUMENTARE DISPONIBILĂ

1. **README.md** (l10n_ro_corresponding_account)
   - Descriere Concept Contul Corespondent
   
2. **business_requirements.md**
   - Cerințe detaliate pe proces
   
3. **process_flow.md**
   - Fluxuri de business documentate

---

**Repo**: adrian-dks/declaratii  
**Autor**: adrian-dks, Flavia0320  
**Maintained by**: Dakai SOFT SRL  
**License**: OPL-1 (Odoo Public License)

---

*Analiză completă — 2024*
