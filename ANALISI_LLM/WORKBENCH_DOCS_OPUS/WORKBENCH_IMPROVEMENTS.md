# Workbench Improvements & Evolution Roadmap

## Sommario

1. [Analisi dello Stato Attuale](#1-analisi-dello-stato-attuale)
2. [Miglioramenti UI/UX](#2-miglioramenti-uiux)
3. [Nuove Funzionalità](#3-nuove-funzionalità)
4. [Ottimizzazioni Performance](#4-ottimizzazioni-performance)
5. [Miglioramenti Architetturali](#5-miglioramenti-architetturali)
6. [Integrazioni](#6-integrazioni)
7. [Piano di Implementazione](#7-piano-di-implementazione)

---

## 1. Analisi dello Stato Attuale

### Punti di Forza

| Aspetto | Valutazione |
|---------|-------------|
| Compilazione real-time | Ottimo |
| Integrazione dbt-core | Buono |
| UI drag & drop | Buono |
| Data profiling | Buono |

### Aree di Miglioramento

| Aspetto | Problema | Priorità |
|---------|----------|----------|
| **Nessun autocomplete** | L'editor non suggerisce ref(), source(), macro | Alta |
| **Nessuna history** | Le query non vengono salvate | Alta |
| **Feed RSS fisso** | Hacker News hardcoded, non configurabile | Bassa |
| **No multi-tab** | Non si possono aprire più query | Media |
| **No syntax validation** | Nessun linting SQL/Jinja | Media |
| **No database explorer** | Non si vedono tabelle/colonne | Alta |
| **No YAML editing** | Tab YAML non funzionale | Media |
| **No export** | Non si possono esportare risultati | Media |
| **No dark mode persistente** | Si resetta al refresh | Bassa |
| **No collaborative** | Single user only | Bassa |

---

## 2. Miglioramenti UI/UX

### 2.1 Database Explorer Panel

**Problema**: L'utente non può vedere la struttura del database.

**Soluzione**: Aggiungere un pannello laterale con albero navigabile.

```
┌─────────────────────────────────────────┐
│ 📊 Database Explorer                    │
├─────────────────────────────────────────┤
│ ▼ 📁 Sources                            │
│   ▼ 📦 raw                              │
│     ▼ 📋 customers                      │
│         id (INT)                        │
│         name (VARCHAR)                  │
│         email (VARCHAR)                 │
│     ▶ 📋 orders                         │
│   ▶ 📦 external                         │
│ ▼ 📁 Models                             │
│   ▶ 📂 staging                          │
│   ▼ 📂 marts                            │
│     ▶ 📋 dim_customers                  │
│     ▶ 📋 fct_orders                     │
└─────────────────────────────────────────┘
```

**Implementazione**:

```python
# components/explorer.py
@t.final
class DatabaseExplorer(Dashboard.Item):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._expanded_nodes = set()

    def _build_tree(self, ctx: DbtProject) -> dict:
        """Build tree structure from manifest and catalog."""
        tree = {"sources": {}, "models": {}}

        # Sources from manifest
        for source in ctx.manifest.sources.values():
            source_name = source.source_name
            if source_name not in tree["sources"]:
                tree["sources"][source_name] = {}
            tree["sources"][source_name][source.name] = {
                "columns": list(source.columns.keys()),
                "path": source.path,
            }

        # Models organized by folder
        for node in ctx.manifest.nodes.values():
            if node.resource_type == "model":
                folder = Path(node.original_file_path).parent.name
                if folder not in tree["models"]:
                    tree["models"][folder] = {}
                tree["models"][folder][node.name] = {
                    "columns": list(node.columns.keys()),
                    "path": node.original_file_path,
                }

        return tree

    def __call__(self):
        with mui.Paper(...):
            with self.title_bar():
                mui.icon.Storage()
                mui.Typography("Database Explorer")

            # Render tree with MUI TreeView
            with mui.TreeView(
                defaultCollapseIcon=mui.icon.ExpandMore,
                defaultExpandIcon=mui.icon.ChevronRight,
            ):
                self._render_tree(self._build_tree(state.app.ctx))
```

**Funzionalità aggiuntive**:
- Click su colonna → inserisce nome nell'editor
- Click su modello → inserisce `{{ ref('model') }}`
- Doppio click → apre il modello nell'editor
- Ricerca/filtro

---

### 2.2 Autocomplete Intelligente

**Problema**: Nessun suggerimento durante la digitazione.

**Soluzione**: Integrare LSP-like autocomplete nel Monaco editor.

```python
# Configurazione Monaco con autocomplete personalizzato
def get_autocomplete_suggestions(ctx: DbtProject) -> list[dict]:
    """Generate autocomplete items from dbt context."""
    suggestions = []

    # ref() suggestions
    for node in ctx.manifest.nodes.values():
        if node.resource_type == "model":
            suggestions.append({
                "label": f"ref('{node.name}')",
                "kind": "Function",
                "insertText": f"{{{{ ref('{node.name}') }}}}",
                "detail": f"Model: {node.original_file_path}",
                "documentation": node.description or "No description",
            })

    # source() suggestions
    for source in ctx.manifest.sources.values():
        suggestions.append({
            "label": f"source('{source.source_name}', '{source.name}')",
            "kind": "Function",
            "insertText": f"{{{{ source('{source.source_name}', '{source.name}') }}}}",
            "detail": f"Source: {source.source_name}.{source.name}",
        })

    # Macro suggestions
    for macro in ctx.manifest.macros.values():
        if macro.package_name == ctx.runtime_cfg.project_name:
            suggestions.append({
                "label": macro.name,
                "kind": "Function",
                "insertText": f"{{{{ {macro.name}() }}}}",
                "detail": f"Macro: {macro.original_file_path}",
            })

    return suggestions
```

**Integrazione Monaco**:

```javascript
// Custom completion provider
monaco.languages.registerCompletionItemProvider('sql', {
    triggerCharacters: ['{', "'", '"'],
    provideCompletionItems: (model, position) => {
        // Call Python backend for suggestions
        return { suggestions: window.dbtSuggestions };
    }
});
```

---

### 2.3 Query History

**Problema**: Le query eseguite vanno perse.

**Soluzione**: Salvare history locale con ricerca.

```python
# components/history.py
import json
from pathlib import Path
from datetime import datetime

HISTORY_FILE = Path.home() / ".dbt-osmosis" / "query_history.json"

@dataclass
class QueryHistoryItem:
    sql: str
    compiled_sql: str
    executed_at: datetime
    duration_ms: float
    row_count: int
    status: str  # "success", "error"
    error_message: str | None = None

class QueryHistory:
    def __init__(self, max_items: int = 100):
        self.max_items = max_items
        self._history: list[QueryHistoryItem] = []
        self._load()

    def _load(self):
        if HISTORY_FILE.exists():
            data = json.loads(HISTORY_FILE.read_text())
            self._history = [QueryHistoryItem(**item) for item in data]

    def _save(self):
        HISTORY_FILE.parent.mkdir(parents=True, exist_ok=True)
        data = [asdict(item) for item in self._history[-self.max_items:]]
        HISTORY_FILE.write_text(json.dumps(data, default=str))

    def add(self, item: QueryHistoryItem):
        self._history.append(item)
        self._save()

    def search(self, query: str) -> list[QueryHistoryItem]:
        return [h for h in self._history if query.lower() in h.sql.lower()]
```

**UI Component**:

```
┌─────────────────────────────────────────┐
│ 📜 Query History                   🔍   │
├─────────────────────────────────────────┤
│ [Search queries...]                     │
├─────────────────────────────────────────┤
│ ✅ 14:32 - SELECT * FROM customers...   │
│    Duration: 234ms | Rows: 1,523        │
├─────────────────────────────────────────┤
│ ✅ 14:28 - SELECT COUNT(*) FROM ord...  │
│    Duration: 45ms | Rows: 1             │
├─────────────────────────────────────────┤
│ ❌ 14:25 - SELECT * FROM {{ ref(...     │
│    Error: Model not found               │
└─────────────────────────────────────────┘
```

---

### 2.4 Multi-Tab Editor

**Problema**: Si può lavorare su una sola query alla volta.

**Soluzione**: Sistema di tab come VS Code.

```python
# Stato per multi-tab
@dataclass
class EditorTab:
    id: str
    title: str
    content: str
    language: str = "sql"
    is_dirty: bool = False
    model_path: str | None = None  # Se collegato a un modello

class MultiTabEditor(Dashboard.Item):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._tabs: list[EditorTab] = [
            EditorTab(id="scratch", title="SCRATCH", content=default_prompt)
        ]
        self._active_tab_id: str = "scratch"

    def add_tab(self, title: str, content: str = "", model_path: str | None = None):
        tab = EditorTab(
            id=str(uuid4()),
            title=title,
            content=content,
            model_path=model_path,
        )
        self._tabs.append(tab)
        self._active_tab_id = tab.id

    def close_tab(self, tab_id: str):
        if len(self._tabs) > 1:
            self._tabs = [t for t in self._tabs if t.id != tab_id]
            if self._active_tab_id == tab_id:
                self._active_tab_id = self._tabs[0].id
```

**UI**:

```
┌─────────────────────────────────────────────────────────────┐
│ [SCRATCH] [customers.sql ●] [orders.sql] [+]               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SELECT                                                     │
│      customer_id,                                           │
│      customer_name                                          │
│  FROM {{ ref('stg_customers') }}                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 2.5 Result Export

**Problema**: Non si possono esportare i risultati.

**Soluzione**: Aggiungere export in vari formati.

```python
def export_results(df: pd.DataFrame, format: str) -> bytes:
    """Export DataFrame to various formats."""
    if format == "csv":
        return df.to_csv(index=False).encode()
    elif format == "json":
        return df.to_json(orient="records").encode()
    elif format == "parquet":
        buffer = io.BytesIO()
        df.to_parquet(buffer, index=False)
        return buffer.getvalue()
    elif format == "excel":
        buffer = io.BytesIO()
        df.to_excel(buffer, index=False)
        return buffer.getvalue()

# Nel Preview component
with mui.Stack(direction="row", spacing=2):
    mui.Button("Run Query", onClick=run_query)

    # Export dropdown
    with mui.Menu():
        mui.MenuItem("Export as CSV", onClick=lambda: export("csv"))
        mui.MenuItem("Export as JSON", onClick=lambda: export("json"))
        mui.MenuItem("Export as Parquet", onClick=lambda: export("parquet"))
        mui.MenuItem("Export as Excel", onClick=lambda: export("excel"))
```

---

## 3. Nuove Funzionalità

### 3.1 SQL Linting & Validation

**Descrizione**: Validazione SQL in tempo reale con suggerimenti.

```python
# Integrazione con sqlfluff
from sqlfluff.core import Linter

def lint_sql(sql: str, dialect: str = "ansi") -> list[dict]:
    """Lint SQL and return issues."""
    linter = Linter(dialect=dialect)
    result = linter.lint_string(sql)

    issues = []
    for violation in result.violations:
        issues.append({
            "line": violation.line_no,
            "column": violation.line_pos,
            "message": violation.desc,
            "severity": "warning" if violation.warning else "error",
            "rule": violation.rule_code,
        })
    return issues
```

**UI**: Sottolineature rosse/gialle nell'editor + pannello errori.

```
┌─────────────────────────────────────────┐
│ ⚠️ SQL Issues (3)                       │
├─────────────────────────────────────────┤
│ ⚠ Line 5: Trailing whitespace          │
│ ❌ Line 8: Ambiguous column reference   │
│ ⚠ Line 12: Use explicit column list    │
└─────────────────────────────────────────┘
```

---

### 3.2 Query Explain / Execution Plan

**Descrizione**: Visualizzare il piano di esecuzione della query.

```python
def get_query_plan(ctx: DbtProject, sql: str) -> dict:
    """Get query execution plan."""
    explain_sql = f"EXPLAIN {sql}"
    try:
        _, table = execute_sql_code(ctx, explain_sql)
        return parse_explain_output(table)
    except Exception as e:
        return {"error": str(e)}

def parse_explain_output(table) -> dict:
    """Parse EXPLAIN output into structured format."""
    # Adapter-specific parsing
    # Returns tree structure for visualization
    pass
```

**UI**: Visualizzazione grafica del piano.

```
┌─────────────────────────────────────────────────────────────┐
│ 📊 Query Plan                                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐                                        │
│  │ Seq Scan        │ Cost: 0.00..35.50                      │
│  │ customers       │ Rows: 1000                             │
│  └────────┬────────┘                                        │
│           │                                                 │
│  ┌────────▼────────┐                                        │
│  │ Hash Join       │ Cost: 35.50..125.00                    │
│  │                 │ Rows: 5000                             │
│  └────────┬────────┘                                        │
│           │                                                 │
│  ┌────────▼────────┐                                        │
│  │ Sort            │ Cost: 125.00..130.00                   │
│  │ order_date      │ Rows: 5000                             │
│  └─────────────────┘                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 3.3 Saved Queries / Snippets

**Descrizione**: Salvare e organizzare query frequenti.

```python
@dataclass
class SavedQuery:
    id: str
    name: str
    description: str
    sql: str
    tags: list[str]
    created_at: datetime
    updated_at: datetime
    folder: str = "Default"

class QueryLibrary:
    """Manage saved queries."""

    def __init__(self):
        self._queries: dict[str, SavedQuery] = {}
        self._load()

    def save(self, name: str, sql: str, description: str = "", tags: list[str] = None):
        query = SavedQuery(
            id=str(uuid4()),
            name=name,
            description=description,
            sql=sql,
            tags=tags or [],
            created_at=datetime.now(),
            updated_at=datetime.now(),
        )
        self._queries[query.id] = query
        self._persist()

    def search(self, term: str) -> list[SavedQuery]:
        return [
            q for q in self._queries.values()
            if term.lower() in q.name.lower()
            or term.lower() in q.description.lower()
            or any(term.lower() in t.lower() for t in q.tags)
        ]
```

**UI**:

```
┌─────────────────────────────────────────┐
│ 📁 Query Library                   [+]  │
├─────────────────────────────────────────┤
│ [🔍 Search...]                          │
├─────────────────────────────────────────┤
│ 📂 Analytics                            │
│   📄 Daily Revenue                      │
│   📄 Customer Segments                  │
│ 📂 Debugging                            │
│   📄 Find Orphan Records                │
│   📄 Data Quality Checks                │
│ 📂 Templates                            │
│   📄 Incremental Model                  │
│   📄 Snapshot Template                  │
└─────────────────────────────────────────┘
```

---

### 3.4 Diff View per Modelli

**Descrizione**: Vedere le differenze tra versione salvata e modificata.

```python
# Integrazione diff
import difflib

def generate_diff(original: str, modified: str) -> str:
    """Generate unified diff."""
    diff = difflib.unified_diff(
        original.splitlines(keepends=True),
        modified.splitlines(keepends=True),
        fromfile="Original",
        tofile="Modified",
    )
    return "".join(diff)

# Monaco diff editor
editor.MonacoDiff(
    original=state.app.original_content,
    modified=state.app.current_content,
    language="sql",
    theme="vs-dark",
)
```

---

### 3.5 Query Scheduling / Automation

**Descrizione**: Pianificare esecuzione query ricorrenti.

```python
from apscheduler.schedulers.background import BackgroundScheduler

class QueryScheduler:
    def __init__(self):
        self._scheduler = BackgroundScheduler()
        self._jobs: dict[str, dict] = {}

    def schedule(
        self,
        query_id: str,
        sql: str,
        cron: str,  # "0 9 * * *" = ogni giorno alle 9
        notify_email: str | None = None,
    ):
        job = self._scheduler.add_job(
            self._execute_scheduled,
            trigger=CronTrigger.from_crontab(cron),
            args=[query_id, sql, notify_email],
            id=query_id,
        )
        self._jobs[query_id] = {
            "sql": sql,
            "cron": cron,
            "next_run": job.next_run_time,
        }

    def _execute_scheduled(self, query_id: str, sql: str, notify_email: str | None):
        """Execute scheduled query and optionally notify."""
        try:
            result = execute_sql_code(self._ctx, sql)
            if notify_email:
                self._send_notification(notify_email, query_id, result)
        except Exception as e:
            logger.error(f"Scheduled query {query_id} failed: {e}")
```

---

### 3.6 Collaborative Features

**Descrizione**: Condivisione query e collaborazione real-time.

**Funzionalità**:
- Condividi query via link
- Commenti su query
- Cursori multipli (like Google Docs)
- Chat integrata

**Tecnologie necessarie**:
- WebSocket per real-time
- Redis per stato condiviso
- Database per persistenza

---

### 3.7 AI Assistant Integrato

**Descrizione**: Assistente AI per SQL e dbt.

```python
# Integrazione con LLM module esistente
from dbt_osmosis.core.llm import get_llm_client

async def ai_suggest_query(prompt: str, context: dict) -> str:
    """Generate SQL query from natural language."""
    client, model = get_llm_client()

    messages = [
        {
            "role": "system",
            "content": f"""You are a SQL expert for dbt projects.
Available models: {context['models']}
Available sources: {context['sources']}
Generate only SQL code, no explanations."""
        },
        {"role": "user", "content": prompt}
    ]

    response = client.chat.completions.create(
        model=model,
        messages=messages,
        temperature=0.3,
    )
    return response.choices[0].message.content

async def ai_explain_query(sql: str) -> str:
    """Explain what a SQL query does."""
    # ...

async def ai_optimize_query(sql: str) -> str:
    """Suggest optimizations for a query."""
    # ...
```

**UI**:

```
┌─────────────────────────────────────────────────────────────┐
│ 🤖 AI Assistant                                             │
├─────────────────────────────────────────────────────────────┤
│ [Write a query to find customers who ordered last month]   │
│                                                    [Ask AI] │
├─────────────────────────────────────────────────────────────┤
│ 💬 Generated SQL:                                           │
│                                                             │
│ SELECT c.customer_id, c.name                               │
│ FROM {{ ref('dim_customers') }} c                          │
│ JOIN {{ ref('fct_orders') }} o ON c.customer_id = o.cid    │
│ WHERE o.order_date >= DATE_TRUNC('month', CURRENT_DATE)    │
│   AND o.order_date < CURRENT_DATE                          │
│                                                             │
│                              [Insert] [Explain] [Optimize] │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Ottimizzazioni Performance

### 4.1 Lazy Loading dei Risultati

**Problema**: Query con molti risultati rallentano l'UI.

**Soluzione**: Paginazione server-side.

```python
class LazyQueryResult:
    """Paginated query results."""

    def __init__(self, ctx: DbtProject, sql: str, page_size: int = 100):
        self._ctx = ctx
        self._sql = sql
        self._page_size = page_size
        self._total_count: int | None = None
        self._cache: dict[int, pd.DataFrame] = {}

    def get_page(self, page: int) -> pd.DataFrame:
        if page in self._cache:
            return self._cache[page]

        offset = page * self._page_size
        paginated_sql = f"""
        SELECT * FROM ({self._sql}) AS _q
        LIMIT {self._page_size} OFFSET {offset}
        """
        _, table = execute_sql_code(self._ctx, paginated_sql)
        df = pd.DataFrame(...)
        self._cache[page] = df
        return df

    @property
    def total_count(self) -> int:
        if self._total_count is None:
            count_sql = f"SELECT COUNT(*) FROM ({self._sql}) AS _q"
            _, table = execute_sql_code(self._ctx, count_sql)
            self._total_count = table.rows[0][0]
        return self._total_count
```

---

### 4.2 Query Result Caching

**Problema**: Stessa query eseguita più volte.

**Soluzione**: Cache con invalidazione intelligente.

```python
from functools import lru_cache
from hashlib import sha256

class QueryCache:
    def __init__(self, max_size: int = 50, ttl_seconds: int = 300):
        self._cache: OrderedDict[str, tuple[datetime, pd.DataFrame]] = OrderedDict()
        self._max_size = max_size
        self._ttl = timedelta(seconds=ttl_seconds)

    def _hash_query(self, sql: str, target: str) -> str:
        return sha256(f"{sql}:{target}".encode()).hexdigest()

    def get(self, sql: str, target: str) -> pd.DataFrame | None:
        key = self._hash_query(sql, target)
        if key in self._cache:
            timestamp, df = self._cache[key]
            if datetime.now() - timestamp < self._ttl:
                # Move to end (LRU)
                self._cache.move_to_end(key)
                return df
            else:
                del self._cache[key]
        return None

    def set(self, sql: str, target: str, df: pd.DataFrame):
        key = self._hash_query(sql, target)
        if len(self._cache) >= self._max_size:
            self._cache.popitem(last=False)  # Remove oldest
        self._cache[key] = (datetime.now(), df)
```

---

### 4.3 Compilation Caching

**Problema**: Ricompilazione ad ogni keystroke.

**Soluzione**: Debounce + cache.

```python
from threading import Timer

class DebouncedCompiler:
    def __init__(self, compile_fn: Callable, delay: float = 0.5):
        self._compile_fn = compile_fn
        self._delay = delay
        self._timer: Timer | None = None
        self._cache: dict[str, str] = {}

    def compile(self, sql: str) -> None:
        # Cancel previous timer
        if self._timer:
            self._timer.cancel()

        # Check cache
        if sql in self._cache:
            state.app.compiled_query = self._cache[sql]
            return

        # Schedule compilation
        self._timer = Timer(self._delay, self._do_compile, [sql])
        self._timer.start()

    def _do_compile(self, sql: str):
        result = self._compile_fn(sql)
        self._cache[sql] = result
        state.app.compiled_query = result
```

---

### 4.4 Streaming Results

**Problema**: Attesa lunga per query grandi.

**Soluzione**: Streaming progressivo.

```python
async def stream_query_results(ctx: DbtProject, sql: str):
    """Stream query results as they arrive."""
    # Usa cursor server-side se supportato
    connection = ctx.adapter.get_connection()

    with connection.cursor() as cursor:
        cursor.execute(sql)

        columns = [desc[0] for desc in cursor.description]
        yield {"type": "columns", "data": columns}

        batch_size = 100
        while True:
            rows = cursor.fetchmany(batch_size)
            if not rows:
                break
            yield {"type": "rows", "data": rows}
```

---

## 5. Miglioramenti Architetturali

### 5.1 Separazione Backend/Frontend

**Problema attuale**: Tutto in Streamlit, difficile da testare e scalare.

**Soluzione**: Architettura API-based.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend                                 │
│                    (React / Streamlit)                          │
└───────────────────────────────┬─────────────────────────────────┘
                                │ REST/WebSocket
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                         API Layer                                │
│                    (FastAPI / Flask)                            │
├─────────────────────────────────────────────────────────────────┤
│ GET  /api/models              - List models                     │
│ GET  /api/models/{name}       - Get model details               │
│ POST /api/compile             - Compile SQL                     │
│ POST /api/execute             - Execute query                   │
│ GET  /api/history             - Query history                   │
│ WS   /api/ws                  - Real-time updates               │
└───────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      dbt-osmosis Core                            │
└─────────────────────────────────────────────────────────────────┘
```

**API FastAPI**:

```python
# api/main.py
from fastapi import FastAPI, WebSocket
from pydantic import BaseModel

app = FastAPI(title="dbt-osmosis Workbench API")

class CompileRequest(BaseModel):
    sql: str
    target: str | None = None

class CompileResponse(BaseModel):
    compiled_sql: str
    error: str | None = None

@app.post("/api/compile", response_model=CompileResponse)
async def compile_sql(request: CompileRequest):
    try:
        result = compile_sql_code(ctx, request.sql)
        return CompileResponse(compiled_sql=result.compiled_code)
    except Exception as e:
        return CompileResponse(compiled_sql="", error=str(e))

@app.websocket("/api/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_json()
        if data["type"] == "compile":
            result = compile_sql_code(ctx, data["sql"])
            await websocket.send_json({
                "type": "compiled",
                "sql": result.compiled_code,
            })
```

---

### 5.2 Plugin System per Componenti

**Descrizione**: Permettere componenti custom.

```python
# Plugin interface
class WorkbenchPlugin(ABC):
    @property
    @abstractmethod
    def name(self) -> str:
        """Plugin name."""

    @property
    @abstractmethod
    def component(self) -> type[Dashboard.Item]:
        """Dashboard component class."""

    @property
    def sidebar_section(self) -> Callable | None:
        """Optional sidebar section."""
        return None

# Plugin registration
class PluginManager:
    def __init__(self):
        self._plugins: list[WorkbenchPlugin] = []

    def register(self, plugin: WorkbenchPlugin):
        self._plugins.append(plugin)

    def load_from_entrypoints(self):
        """Load plugins from setuptools entrypoints."""
        for ep in importlib.metadata.entry_points(group="dbt_osmosis.workbench"):
            plugin_class = ep.load()
            self.register(plugin_class())
```

**pyproject.toml del plugin**:

```toml
[project.entry-points."dbt_osmosis.workbench"]
my_plugin = "my_package:MyWorkbenchPlugin"
```

---

### 5.3 Configurazione Esternalizzata

**Problema**: Configurazioni hardcoded.

**Soluzione**: File di configurazione.

```yaml
# ~/.dbt-osmosis/workbench.yaml
workbench:
  theme: dark
  layout:
    editor:
      position: [0, 0]
      size: [6, 11]
    preview:
      position: [0, 11]
      size: [12, 9]

  features:
    realtime_compilation: true
    autocomplete: true
    linting: true

  feed:
    enabled: true
    url: "https://news.ycombinator.com/rss"  # Customizable!

  history:
    max_items: 100
    persist: true

  cache:
    query_ttl: 300
    max_cached_queries: 50
```

---

## 6. Integrazioni

### 6.1 IDE Integration (VS Code Extension)

**Descrizione**: Estensione VS Code per preview in-editor.

```typescript
// extension.ts
import * as vscode from 'vscode';

export function activate(context: vscode.ExtensionContext) {
    let panel: vscode.WebviewPanel | undefined;

    context.subscriptions.push(
        vscode.commands.registerCommand('dbt-osmosis.preview', () => {
            panel = vscode.window.createWebviewPanel(
                'dbtPreview',
                'dbt Preview',
                vscode.ViewColumn.Beside,
                { enableScripts: true }
            );

            // Load workbench in webview
            panel.webview.html = getWorkbenchHtml();
        })
    );

    // Auto-compile on save
    vscode.workspace.onDidSaveTextDocument((doc) => {
        if (doc.fileName.endsWith('.sql')) {
            compileAndUpdatePreview(doc.getText());
        }
    });
}
```

---

### 6.2 dbt Cloud Integration

**Descrizione**: Connessione a dbt Cloud per progetti cloud.

```python
class DbtCloudConnection:
    def __init__(self, account_id: str, api_token: str):
        self._account_id = account_id
        self._api_token = api_token
        self._base_url = f"https://cloud.getdbt.com/api/v2/accounts/{account_id}"

    async def get_manifest(self, project_id: str, environment_id: str) -> dict:
        """Fetch manifest from dbt Cloud."""
        url = f"{self._base_url}/projects/{project_id}/artifacts/manifest.json"
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                url,
                headers={"Authorization": f"Token {self._api_token}"},
                params={"environment_id": environment_id},
            )
            return resp.json()

    async def run_query(self, sql: str, warehouse_id: str) -> dict:
        """Execute query via dbt Cloud."""
        # Use dbt Cloud's query API
        pass
```

---

### 6.3 Git Integration

**Descrizione**: Visualizzare stato Git dei modelli.

```python
import git

class GitIntegration:
    def __init__(self, repo_path: str):
        self._repo = git.Repo(repo_path)

    def get_file_status(self, path: str) -> str:
        """Get git status of a file."""
        # Returns: "modified", "staged", "untracked", "clean"
        if path in self._repo.untracked_files:
            return "untracked"
        diff_staged = self._repo.index.diff("HEAD")
        diff_unstaged = self._repo.index.diff(None)
        # ...

    def get_file_history(self, path: str, limit: int = 10) -> list[dict]:
        """Get commit history for a file."""
        commits = []
        for commit in self._repo.iter_commits(paths=path, max_count=limit):
            commits.append({
                "sha": commit.hexsha[:7],
                "message": commit.message.strip(),
                "author": commit.author.name,
                "date": commit.committed_datetime,
            })
        return commits
```

**UI**: Badge colorati sui modelli nell'explorer.

---

## 7. Piano di Implementazione

### Fase 1: Quick Wins (1-2 settimane)

| Feature | Effort | Impatto |
|---------|--------|---------|
| Query History | 3 giorni | Alto |
| Result Export (CSV/JSON) | 1 giorno | Medio |
| Configurazione Feed RSS | 0.5 giorni | Basso |
| Dark mode persistente | 0.5 giorni | Basso |
| Keyboard shortcuts aggiuntivi | 1 giorno | Medio |

### Fase 2: Core Improvements (3-4 settimane)

| Feature | Effort | Impatto |
|---------|--------|---------|
| Database Explorer | 1 settimana | Alto |
| Autocomplete base | 1 settimana | Alto |
| Multi-tab editor | 1 settimana | Alto |
| Query caching | 3 giorni | Medio |

### Fase 3: Advanced Features (2-3 mesi)

| Feature | Effort | Impatto |
|---------|--------|---------|
| SQL Linting | 2 settimane | Medio |
| Query Plan visualization | 2 settimane | Medio |
| Saved Queries library | 2 settimane | Alto |
| AI Assistant | 3 settimane | Alto |

### Fase 4: Architecture Evolution (3-6 mesi)

| Feature | Effort | Impatto |
|---------|--------|---------|
| API-based backend | 2 mesi | Alto |
| Plugin system | 1 mese | Medio |
| VS Code extension | 1 mese | Medio |
| Collaborative features | 2 mesi | Medio |

---

## Appendice: Priorità Features

### Must Have (P0)

- Database Explorer
- Autocomplete per ref/source
- Query History
- Export risultati

### Should Have (P1)

- Multi-tab editor
- SQL Linting
- Saved Queries
- Query caching

### Nice to Have (P2)

- AI Assistant
- Query scheduling
- Git integration
- Collaborative editing

### Future (P3)

- VS Code extension
- dbt Cloud integration
- Plugin system
- Full API separation

---

*Documento di miglioramenti per dbt-osmosis Workbench*
*Creato: Dicembre 2024*
