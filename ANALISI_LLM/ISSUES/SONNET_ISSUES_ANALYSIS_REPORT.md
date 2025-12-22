# dbt-osmosis - Issues Analysis Report

**Data Analisi**: 18 Dicembre 2024
**Repository**: https://github.com/z3z1ma/dbt-osmosis
**Issues Aperti**: 31
**Versione Corrente**: 1.1.17

---

## Executive Summary

Il progetto dbt-osmosis presenta **31 issue aperti** che richiedono attenzione. L'analisi ha identificato:

- **5 bug critici** che bloccano funzionalità esistenti
- **8 bug di media priorità** che impattano l'esperienza utente
- **12 enhancement/feature request** per nuove funzionalità
- **6 miglioramenti tecnici** per architettura e manutenibilità

### Problematiche Principali

1. **Compatibilità dbt-core 1.10**: Issue più critico, blocca l'adozione di nuove versioni
2. **Bug nell'inheritance system**: Problemi con colonne quotate, descrizioni custom, meta fields
3. **Gestione dei packages**: FileNotFoundError con dbt packages esterni
4. **Formattazione YAML**: Whitespace e formatting issues
5. **Jinja rendering**: Problemi con variabili Jinja in vari contesti

---

## Priorità di Risoluzione

### 🔴 PRIORITÀ CRITICA - Risolvere Immediatamente

#### Issue #263: Support dbt 1.10
**Impatto**: ⭐⭐⭐⭐⭐ (Blocca aggiornamenti di sicurezza e nuove funzionalità dbt)
**Complessità**: 🔧🔧🔧🔧 (Alta - richiede refactoring significativo)
**Utenti Impattati**: Tutti gli utenti che vogliono usare dbt 1.10+

**Problema**:
- dbt 1.10 ha introdotto cambiamenti allo schema `meta`
- dbt-osmosis non supporta dbt 1.10+
- Gli utenti non possono aggiornare dbt per patch di sicurezza e nuove features (come la sintassi `arguments`)

**Root Cause**:
- File `osmosis.py` (~3000 righe) difficile da refactorare
- Necessità di gestire sia dbt 1.9 che 1.10 (o decidere di droppare 1.9)

**Suggerimenti per la Risoluzione**:

**Opzione A - Backward Compatibility (Raccomandata)**:
```python
# In config.py o file separato
def get_meta_schema_version():
    """Detect dbt-core version and return appropriate schema handler."""
    import dbt.version
    if dbt.version.__version__.startswith("1.10"):
        return MetaSchemaV10()
    else:
        return MetaSchemaV9()

class MetaSchemaV9:
    def parse_meta(self, meta_dict):
        # dbt 1.9 logic
        pass

class MetaSchemaV10:
    def parse_meta(self, meta_dict):
        # dbt 1.10 logic with new schema
        pass
```

**Opzione B - Forward-Only (Più Semplice)**:
```toml
# pyproject.toml
dependencies = [
  "dbt-core>=1.10,<1.12",  # Drop support for 1.9
  # ...
]
```

**Piano di Implementazione**:
1. Creare branch `feat/dbt-1.10-support`
2. Aggiungere test per dbt 1.10.x
3. Identificare tutti i luoghi dove si accede a `meta`
4. Implementare adapter pattern per gestire diverse versioni
5. Testare su progetti reali con dbt 1.9 e 1.10
6. Release come 1.2.0 o 2.0.0 (se breaking)

**Effort Stimato**: 20-40 ore
**Risorse Necessarie**: 1 developer senior con esperienza dbt-core

**File da Modificare**:
- `src/dbt_osmosis/core/config.py`
- `src/dbt_osmosis/core/inheritance.py`
- `src/dbt_osmosis/core/introspection.py`
- `pyproject.toml` (version constraints)
- `tests/` (add dbt 1.10 test cases)

---

#### Issue #283: FileNotFoundError with dbt Packages
**Impatto**: ⭐⭐⭐⭐ (Blocca uso con packages popolari come elementary)
**Complessità**: 🔧🔧 (Media - path resolution)
**Utenti Impattati**: Chiunque usi dbt packages (elementary, dbt_utils, etc.)

