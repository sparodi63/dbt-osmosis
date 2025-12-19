# dbt-osmosis - Issues Quick Reference

**Report Data**: 18 Dicembre 2024
**Issues Totali**: 31 aperti

---

## 🔴 CRITICI (Da Risolvere Immediatamente)

| # | Titolo | Priorità | Effort | Status |
|---|--------|----------|--------|--------|
| **#263** | Support dbt 1.10 | ⭐⭐⭐⭐⭐ | 20-40h | 🔴 Blocca aggiornamenti |
| **#283** | FileNotFoundError con packages | ⭐⭐⭐⭐ | 4-8h | 🔴 Blocca elementary |
| **#278** | Empty tags/meta aggiunti | ⭐⭐⭐⭐ | 2-4h | ✅ Fix già presente |
| **#286** | Progenitor + skip-merge-meta | ⭐⭐⭐ | 4-6h | 🔴 Feature rotta |

**Total Effort Critici**: ~40-60 ore

---

## 🟠 ALTA PRIORITÀ (1-2 Settimane)

| # | Titolo | Priorità | Effort | Complessità |
|---|--------|----------|--------|-------------|
| **#289** | Quoted columns con spazi | ⭐⭐⭐ | 6-10h | 🔧🔧 Media |
| **#287** | Preserve custom descriptions | ⭐⭐⭐ | 8-12h | 🔧🔧🔧 Media-Alta |
| **#272** | Jinja in source names | ⭐⭐⭐ | 6-10h | 🔧🔧 Media |

**Total Effort Alta**: ~20-32 ore

---

## 🟡 MEDIA PRIORITÀ (1 Mese)

| # | Titolo | Priorità | Effort |
|---|--------|----------|--------|
| **#276** | Trailing whitespace | ⭐⭐ | 2-4h |
| **#274** | Vars initialization timing | ⭐⭐ | 2-3h |
| **#279** | Dynamic YAML filenames | ⭐⭐ | 8-12h |
| **#277** | Custom LLM prompts | ⭐⭐ | 6-8h |
| **#266** | Policy tags rendering | ⭐⭐ | 1-2h (docs) |

**Total Effort Media**: ~20-30 ore

---

## 🔵 BASSA PRIORITÀ (Backlog)

| # | Titolo | Tipo | Effort |
|---|--------|------|--------|
| **#273** | Adopt dbt-core-interface | Refactoring | 40-80h |
| **#231** | include_external option | Enhancement | 4-6h |
| **#219** | Flag combination issues | Bug | 4-8h |
| **#217** | New tables not added | Bug | 6-10h |
| **#216** | Macros inheritance | Enhancement | 6-10h |
| **#172** | Progenitor accuracy | Enhancement | 10-15h |
| **#167** | DBT Cloud support | Feature | 40+h |

---

## Piano Sprint Raccomandato

### Sprint 1 (Week 1-2): CRITICAL FIXES
```
✓ #278 - Empty tags/meta (già fixato, serve release)
✓ #283 - FileNotFoundError packages
✓ #286 - Progenitor compatibility
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Deliverable: Release 1.1.18
Effort: 10-18h
```

### Sprint 2 (Week 3-4): DBT 1.10 + INHERITANCE
```
✓ #263 - dbt 1.10 support (PRIORITÀ MASSIMA)
✓ #289 - Quoted columns
✓ #287 - Custom descriptions
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Deliverable: Release 1.2.0 o 2.0.0
Effort: 34-62h
```

### Sprint 3 (Week 5-6): QUALITY OF LIFE
```
✓ #272 - Jinja sources
✓ #274 - Vars timing
✓ #276 - Whitespace
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Deliverable: Release 1.2.1
Effort: 10-17h
```

### Sprint 4+ (Future): ENHANCEMENTS
```
✓ #279 - Dynamic filenames
✓ #277 - Custom prompts
✓ #273 - dbt-core-interface refactor
✓ Altri enhancement issues
```

---

## Issue Details - Quick Summary

### #263: dbt 1.10 Support
**Problema**: dbt 1.10 ha cambiato schema `meta`, dbt-osmosis non compatibile
**Impatto**: Blocca aggiornamenti di sicurezza e nuove features dbt
**Soluzione**: Implementare backward compatibility con adapter pattern
**File**: `config.py`, `inheritance.py`, `introspection.py`

### #283: FileNotFoundError Packages
**Problema**: Cerca file packages in directory sbagliata
**Soluzione**: Fix path resolution in `path_management.py:_is_file_match()`
```python
if node.package_name != node.project_name:
    package_path = Path(root, 'dbt_packages', node.package_name, ...)
```

### #278: Empty Tags/Meta
**Problema**: Aggiunge `tags: []` e `meta: {}` anche con skip flags
**Status**: ✅ **FIX GIÀ PRESENTE** nel fork (commit 3476a5a)
**Azione**: Testing e release

### #286: Progenitor Incompatibilità
**Problema**: `--add-progenitor-to-meta` non funziona con `--skip-merge-meta`
**Soluzione**: Special case per progenitor in `transforms.py:217`

