# dbt-osmosis (fork: `sparodi63/dbt-osmosis`) — Technical Documentation

## 1. Overview

**dbt-osmosis** is a developer utility that automates the creation, organization, synchronization, and maintenance of dbt schema YAML files.  
It is designed to reduce manual documentation work and enforce consistency between dbt models, their underlying warehouse schemas, and project conventions.

The tool operates as:
- a **CLI utility** (primary usage)
- an optional **Streamlit workbench**
- a **dbt runtime companion** (loading manifest, catalog, adapter)

This documentation describes the internal architecture and behavior of the code under:

src/dbt_osmosis/

yaml
Copia codice

---

## 2. Core Responsibilities

dbt-osmosis performs five major tasks:

1. **YAML scaffolding**
   - Generate missing schema.yml files for models and sources

2. **YAML restructuring**
   - Move and merge YAML files according to configurable rules

3. **Schema synchronization**
   - Add missing columns
   - Remove obsolete columns
   - Sync data types from the warehouse or catalog

4. **Documentation propagation**
   - Inherit descriptions, tags, and metadata from upstream nodes

5. **Optional documentation synthesis**
   - Generate missing column descriptions using OpenAI

---

## 3. Package Structure

dbt_osmosis/
├── cli/
│ └── main.py # CLI entry points
├── core/
│ ├── osmosis.py # Main engine and pipelines
│ ├── llm.py # OpenAI integration
│ └── logger.py # Logging utilities
├── sql/
│ └── proxy.py # dbt SQL compile/execute abstraction
└── workbench/
└── app.py # Streamlit UI

yaml
Copia codice

---

## 4. CLI Commands

### 4.1 YAML Commands

| Command | Description |
|------|------------|
| `yaml refactor` | Full workflow: organize + sync + document |
| `yaml organize` | Only restructure YAML files |
| `yaml document` | Only propagate documentation |

### 4.2 SQL Commands

| Command | Description |
|------|------------|
| `sql compile` | Compile dbt SQL using runtime context |
| `sql execute` | Execute compiled SQL |

### 4.3 Workbench

dbt-osmosis workbench

less
Copia codice

Launches a Streamlit UI for:
- compiling dbt SQL
- executing queries
- basic data profiling

---

## 5. High-Level Architecture

### 5.1 End-to-End Flow (`yaml refactor`)