**Problema**:
dbt-osmosis cerca i file dei packages nella directory source invece che in `target/compiled` o `dbt_packages/`

**Errore**:
```
FileNotFoundError: ../src/models/edr/run_results/snapshot_run_results.sql
```

**Root Cause**:
Nel file `path_management.py`, il metodo `_is_file_match` non gestisce correttamente i package paths.

**Suggerimenti per la Risoluzione**:

```python
# In src/dbt_osmosis/core/path_management.py

def _is_file_match(node: ResultNode, root: Path) -> bool:
    """Check if file exists, handling both project and package files."""

    # Check if this is a package (not the main project)
    if node.package_name != node.project_name:
        # Try dbt_packages directory first (modern dbt)
        package_path = Path(root, 'dbt_packages', node.package_name, node.original_file_path)
        if package_path.exists():
            return True

        # Try modules directory (legacy dbt)
        modules_path = Path(root, 'modules', node.package_name, node.original_file_path)
        if modules_path.exists():
            return True

        # Try target/compiled for compiled packages
        compiled_path = Path(root, 'target', 'compiled', node.package_name, node.original_file_path)
        if compiled_path.exists():
            return True

        return False

    # Original logic for project files
    return Path(root, node.original_file_path).exists()
```

**Testing**:
```bash
# Test with elementary package
dbt deps
dbt compile
dbt-osmosis yaml refactor --dry-run

# Verify no FileNotFoundError for package files
```

**Effort Stimato**: 4-8 ore
**Risorse Necessarie**: 1 developer

**File da Modificare**:
- `src/dbt_osmosis/core/path_management.py` (~line 148 in sync_operations.py reference)
- `tests/test_path_management.py` (add package tests)

---

#### Issue #278: Empty Tags and Meta Keys Added
**Impatto**: ⭐⭐⭐⭐ (Inquina YAML files, causa problemi con version control)
**Complessità**: 🔧🔧 (Media)
**Utenti Impattati**: Tutti gli utenti con dbt-core >= 1.9.6

**Problema**:
Con `--skip-add-tags` e `--skip-merge-meta`, dbt-osmosis aggiunge comunque:
```yaml
config:
  tags: []
  meta: {}
```