### #289: Quoted Columns
**Problema**: Non eredita colonne con nomi quotati contenenti spazi
**Soluzione**: Migliorare `normalize_column_name()` e aggiungere plugin

### #287: Custom Descriptions
**Problema**: `--force-inherit` sovrascrive descrizioni custom
**Soluzione**: Aggiungere flag `meta.osmosis_keep_description: true`

### #272: Jinja Sources
**Problema**: Crea duplicati con Jinja template in source names
**Soluzione**: Resolve Jinja prima del comparison in `sync_operations.py:148`

---

## File Chiave da Modificare

### Per Issue Critici:
```
src/dbt_osmosis/core/
├── config.py           # #263 dbt 1.10
├── inheritance.py      # #263, #278, #286
├── introspection.py    # #263, #289
├── path_management.py  # #283
├── transforms.py       # #278, #286, #287
└── sync_operations.py  # #272
```

### Per Testing:
```
tests/
├── test_dbt_110_compatibility.py  # NEW
├── test_packages.py               # NEW
├── test_quoted_identifiers.py     # NEW
├── test_progenitor.py             # NEW
└── test_jinja_sources.py          # NEW
```

---

## Comandi Test Rapidi

### Test Issue #263 (dbt 1.10)
```bash
pip install "dbt-core==1.10.0"
dbt-osmosis yaml refactor --dry-run
```

### Test Issue #283 (Packages)
```bash
# Aggiungi elementary al packages.yml
dbt deps
dbt-osmosis yaml refactor
# Should not: FileNotFoundError
```

### Test Issue #278 (Empty tags/meta)
```bash
dbt-osmosis yaml document --skip-add-tags --skip-merge-meta
# Verify no empty tags: [] or meta: {}
```

### Test Issue #286 (Progenitor)
```bash
dbt-osmosis yaml refactor \
  --add-progenitor-to-meta \
  --skip-merge-meta
# Verify meta.osmosis_progenitor is added
```

### Test Issue #289 (Quoted columns)
```sql
-- models/test.sql
select col as "Column Name" from upstream
```
```bash
dbt-osmosis yaml refactor
# Should inherit documentation
```

---

## Resource Allocation

### Minimal Team (2 settimane)
- **1 Senior Dev** (full-time): #263, #286, #287
- **1 Mid-level Dev** (full-time): #283, #289, #272
- **Part-time**: Testing, docs

### Timeline
- **Week 1**: Sprint 1 (critical fixes)
- **Week 2**: Sprint 1 completion + release
- **Week 3-4**: Sprint 2 (dbt 1.10)
- **Week 5-6**: Sprint 3 (QoL improvements)

---

## Success Metrics

### Week 2 Goals
- ✅ 3 issue critici chiusi
- ✅ Release 1.1.18 pubblicata
- ✅ Test coverage >60%

### Month 1 Goals
- ✅ dbt 1.10 support rilasciato
- ✅ 6+ issue totali chiusi
- ✅ CI/CD pipeline attivo

### Month 3 Goals
- ✅ 12+ issue totali chiusi
- ✅ Test coverage >80%
- ✅ Documentation completa

---

## Risk Assessment

### 🔴 High Risk
- **Maintainer availability**: Progetto sembra in pausa
- **dbt-core API changes**: Versioni future possono rompere cose
- **#263 complexity**: Refactoring significativo richiesto

### 🟡 Medium Risk
- **Breaking changes**: Alcuni fix possono richiedere breaking changes
- **Testing gaps**: Coverage attuale sconosciuto
- **Community expectations**: Molti utenti dipendono dal tool

### 🟢 Low Risk
- **Clear solutions**: Molti issue hanno fix chiari
- **Active community**: 31 issue = community engaged
- **Fork advantage**: Alcuni fix già presenti

---

## Quick Actions Checklist

### Questa Settimana
- [ ] Review issue #278 fix nel fork
- [ ] Prepare release 1.1.18
- [ ] Setup test environment per #283
- [ ] Iniziare design per #263

### Prossime 2 Settimane
- [ ] Release 1.1.18 con fix critici
- [ ] Fix #283 e #286
- [ ] Planning dettagliato per dbt 1.10
- [ ] Setup CI/CD pipeline

### Questo Mese
- [ ] Implementare dbt 1.10 support
- [ ] Fix issue inheritance (#289, #287)
- [ ] Release 1.2.0
- [ ] Aggiornare documentazione

---

## Contatti e Risorse

**Repository**: https://github.com/z3z1ma/dbt-osmosis
**Report Completo**: `ISSUES_ANALYSIS_REPORT.md`
**Maintenance Guide**: `DBT_OSMOSIS_MAINTENANCE_GUIDE.md`

**Per Domande**:
- Aprire GitHub Discussion
- Tag maintainer (@z3z1ma)
- Community dbt Slack #dbt-osmosis

---

**Last Updated**: 2024-12-18
**Next Review**: 2025-01-15 (monthly)