```mermaid
flowchart TD
  A[CLI Command] --> B[DbtConfiguration]
  B --> C[DbtProjectContext]
  C --> D[YamlRefactorContext]

  D --> E[Create Missing Source YAMLs]
  D --> F[Draft Restructure Plan]
  F --> G[Apply Restructure Plan]

  G --> H[Transform Pipeline]
  H --> I[Add Missing Columns]
  I --> J[Remove Obsolete Columns]
  J --> K[Inherit Documentation]
  K --> L[Sort Columns]
  L --> M[Sync Data Types]
  M --> N{Synthesize Docs?}
  N -->|Yes| O[OpenAI Synthesis]
  N -->|No| P[Finish]
6. Runtime Context Objects
6.1 DbtConfiguration
Encapsulates dbt runtime configuration:

project_dir

profiles_dir

target

profile

threads

vars

disable_introspection

6.2 DbtProjectContext
Represents a loaded dbt project:

Manifest

Adapter

dbt runtime configuration

Node graph and lineage helpers

Created by:

python
Copia codice
create_dbt_project_context(settings)
6.3 YamlRefactorSettings
Controls refactor behavior:

Node filters (fqn, models)

Skip flags:

skip_add_columns

skip_add_tags

skip_merge_meta

Inheritance controls:

force_inherit_descriptions

add_inheritance_for_specified_keys

Output normalization:

output_to_lower

Introspection configuration:

catalog_path

create_catalog_if_not_exists

6.4 YamlRefactorContext
Shared mutable state across operations:

dbt project context

settings

thread pool

YAML parser (ruamel.yaml)

mutation tracking

7. YAML Restructuring System
7.1 YAML Location Mapping
For each dbt node:

determine current YAML file

compute expected YAML file using templates

Key helpers:

get_current_yaml_path

get_target_yaml_path

build_yaml_file_mapping

7.2 Restructure Plan Model
Class	Responsibility
SchemaFileLocation	Node → YAML location mapping
SchemaFileMigration	YAML move/merge
RestructureOperation	Atomic filesystem/YAML change
RestructureDeltaPlan	Collection of operations

7.3 Plan and Apply
mermaid
Copia codice
sequenceDiagram
  participant CLI
  participant Context
  participant Planner
  participant Applier

  CLI->>Context: initialize
  Context->>Planner: draft plan
  Planner-->>Context: delta plan
  Context->>Applier: apply plan
  Applier-->>Context: updated YAMLs
8. Transform Pipeline
8.1 Pipeline Abstraction
TransformOperation

TransformPipeline

Supports chaining:

python
Copia codice
op1 >> op2 >> op3
8.2 Default Refactor Pipeline
inject_missing_columns

remove_columns_not_in_database

inherit_upstream_column_knowledge

sort_columns_as_configured

synchronize_data_types

optional synthesize_missing_documentation_with_openai

9. Core Transform Operations
9.1 Add Missing Columns
Adds YAML columns discovered via:

catalog.json

live adapter introspection

Controlled by:

skip_add_columns

skip_add_source_columns

9.2 Remove Obsolete Columns
Deletes YAML columns no longer present in the relation schema.

9.3 Inherit Upstream Documentation
Propagates:

descriptions

tags

metadata keys

Based on dbt lineage graph.

Key flags:

force_inherit_descriptions

skip_add_tags

skip_merge_meta

9.4 Synchronize Data Types
Populates:

data_type

precision and scale (if supported)

Normalizes casing when configured.

9.5 OpenAI Documentation Synthesis
Fills missing descriptions only

Enabled via CLI flag

Implemented in core/llm.py

10. SQL Subsystem
10.1 SQL Compilation
python
Copia codice
compile_sql_code(...)
Uses dbt compiler to resolve Jinja, refs, and macros.

10.2 SQL Execution
python
Copia codice
execute_sql_code(...)
Executes compiled SQL via dbt adapter.

10.3 DbtSession (sql/proxy.py)
Encapsulates:

project settings

adapter lifecycle

compile + execute helpers

Used by CLI and Workbench.

11. Workbench (Streamlit)
Located in:

bash
Copia codice
workbench/app.py
Provides:

SQL editor

compile button

execute button

profiling view

Launched with:

Copia codice
dbt-osmosis workbench
12. Maintenance Considerations
High-risk areas when upgrading dbt:

Manifest schema changes

Adapter introspection differences

YAML formatting stability

Lineage resolution edge cases

Catalog.json alignment

13. Conceptual Summary
dbt-osmosis is:

a dbt manifest interpreter

a YAML router

a filesystem migration engine

a schema synchronizer

a documentation propagation engine

optionally, an LLM-powered documentation assistant

14. Module Responsibility Table
Module	Responsibility
cli/main.py	CLI wiring
core/osmosis.py	Core engine
core/llm.py	OpenAI integration
sql/proxy.py	dbt SQL abstraction
workbench/app.py	Streamlit UI

End of documentation

markdown
Copia codice

If you want, next steps can include:
- generating an **architecture diagram (PNG/SVG)**
- annotating **each function with docstrings**
- producing a **developer maintenance guide**
- mapping **dbt-osmosis vs dbt-core versions compatibility**
- preparing a **refactor plan for Dremio / DuckDB / Iceberg use cases**
Hai detto:
provide a non markdown version for me to read here
ChatGPT ha detto:
Below is the same documentation content, rewritten as plain text (no Markdown, no code blocks), formatted for direct reading in the chat.

dbt-osmosis (fork: sparodi63/dbt-osmosis)
Technical Documentation

Overview

dbt-osmosis is a developer utility that automates the creation, organization, synchronization, and maintenance of dbt schema YAML files.
Its primary goal is to reduce manual documentation work and enforce consistency between dbt models, their underlying warehouse schemas, and project conventions.

The tool operates as:

a CLI utility (primary usage)

an optional Streamlit workbench

a dbt runtime companion that loads the manifest, catalog, and adapter

This documentation describes the internal architecture and behavior of the code under:
src/dbt_osmosis/

Core Responsibilities

dbt-osmosis performs five major tasks:

YAML scaffolding
Generates missing schema.yml files for models and sources.

YAML restructuring
Moves and merges YAML files according to configurable project rules.

Schema synchronization

Adds missing columns

Removes obsolete columns

Synchronizes data types from the warehouse or catalog

Documentation propagation
Inherits descriptions, tags, and metadata from upstream dbt nodes.

Optional documentation synthesis
Generates missing column descriptions using OpenAI.

Package Structure

The internal package layout is:

dbt_osmosis/

cli/main.py
CLI entry points and command wiring

core/osmosis.py
Main engine: context creation, restructuring logic, and transform pipelines

core/llm.py
OpenAI-based documentation synthesis logic

core/logger.py
Logging utilities

sql/proxy.py
Abstraction around dbt SQL compilation and execution

workbench/app.py
Streamlit UI for interactive SQL compilation and execution

CLI Commands

YAML-related commands:

yaml refactor
Executes the full workflow: YAML organization, schema synchronization, and documentation propagation.

yaml organize
Only restructures YAML files on disk; does not modify documentation.

yaml document
Only propagates documentation; does not move or restructure YAML files.

SQL-related commands:

sql compile
Compiles dbt SQL using the dbt runtime context.

sql execute
Executes compiled SQL via the dbt adapter.

Workbench command:

workbench
Launches the Streamlit UI for SQL editing, compilation, execution, and profiling.

High-Level Architecture

The main execution flow for “yaml refactor” is:

CLI parses arguments

A dbt configuration object is created

The dbt project context is loaded (manifest, adapter, graph)

A YAML refactor context is initialized

Missing source YAMLs are created if needed

A restructure plan is drafted

The restructure plan is applied

A transform pipeline runs to sync schema and documentation

Optional OpenAI synthesis fills missing descriptions

Runtime Context Objects

DbtConfiguration
This object encapsulates dbt runtime configuration, including:

project directory

profiles directory

target and profile

thread count

variables

flags controlling live introspection

DbtProjectContext
Represents a fully loaded dbt project. It contains:

the dbt manifest

the active adapter

runtime configuration

helper functions for graph traversal and lineage resolution

It is created by a factory function that initializes dbt internals.

YamlRefactorSettings
Controls how YAML refactoring behaves. It includes:

filters to limit which models are affected

flags to skip certain operations (columns, tags, metadata)

inheritance behavior settings

output normalization options

catalog and introspection configuration

YamlRefactorContext
A shared mutable context passed to all operations. It contains:

the dbt project context

refactor settings

a thread pool

a YAML parser instance

mutation tracking counters

YAML Restructuring System

For each dbt node, dbt-osmosis:

determines the current YAML file location

computes the expected YAML file location using templates

This mapping is used to decide whether a node’s documentation is already compliant or needs to be moved.

Key components:

SchemaFileLocation: represents where a node’s YAML currently lives versus where it should live

SchemaFileMigration: describes a move or merge of YAML content

RestructureOperation: an atomic filesystem/YAML change

RestructureDeltaPlan: a list of restructure operations

The plan is generated first and then applied, allowing dry-run or CI enforcement modes.

Transform Pipeline

dbt-osmosis uses a pipeline abstraction to chain transformations.

TransformOperation
Wraps a single operation that mutates YAML or metadata.

TransformPipeline
An ordered collection of operations that can be chained and executed sequentially.

The default refactor pipeline consists of:

Inject missing columns

Remove columns not present in the database

Inherit upstream documentation

Sort columns

Synchronize data types

Optionally synthesize missing documentation via OpenAI

Core Transform Operations

Add missing columns
Adds columns found in the warehouse schema or catalog that are missing from YAML files.

Remove obsolete columns
Deletes YAML columns that no longer exist in the underlying relation.

Inherit upstream documentation
Propagates column descriptions, tags, and selected metadata from upstream dbt nodes based on lineage.

Synchronize data types
Populates column data types, including precision and scale where available.

OpenAI documentation synthesis
Fills missing column descriptions only, never overwriting existing documentation, and is enabled explicitly via CLI flags.

SQL Subsystem

SQL compilation
Compiles dbt SQL by resolving Jinja, macros, and refs using dbt’s compiler.

SQL execution
Executes compiled SQL via the active dbt adapter.

DbtSession
A lightweight abstraction that manages dbt runtime context for compiling and executing SQL.
Used by both the CLI and the Streamlit workbench.

Workbench (Streamlit)

The workbench provides:

a SQL editor

compile action

execute action

basic profiling capabilities

It is designed for interactive exploration rather than production workflows.

Maintenance Considerations

When maintaining or extending dbt-osmosis, the most fragile areas are:

dbt manifest schema changes across versions

adapter-specific introspection behavior

YAML formatting stability

complex lineage resolution

catalog.json alignment with manifest data

Conceptual Summary

dbt-osmosis is best understood as:

a dbt manifest interpreter

a YAML routing and restructuring engine

a schema synchronization tool

a documentation propagation system

an optional LLM-powered documentation assistant

Module Responsibility Summary

cli/main.py
Handles CLI parsing and command orchestration.

core/osmosis.py
Implements the core refactor logic and transformation pipelines.

core/llm.py
Contains OpenAI integration logic.

sql/proxy.py
Provides dbt SQL compilation and execution abstractions.

workbench/app.py
Implements the Streamlit user interface.