**Root Cause**:
Cambiamento in dbt-core >= 1.9.6 (PR #11671). Il problema è già stato identificato nel fork.

**Suggerimenti per la Risoluzione**:

**Soluzione già implementata nel fork** (da verificare e mergere):
```python
# In src/dbt_osmosis/core/inheritance.py:207-212

# Remove empty tags and meta regardless of flags
if graph_edge.get("tags") == []:
    del graph_edge["tags"]
if graph_edge.get("meta") == {}:
    del graph_edge["meta"]
```

**E in transforms.py:232-235**:
```python
# Remove empty tags and meta to avoid adding them to the node
if updated_metadata.get("tags") == []:
    updated_metadata.pop("tags", None)
if updated_metadata.get("meta") == {}:
    updated_metadata.pop("meta", None)
```

**Piano di Implementazione**:
1. ✅ Fix già presente nel fork (commit `3476a5a`)
2. Verificare che funzioni con dbt-core 1.9.6+
3. Aggiungere regression test
4. Rilasciare come patch release

**Testing**:
```bash
# Test con dbt-core 1.9.6+
dbt-osmosis yaml document --skip-add-tags --skip-merge-meta

# Verificare che non vengano aggiunti tags: [] o meta: {}
```

**Effort Stimato**: 2-4 ore (principalmente testing)
**Risorse Necessarie**: 1 developer

**Status**: ✅ **GIÀ RISOLTO NEL FORK** - Needs release

---

#### Issue #286: Progenitor Metadata Flag Incompatible with skip-merge-meta
**Impatto**: ⭐⭐⭐ (Feature importante non funziona)
**Complessità**: 🔧🔧 (Media)
**Utenti Impattati**: Utenti che vogliono tracciare l'origine della documentazione

**Problema**:
Quando si usano insieme `--add-progenitor-to-meta` e `--skip-merge-meta`, il progenitor non viene aggiunto.

**Root Cause**:
Conflitto logico:
1. `inheritance.py:139-147` aggiunge `osmosis_progenitor` al campo `meta`
2. `transforms.py:217` esclude `meta` dagli inheritable fields se `skip-merge-meta` è true

**Suggerimenti per la Risoluzione**:

```python
# In src/dbt_osmosis/core/transforms.py

def inherit_upstream_column_knowledge(context, node):
    """Inherit column knowledge with special handling for progenitor."""

    column_knowledge_graph = _build_column_knowledge_graph(context, node)

    for name, node_column in node.columns.items():
        kwargs = column_knowledge_graph.get(name)
        if kwargs is None:
            continue

        inheritable = ["description"]

        # Handle tags
        if not _get_setting_for_node("skip-add-tags", node, name, ...):
            inheritable.append("tags")

        # Handle meta - special case for progenitor
        skip_merge_meta = _get_setting_for_node("skip-merge-meta", node, name, ...)
        add_progenitor = _get_setting_for_node("add-progenitor-to-meta", node, name, ...)

        if not skip_merge_meta:
            # Normal case: inherit all meta
            inheritable.append("meta")
        elif add_progenitor:
            # Special case: skip meta merge BUT still add progenitor
            if "meta" in kwargs and "osmosis_progenitor" in kwargs["meta"]:
                # Only inherit the progenitor field, not other meta
                node_column.meta = node_column.meta or {}
                node_column.meta["osmosis_progenitor"] = kwargs["meta"]["osmosis_progenitor"]
                kwargs.pop("meta")  # Remove from general inheritance

        # Continue with normal inheritance...
```

**Testing**:
```bash
# Test combination of flags
dbt-osmosis yaml refactor \
  --add-progenitor-to-meta \
  --skip-merge-meta

# Verify meta.osmosis_progenitor is added
```

**Effort Stimato**: 4-6 ore
**Risorse Necessarie**: 1 developer

**File da Modificare**:
- `src/dbt_osmosis/core/transforms.py:217`
- `src/dbt_osmosis/core/inheritance.py:139-147`
- `tests/test_progenitor.py` (new file)

---

### 🟠 PRIORITÀ ALTA - Risolvere entro 1-2 settimane

#### Issue #289: Not Inheriting Quoted Columns with Spaces
**Impatto**: ⭐⭐⭐ (Colpisce utenti con convenzioni naming specifiche)
**Complessità**: 🔧🔧 (Media)
**Utenti Impattati**: Utenti con quoted identifiers che contengono spazi

**Problema**:
```sql
select
    upstream_col_1 as "new_col_1",  -- ✅ Funziona
    upstream_col_2 as "New Col 2"   -- ❌ Non eredita
```

**Root Cause**:
Il sistema di column matching non gestisce correttamente quoted identifiers con spazi.

**Suggerimenti per la Risoluzione**:

```python
# In src/dbt_osmosis/core/introspection.py

def normalize_column_name(column_name: str, adapter_type: str) -> str:
    """Normalize column names, handling quoted identifiers."""

    # Check if this is a quoted identifier
    if column_name.startswith('"') and column_name.endswith('"'):
        # Remove quotes and preserve case and spaces
        return column_name[1:-1]

    # Original normalization logic
    if adapter_type in ("snowflake", "bigquery", "redshift"):
        return column_name.lower()

    return column_name

# Add to plugin system
class QuotedIdentifierMatching:
    """Plugin for matching quoted identifiers with spaces."""

    def match(self, column_a: str, column_b: str) -> bool:
        """Match columns considering quotes and spaces."""

        # Strip quotes for comparison
        a = column_a.strip('"').strip()
        b = column_b.strip('"').strip()

        # Case-insensitive match
        return a.lower() == b.lower()
```

**Testing**:
```sql
-- Test model: models/test_quoted.sql
select
    id as "Customer ID",
    name as "Full Name",
    email as "Email Address"
from {{ ref('source_model') }}
```

```bash
dbt-osmosis yaml refactor --dry-run
# Verify all columns inherit documentation
```

**Effort Stimato**: 6-10 ore
**Risorse Necessarie**: 1 developer

**File da Modificare**:
- `src/dbt_osmosis/core/introspection.py`
- `src/dbt_osmosis/core/plugins.py`
- `src/dbt_osmosis/core/inheritance.py`
- `tests/test_quoted_identifiers.py` (new)

---

#### Issue #287: Preserve Custom Descriptions During Inheritance
**Impatto**: ⭐⭐⭐ (Caso d'uso comune: documentazione specifica per mart)
**Complessità**: 🔧🔧🔧 (Media-Alta)
**Utenti Impattati**: Team che customizzano documentazione nei mart

**Problema**:
`--force-inherit-descriptions` sovrascrive TUTTE le descrizioni downstream, anche quelle custom.

**Scenario**:
```yaml
# staging/stg_shops.yml
- name: shop_id
  description: "ID of shop"

# marts/dim_shops.yml
- name: shop_id
  description: "ID of the latest shop where user made purchase"  # Viene sovrascritto!
```

**Soluzione Desiderata**:
Ripristinare la funzionalità `osmosis_keep_description` deprecata nella v1.1.6.

**Suggerimenti per la Risoluzione**:

**Opzione A - Meta Flag (Raccomandato)**:
```yaml
# marts/dim_shops.yml
- name: shop_id
  description: "ID of the latest shop where user made purchase"
  meta:
    osmosis_keep_description: true  # Flag per preservare
```

**Opzione B - Nuovo Flag CLI**:
```bash
dbt-osmosis yaml refactor \
  --force-inherit-descriptions \
  --preserve-custom-descriptions
```

**Implementazione Opzione A**:
```python
# In src/dbt_osmosis/core/inheritance.py

def _build_column_knowledge_graph(context, node):
    """Build knowledge graph with custom description preservation."""

    # ... existing code ...

    for generation in reversed(sorted(tree.keys())):
        for ancestor_uid in ancestors:
            # ... get ancestor columns ...

            for name, _ in node.columns.items():
                # Check if column has osmosis_keep_description flag
                current_col = node.columns[name]
                keep_desc = (
                    current_col.meta
                    and current_col.meta.get("osmosis_keep_description") == True
                )

                if keep_desc and current_col.description:
                    # Skip inheriting description for this column
                    graph_edge.pop("description", None)

                # ... continue with inheritance ...
```

**Testing**:
```yaml
# Test case
columns:
  - name: shop_id
    description: "Custom description"
    meta:
      osmosis_keep_description: true
```

```bash
dbt-osmosis yaml refactor --force-inherit-descriptions
# Verify custom description is preserved
```

**Effort Stimato**: 8-12 ore
**Risorse Necessarie**: 1 developer

**File da Modificare**:
- `src/dbt_osmosis/core/inheritance.py`
- `src/dbt_osmosis/core/settings.py` (if adding CLI flag)
- Documentation
- `tests/test_preserve_descriptions.py` (new)

---

#### Issue #272: Dynamic Source Names with Jinja Create Duplicates
**Impatto**: ⭐⭐⭐ (Confusione nei YAML files)
**Complessità**: 🔧🔧 (Media)
**Utenti Impattati**: Utenti che usano Jinja in source names

**Problema**:
```yaml
# Source definition
- name: "{{ var('some_source')[target.name] }}"

# dbt-osmosis crea:
- name: "{{ var('some_source')[target.name] }}"
- name: "some_source_dev"  # Duplicato!
```

**Root Cause**:
In `sync_operations.py:148`:
```python
if s.get("name") == node.source_name:  # String comparison fails
```

**Suggerimenti per la Risoluzione**:

```python
# In src/dbt_osmosis/core/sync_operations.py

def _resolve_jinja_if_needed(template_str: str, context) -> str:
    """Resolve Jinja template to actual value."""
    if "{{" in template_str and "}}" in template_str:
        try:
            from jinja2 import Template
            # Get vars from context
            vars_dict = context.project.runtime_cfg.vars.to_dict()
            target = context.project.runtime_cfg.target_name

            template = Template(template_str)
            return template.render(var=lambda x: vars_dict.get(x, {}), target={'name': target})
        except Exception:
            return template_str
    return template_str

def sync_source_to_yaml(context, node):
    """Sync source with Jinja resolution."""

    # ... existing code ...

    for s in sources:
        source_name = s.get("name")

        # Resolve Jinja template
        resolved_name = _resolve_jinja_if_needed(source_name, context)

        # Compare resolved names
        if resolved_name == node.source_name:
            # Match found, update existing source
            # ...
```

**Alternative - Add Warning**:
```python
def check_source_name_template(source_name: str) -> bool:
    """Check if source name contains Jinja."""
    if "{{" in source_name and "}}" in source_name:
        logger.warning(
            f"Source name contains Jinja template: {source_name}. "
            f"Consider using static names to avoid duplication issues."
        )
        return True
    return False
```

**Testing**:
```yaml
# Test configuration
vars:
  some_source:
    dev: "source_dev"
    prod: "source_prod"

sources:
  - name: "{{ var('some_source')[target.name] }}"
```

```bash
dbt-osmosis yaml refactor
# Verify no duplicates created
```

**Effort Stimato**: 6-10 ore
**Risorse Necessarie**: 1 developer

**File da Modificare**:
- `src/dbt_osmosis/core/sync_operations.py:148`
- `tests/test_jinja_sources.py` (new)

---

### 🟡 PRIORITÀ MEDIA - Risolvere entro 1 mese

#### Issue #276: Unwanted Trailing Whitespace
**Impatto**: ⭐⭐ (Fastidioso ma non bloccante)
**Complessità**: 🔧 (Bassa)
**Utenti Impattati**: Utenti con linting strict su whitespace

**Problema**:
dbt-osmosis aggiunge trailing whitespace nelle descrizioni multi-linea.

**Suggerimenti per la Risoluzione**:

```python
# In src/dbt_osmosis/core/schema/writer.py

def format_description(description: str) -> str:
    """Format description removing trailing whitespace."""

    # Split into lines
    lines = description.split('\n')

    # Strip trailing whitespace from each line
    lines = [line.rstrip() for line in lines]

    # Rejoin
    return '\n'.join(lines)

def write_column(column_data: dict) -> dict:
    """Write column with formatted description."""

    if 'description' in column_data:
        column_data['description'] = format_description(column_data['description'])

    return column_data
```

**Testing**:
```python
def test_no_trailing_whitespace():
    """Test that descriptions have no trailing whitespace."""
    description = "Line 1  \nLine 2  \nLine 3  "
    formatted = format_description(description)

    for line in formatted.split('\n'):
        assert not line.endswith(' '), f"Line has trailing space: {line!r}"
```

**Effort Stimato**: 2-4 ore
**Risorse Necessarie**: 1 developer

**File da Modificare**:
- `src/dbt_osmosis/core/schema/writer.py`
- `tests/test_formatting.py`

---

#### Issue #274: Variable Initialization Timing in DbtConfiguration
**Impatto**: ⭐⭐ (Causa errori di compilazione)
**Complessità**: 🔧 (Bassa)
**Utenti Impattati**: Utenti che usano vars in pre-hooks o test configs

**Problema**:
`DbtConfiguration` viene creato senza `vars`, causando:
```
Required var 'yyyymmdd' not found in config
```

**Soluzione già proposta dall'utente**:
```python
# In cli/main.py

import yaml as yaml_handler

settings = DbtConfiguration(
    project_dir=t.cast(str, project_dir),
    profiles_dir=t.cast(str, profiles_dir),
    target=target,
    profile=profile,
    threads=threads,
    vars=yaml_handler.safe_load(vars) if vars else None,  # Add vars here
    disable_introspection=disable_introspection,
)
```

**Testing**:
```yaml
# dbt_project.yml
vars:
  yyyymmdd: "20241218"

# Test with vars in pre-hook
on-run-start:
  - "{{ log('Date: ' ~ var('yyyymmdd')) }}"
```

```bash
dbt-osmosis yaml refactor --vars '{"yyyymmdd": "20241218"}'
# Should not error
```

**Effort Stimato**: 2-3 ore
**Risorse Necessarie**: 1 developer

**File da Modificare**:
- `src/dbt_osmosis/cli/main.py` (multiple commands)
- `tests/test_vars_timing.py` (new)

---

#### Issue #279: Dynamic YAML Filename Generation
**Impatto**: ⭐⭐ (Feature request, non critica)
**Complessità**: 🔧🔧🔧 (Media-Alta)
**Utenti Impattati**: Utenti con deep nested directories

**Problema**:
Richiesta per generare nomi file come `_staging__jaffle_shop__models.yml` basati su directory nesting.

**Suggerimenti per la Risoluzione**:

```python
# In src/dbt_osmosis/core/path_management.py

def _get_yaml_path_template(node: ResultNode, config: dict) -> str:
    """Get YAML path template with FQN support."""

    template = config.get("dbt-osmosis", "_{model}.yml")

    # Support new FQN-based placeholders
    template = template.replace("{fqn_joined}", "__".join(node.fqn))
    template = template.replace("{fqn[-1]}", node.fqn[-1] if node.fqn else "")
    template = template.replace("{fqn[-2]}", node.fqn[-2] if len(node.fqn) > 1 else "")

    # ... existing replacements ...

    return template
```

**Configuration Example**:
```yaml
models:
  my_project:
    staging:
      +dbt-osmosis: "_{fqn_joined}_models.yml"
      # staging/jaffle_shop/customers.sql
      # → _staging__jaffle_shop_models.yml
```

**Effort Stimato**: 8-12 ore
**Risorse Necessarie**: 1 developer

**File da Modificare**:
- `src/dbt_osmosis/core/path_management.py`
- Documentation
- `tests/test_fqn_templates.py` (new)

---

#### Issue #277: Custom Prompt for LLM Documentation
**Impatto**: ⭐⭐ (Useful feature, non urgente)
**Complessità**: 🔧🔧 (Media)
**Utenti Impattati**: Utenti che usano --synthesize

**Problema**:
Richiesta di customizzare il system prompt per LLM documentation generation.

**Suggerimenti per la Risoluzione**:

```python
# Configuration file: .dbt-osmosis.yml
llm:
  provider: openai
  model: gpt-4o
  custom_prompt: |
    You are a data documentation expert.
    Follow these rules:
    - Use business terminology, not technical jargon
    - Be concise (max 2 sentences per column)
    - Include data type in description
    - Mention PII/sensitive data if applicable

# In src/dbt_osmosis/core/llm.py

def get_system_prompt(context) -> str:
    """Get system prompt from config or use default."""

    # Try to load from .dbt-osmosis.yml
    config_file = Path(context.project.project_root) / ".dbt-osmosis.yml"

    if config_file.exists():
        with open(config_file) as f:
            config = yaml.safe_load(f)
            custom_prompt = config.get("llm", {}).get("custom_prompt")
            if custom_prompt:
                return custom_prompt

    # Default prompt
    return DEFAULT_SYSTEM_PROMPT

def generate_column_doc(column_name, **kwargs):
    """Generate documentation with custom prompt."""

    system_prompt = get_system_prompt(kwargs.get("context"))

    # ... rest of implementation ...
```

**Effort Stimato**: 6-8 ore
**Risorse Necessarie**: 1 developer

**File da Modificare**:
- `src/dbt_osmosis/core/llm.py`
- `src/dbt_osmosis/core/settings.py`
- Documentation
- `tests/test_custom_prompts.py` (new)

---

#### Issue #266: Prevent Jinja Variable Rendering in Policy Tags
**Impatto**: ⭐⭐ (Già ha workaround)
**Complessità**: 🔧 (Bassa - già risolto)
**Utenti Impattati**: Utenti BigQuery con policy tags

**Problema**:
Policy tags con Jinja vengono renderizzati invece di rimanere template.

**Soluzione Esistente**:
```bash
dbt-osmosis yaml refactor --add-inheritance-for-specified-keys=policy_tags
```

**Azioni Necessarie**:
1. ✅ Soluzione già disponibile via flag
2. Migliorare documentazione
3. Aggiungere esempio nel README

**Effort Stimato**: 1-2 ore (solo docs)
**Risorse Necessarie**: Technical writer o developer

---

### 🔵 PRIORITÀ BASSA - Backlog / Future Enhancements

#### Issue #273: Adopt dbt-core-interface Library
**Impatto**: ⭐⭐ (Refactoring architetturale)
**Complessità**: 🔧🔧🔧🔧 (Alta)
**Utenti Impattati**: Nessuno (internal improvement)

**Descrizione**:
Proposta del maintainer di adottare `dbt-core-interface` per interfacciarsi con dbt-core.

**Benefici**:
- Astrazione dal dbt-core interno
- Più facile supportare multiple versioni dbt
- Codice più manutenibile

**Suggerimenti**:
- Valutare dopo aver risolto issue critici
- Pianificare come parte di major version (2.0)
- Verificare stabilità di dbt-core-interface

**Effort Stimato**: 40-80 ore
**Priorità**: Da valutare dopo issue critici

---

#### Issue #231: CLI Support for include_external Option
**Impatto**: ⭐ (Niche use case)
**Complessità**: 🔧🔧 (Media)

**Descrizione**:
Richiesta di supportare l'opzione `include_external` per abilitare dbt packages.

**Suggerimenti**:
```bash
dbt-osmosis yaml refactor --include-external
```

**Effort Stimato**: 4-6 ore

---

#### Issue #219: Combination of Flags Not Working
**Impatto**: ⭐⭐ (Edge case)
**Complessità**: 🔧🔧 (Media)

**Descrizione**:
`--force-inherit-descriptions` + `--use-unrendered-descriptions` non funzionano come atteso.

**Azioni**:
- Investigare interazione tra flags
- Documentare comportamento atteso
- Fix o documentare limitation

**Effort Stimato**: 4-8 ore

---

#### Issue #217: New Tables Not Added to Existing Source YAML
**Impatto**: ⭐⭐ (Workaround manuale disponibile)
**Complessità**: 🔧🔧 (Media)

**Descrizione**:
Quando si aggiungono nuove tabelle a una source esistente, dbt-osmosis non le aggiunge automaticamente.

**Effort Stimato**: 6-10 ore

---

#### Issue #216: Ensure Macros Are Inherited Not Rendered
**Impatto**: ⭐⭐ (Simile a #266)
**Complessità**: 🔧🔧 (Media)

**Descrizione**:
Doc macros (`{{ doc('...') }}`) vengono renderizzate invece che ereditate.

**Effort Stimato**: 6-10 ore

---

#### Issue #172: Determine osmosis_progenitor Accurately
**Impatto**: ⭐ (Enhancement)
**Complessità**: 🔧🔧🔧 (Media-Alta)

**Descrizione**:
Migliorare logica di determinazione del progenitor quando ci sono multiple sources alla stessa distanza.

**Effort Stimato**: 10-15 ore

---

#### Issue #167: DBT Cloud Support
**Impatto**: ⭐ (Feature request)
**Complessità**: 🔧🔧🔧🔧 (Alta)

**Descrizione**:
Supportare dbt Cloud come alternativa a dbt CLI.

**Effort Stimato**: 40+ ore

---

## Piano di Risoluzione Raccomandato

### Sprint 1 (Settimana 1-2): Issues Critici
**Obiettivo**: Sbloccare funzionalità fondamentali

1. **Issue #278** - Empty tags/meta (2-4h)
   - ✅ Fix già presente, needs testing e release
   - Priorità: Test regression + release

2. **Issue #283** - FileNotFoundError packages (4-8h)
   - Fix path resolution per dbt packages
   - Testing con elementary e altri packages

3. **Issue #286** - Progenitor + skip-merge-meta (4-6h)
   - Fix conflitto logico tra flags
   - Testing combinazioni flags

### Sprint 2 (Settimana 3-4): Compatibility e Inheritance
**Obiettivo**: dbt 1.10 support e migliorare inheritance

4. **Issue #263** - dbt 1.10 support (20-40h)
   - Implementare backward compatibility
   - Testing estensivo
   - Release come 1.2.0 o 2.0.0

5. **Issue #289** - Quoted columns (6-10h)
   - Fix column matching per quoted identifiers
   - Plugin per spazi in nomi colonne

6. **Issue #287** - Preserve custom descriptions (8-12h)
   - Implementare osmosis_keep_description
   - Documentation e examples

### Sprint 3 (Settimana 5-6): Quality of Life
**Obiettivo**: Migliorare UX e risolvere bugs minori

7. **Issue #272** - Jinja source names (6-10h)
   - Resolve Jinja templates
   - Prevent duplicates

8. **Issue #274** - Vars timing (2-3h)
   - Fix initialization order
   - Simple fix

9. **Issue #276** - Trailing whitespace (2-4h)
   - Format descriptions
   - Linting fix

### Sprint 4+ (Future): Enhancements
**Obiettivo**: Nuove features e refactoring

10. **Issue #279** - Dynamic filenames (8-12h)
11. **Issue #277** - Custom LLM prompts (6-8h)
12. **Issue #273** - dbt-core-interface (40-80h)
13. Altri enhancement issues

---

## Metriche di Successo

### KPI per Issues Resolution

**Immediate (2 settimane)**:
- ✅ 3 issue critici risolti (#278, #283, #286)
- ✅ Release 1.1.18 con bug fixes
- ✅ dbt 1.10 support plan definito

**Short-term (1 mese)**:
- ✅ dbt 1.10 support rilasciato
- ✅ 6 issue totali chiusi
- ✅ Test coverage aumentato a >70%

**Medium-term (3 mesi)**:
- ✅ 12 issue totali chiusi
- ✅ CI/CD pipeline attivo
- ✅ Documentation aggiornata

---

## Risorse Necessarie

### Team Composition
- **1 Senior Developer**: Issues critici e architetturali (#263, #273)
- **1-2 Mid-level Developers**: Bug fixes e features (#283, #286, #287, #289)
- **1 Technical Writer**: Documentation updates
- **1 QA/Tester**: Testing e regression

### Time Investment
- **Sprint 1-2**: 60-80 ore development
- **Sprint 3**: 20-30 ore development
- **Ongoing**: 10-15 ore/settimana per maintenance

---

## Raccomandazioni Strategiche

### Immediate Actions
1. ✅ **Release 1.1.18** con fix #278 (già presente nel fork)
2. ✅ **Prioritize #263** - dbt 1.10 blocca molti utenti
3. ✅ **Fix #283** - packages sono comuni

### Process Improvements
1. **Setup CI/CD**
   - Test automatici per ogni PR
   - Test matrix: Python 3.10-3.13, dbt 1.8-1.10
   - Prevent regressions

2. **Issue Triage Process**
   - Weekly issue review
   - Label issues (bug, enhancement, critical)
   - Response time SLA: 48h per critical

3. **Community Engagement**
   - Monthly release cadence
   - CHANGELOG updates
   - Migration guides

### Technical Debt
1. **Refactor osmosis.py** (~3000 lines)
   - Split into smaller modules
   - Improve testability
   - Consider #273 (dbt-core-interface)

2. **Increase Test Coverage**
   - Target: >80%
   - Focus on critical paths
   - Add integration tests

3. **Documentation**
   - Update for new features
   - Add troubleshooting guide
   - More examples

---

## Conclusioni

### Summary
- **31 issue aperti**, di cui **5 critici** che richiedono attenzione immediata
- **Issue #263 (dbt 1.10)** è il più impattante e dovrebbe essere priorità #1
- Molti issue hanno soluzioni chiare e relativamente semplici
- Il fork ha già risolto #278, serve solo release

### Success Path
1. ✅ Fix quick wins (#278, #274, #276) → release 1.1.18
2. ✅ Resolve critical blockers (#283, #286) → release 1.1.19
3. ✅ dbt 1.10 support (#263) → release 1.2.0 o 2.0.0
4. ✅ Improve inheritance (#289, #287) → release 1.2.1
5. ✅ Long-term enhancements → release 2.0.0

### Risk Factors
⚠️ **Maintainer Availability**: Progetto sembra in pausa, servono contributori attivi
⚠️ **dbt-core Changes**: Rapida evoluzione di dbt richiede adattamento continuo
⚠️ **Testing Coverage**: Coverage attuale sconosciuto, potrebbe causare regressioni

### Opportunities
✅ **Active Community**: 31 issue aperti indicano utenti engaged
✅ **Clear Solutions**: Molti issue hanno già proposte di fix
✅ **Fork Ahead**: Questo fork ha già alcuni fix non presenti nell'originale

---

**Prossimi Step Consigliati**:
1. Review questo report con team
2. Prioritize issue basati su business needs
3. Setup development environment
4. Start con Sprint 1 plan
5. Establish regular release cadence

**Contatto**: Per domande o chiarimenti su questo report, aprire discussion nel repository.
