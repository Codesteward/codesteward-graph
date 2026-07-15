# CLAUDE.md — Codesteward MCP Graph Server

## What This Is

This is the **Phase 1 standalone repository** of the Codesteward platform.  It is a self-contained [Model Context Protocol](https://modelcontextprotocol.io) server that parses source repositories into a structural code graph and exposes that graph as queryable tools for AI agents.

It has **no dependency** on the broader Codesteward agent platform (no deepagents, no LangChain, no workflows, no policy engine, no PostgreSQL).  For graph storage it supports three backends: Neo4j, JanusGraph (Apache 2.0), or GraphQLite (embedded SQLite — no server needed, ideal for local dev via `uvx`).

## Architecture

```text
codesteward/                             # repo root (workspace root, not published)
├── pyproject.toml                       # uv workspace — members: packages/*
├── docker-compose.neo4j.yml              # Neo4j stack
├── docker-compose.janusgraph.yml        # JanusGraph stack (Apache 2.0 alternative)
├── Dockerfile.mcp                       # Multi-stage build (supports both backends)
├── .gitlab-ci.yml                       # CI: test → build → publish PyPI → release GitHub
├── CHANGELOG.md
├── packages/
│   ├── codesteward-graph/               # PyPI: codesteward-graph
│   │   ├── pyproject.toml
│   │   └── src/codesteward/
│   │       └── engine/
│   │           ├── graph_builder.py     # Orchestrates parse + graph write (backend-agnostic)
│   │           ├── backends/            # Graph database backend abstraction
│   │           │   ├── base.py          # GraphBackend ABC
│   │           │   ├── neo4j.py         # Neo4j backend (Cypher queries)
│   │           │   ├── janusgraph.py    # JanusGraph backend (Gremlin queries)
│   │           │   └── graphqlite.py    # GraphQLite backend (Cypher over SQLite — embedded)
│   │           ├── tree_sitter_parser.py
│   │           └── parsers/             # Per-language tree-sitter parsers (14 languages)
│   │               ├── base.py          # LanguageParser ABC + LexicalNode/GraphEdge models
│   │               ├── python.py        # GUARDED_BY (decorators, Depends), PROTECTED_BY (APIRouter), taint sources
│   │               ├── typescript.py    # GUARDED_BY (decorators), PROTECTED_BY (router.use), taint sources (Express/NestJS)
│   │               ├── java.py          # GUARDED_BY (annotations), taint sources (Spring MVC/Jakarta EE)
│   │               ├── go.py            # PROTECTED_BY (Gin/Echo group.Use), taint sources (net/http/Gin)
│   │               ├── rust.py          # PROTECTED_BY (Actix scope().wrap()), taint sources (Actix-web/Axum extractors)
│   │               ├── php.py           # PROTECTED_BY (Laravel Route::middleware->group), taint sources (superglobals/Laravel/Symfony/PSR-7)
│   │               ├── csharp.py        # PROTECTED_BY (ASP.NET MapGroup().RequireAuthorization), taint sources ([FromQuery]/HttpRequest)
│   │               ├── kotlin.py        # GUARDED_BY (Spring annotations), taint sources (Spring/Ktor/Http4k)
│   │               ├── scala.py         # taint sources (Play/Akka HTTP)
│   │               ├── c.py             # taint sources (CGI/Mongoose/libmicrohttpd)
│   │               ├── cpp.py           # taint sources (Crow/Drogon/Pistache/Oat++)
│   │               ├── cobol.py         # regex-based (no tree-sitter grammar)
│   │               └── _ast_utils.py    # Shared tree-sitter helpers + TreeSitterBase
│   └── codesteward-mcp/                 # PyPI: codesteward-mcp
│       ├── pyproject.toml               # depends on codesteward-graph
│       └── src/codesteward/
│           └── mcp/
│               ├── config.py            # McpConfig (pydantic-settings) + load_config(); auto-detects backend
│               ├── server.py            # FastMCP server, 5 tools (4 core + optional taint), --transport http|stdio
│               ├── setup.py             # `codesteward-mcp setup` — auto-detect tools, register globally
│               └── tools/
│                   ├── graph.py         # Tool implementations (plain async fns)
│                   └── taint.py         # taint_analysis tool (invokes codesteward-taint binary)
└── tests/                               # All tests run from workspace root
    ├── conftest.py                      # cfg / cfg_with_neo4j / cfg_with_janusgraph fixtures
    ├── test_engine/                     # GraphBuilder + TreeSitterParser tests
    │   ├── test_graph_builder.py
    │   ├── test_tree_sitter_parser.py
    │   └── test_taint_sources.py        # Taint-source emission tests for all parsers
    └── test_mcp/
        └── test_graph_tools.py          # Tests for all 4 MCP tool functions
```

## Key Design Decisions

### Parser architecture

All parsers live in `parsers/` and are tree-sitter based — they subclass
`TreeSitterBase` from `_ast_utils.py` and require the language-specific
grammar package to be installed.  **COBOL is the only exception**: no
tree-sitter grammar is available for it, so `CobolParser` uses regex.

`GraphBuilder` dispatches via `get_parser(language)` from the parsers registry.
There is no automatic fallback from tree-sitter to regex — if a grammar
package is not installed, parsing that language will raise `ImportError`.

The file `engine/tree_sitter_parser.py` is present but not used by
`GraphBuilder` — it is an older standalone probe used only in the test suite.

### Tool implementations are plain async functions

The functions in `tools/graph.py` and `tools/taint.py` are plain `async def` — not LangChain `@tool` decorators.  `server.py` wraps them as FastMCP tools using `@mcp.tool()`.  This keeps the tool logic testable in isolation without an MCP runtime.

The `taint_analysis` tool in `server.py` is registered conditionally: `shutil.which("codesteward-taint")` is called at import time; if the binary is absent the tool is simply not registered and the server starts with the four core tools only.

### Graph backend auto-detection

When `GRAPH_BACKEND` is not explicitly set (default `auto`), the server
auto-detects the backend:

1. `NEO4J_PASSWORD` is set → Neo4j
2. `JANUSGRAPH_URL` is non-default → JanusGraph
3. Otherwise → **GraphQLite** (embedded SQLite, persistent, no server needed)

This means zero-config `uvx` usage defaults to GraphQLite with a persistent
graph at `~/.codesteward/graph.db`.

Stub mode still exists: set `GRAPH_BACKEND=neo4j` with no password to get
parse-only behaviour (nothing persisted, queries return `stub: true`).

### Graph backend abstraction

`engine/backends/` provides a `GraphBackend` ABC with three implementations:

| Backend | Module | Query language | Server required | License |
| ------- | ------ | -------------- | --------------- | ------- |
| Neo4j | `backends/neo4j.py` | Cypher | Yes (bolt) | AGPL / Commercial |
| JanusGraph | `backends/janusgraph.py` | Gremlin (TinkerPop) | Yes (WebSocket) | Apache 2.0 |
| GraphQLite | `backends/graphqlite.py` | Cypher (over SQLite) | No (embedded) | MIT |

Selected via `GRAPH_BACKEND=neo4j`, `GRAPH_BACKEND=janusgraph`,
`GRAPH_BACKEND=graphqlite`, or `GRAPH_BACKEND=auto` (default — resolves to
GraphQLite unless Neo4j/JanusGraph credentials are detected).

`_make_backend(cfg)` in `tools/graph.py` is the factory used by all tool functions.
`GraphBuilder` accepts an optional `backend` parameter and wraps it in `GraphWriter`
(backend-agnostic) or falls back to the legacy `Neo4jWriter` when a raw `neo4j_driver`
is passed.

Named query templates (`lexical`, `referential`, `semantic`, `dependency`) are
reimplemented in each backend's native query language.  Neo4j uses standard Cypher.
GraphQLite builds each query as a Cypher string with escaped literal values (via
`_cypher_escape`) because `$param` interpolation in MATCH property patterns is
unreliable in GraphQLite.  The `referential`, `semantic`, and `dependency` templates
all read target metadata (`target_name`, `target_id`) from edge properties rather
than target nodes, because GraphQLite traversal does not resolve target node
properties (`tgt.*` returns null through `(src)-[r]->(tgt)` patterns).  The
`semantic` template also uses `r.sanitized = 0` instead of `NOT r.sanitized` because
SQLite stores booleans as integers and GraphQLite's Cypher translation of `NOT <int>`
is unreliable.  JanusGraph uses Gremlin equivalents.  Raw query passthrough surfaces
Cypher (Neo4j, GraphQLite) or Gremlin (JanusGraph) — determined by
`backend.raw_query_language`.

The GraphQLite backend wraps synchronous SQLite calls with `asyncio.to_thread()`
to satisfy the async `GraphBackend` interface.  The database is a single `.db` file
(defaults to `~/.codesteward/graph.db`); set `GRAPHQLITE_DB_PATH` to override.

Edges in the GraphQLite backend store `target_name` and `target_id` as relationship
properties.  This is redundant with the target node but necessary because GraphQLite's
`(src)-[r]->(tgt)` pattern does not resolve `tgt.*` properties.  The `write_edges`
method also creates target nodes one at a time with literal values (not UNWIND)
because UNWIND + ON CREATE SET does not persist properties in GraphQLite.

### Multi-tenancy

Every node and edge carries `tenant_id` + `repo_id`.  All graph queries filter by both.  Each `tool_*` function accepts them as parameters; the server defaults them to `cfg.default_tenant_id` / `cfg.default_repo_id`.

### Ignored directories

`GraphBuilder._collect_files()` skips directories listed in `_IGNORED_DIRS`:
`node_modules`, `dist`, `build`, `.next`, `.nuxt`, `coverage`, `__pycache__`,
`.git`, `.venv`, `venv`, `.env`, `env`, `.tox`, `.nox`, `.mypy_cache`,
`.ruff_cache`, `.pytest_cache`, `site-packages`, `.eggs`, `*.egg-info`.

### CALLS cross-file resolution

After all files are parsed, `GraphBuilder._resolve_call_targets()` builds a
`fn_name → node_id` map from all function and class nodes, then rewrites
CALLS edge `target_id` values from bare callee names to proper node IDs.
Ambiguous names (multiple nodes with the same short name) are left
unresolved — they become `external` placeholder nodes in the backend.
This is a single post-parse pass; no repeated scanning.  Typically resolves
~30% of all CALLS edges in a codebase.

### Auth guard edges

Two edge types represent auth guards at different granularities:

| Edge | Granularity | Examples |
| ---- | ----------- | -------- |
| `GUARDED_BY` | Function-level | Python `@login_required`, FastAPI `Depends(auth)`, `@UseGuards(JwtGuard)`, `@PreAuthorize(...)` |
| `PROTECTED_BY` | Router/group-scope | FastAPI `APIRouter(dependencies=[Depends(auth)])`, Express `router.use(auth)`, Gin `api.Use(auth)`, Actix `scope().wrap(auth)`, Laravel `Route::middleware(['auth'])->group(...)`, ASP.NET `MapGroup("/api").RequireAuthorization()` |

Both resolve unrecognised/external guard names to `external` LexicalNodes so they remain queryable.

### Taint-source nodes

Every parser emits synthetic `external` `LexicalNode` entries for HTTP request input points
(e.g. `$_GET`, `req.body`, `web::Json<T>`, `getenv("QUERY_STRING")`).  Each source node has
`node_type="external"` and a `file` field that identifies the framework (e.g. `"php"`,
`"actix_axum"`, `"c_http"`).  A reverse-direction `CALLS` edge `(source) -[:CALLS]-> (handler)`
is also emitted, giving the `codesteward-taint` binary a starting point for L1 traversal
(`source → handler → sink`) without requiring a separate annotation pass.

### Taint-flow edges

`TAINT_FLOW` edges are **not** emitted by any parser.  They are written exclusively by the
`codesteward-taint` Go binary (separate repository: `Codesteward/codesteward-taint`).  The MCP
server invokes it via `tools/taint.py` → `asyncio.create_subprocess_exec`.  The `semantic`
query template in each backend reads `TAINT_FLOW` edges; it returns empty results until
`taint_analysis` has been run.

## Coding Standards

Follow the same standards as the main Codesteward project:

- Python 3.12+ — type hints, `match` statements, f-strings
- Do **not** use `from __future__ import annotations` — not needed on Python 3.12+ and scheduled for removal in a future Python version
- `pydantic` v2 for all models
- `async/await` throughout — all I/O is async
- `structlog` for structured logging
- Docstrings: Google style on all public functions and classes
- Max line length: 100 characters
- Imports: `isort` with Black-compatible profile

## Running Tests

```bash
# Install with dev + all language grammar extras (required for full test coverage)
uv pip install -e ".[graph-all,dev]"

# Full test suite
pytest tests/ -v --cov=src/codesteward

# Taint-source emission tests only
pytest tests/test_engine/test_taint_sources.py -v

# Tree-sitter parser tests only (requires [graph] extra)
pytest tests/test_engine/test_tree_sitter_parser.py -v

# MCP tool tests only
pytest tests/test_mcp/ -v
```

Tests require no external services.  Graph backends are mocked in all `test_mcp/` tests.

## Adding a New Language Parser

1. Create `src/codesteward/engine/parsers/{language}.py` subclassing `BaseParser`.
2. Implement `parse(source, file_path, language, tenant_id, repo_id) -> ParseResult`.
3. Implement `_extract_call_edges()` and (if applicable) `_extract_guarded_by_edges()` / `_extract_protected_by_edges()`.
4. Register the file extension → parser in `GraphBuilder._get_parser()`.
5. Add a `[graph-{language}]` optional dependency in `pyproject.toml` for the tree-sitter grammar.
6. Add tests in `tests/test_engine/`.

## Adding a New MCP Tool

1. Add a plain `async def tool_{name}(... cfg: McpConfig) -> str` in `tools/graph.py`.
2. Import and wrap it with `@mcp.tool()` in `server.py`.
3. Add tests in `tests/test_mcp/test_graph_tools.py`.

## Environment Variables

```bash
TRANSPORT=sse                     # sse (default) | http | stdio
HOST=0.0.0.0
PORT=3000
GRAPH_BACKEND=auto                # auto (default) | neo4j | janusgraph | graphqlite
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=                   # set to enable Neo4j; empty → auto-detect
JANUSGRAPH_URL=ws://localhost:8182/gremlin  # Gremlin Server WebSocket URL
GRAPHQLITE_DB_PATH=               # SQLite file path; defaults to ~/.codesteward/graph.db
DEFAULT_TENANT_ID=local
DEFAULT_REPO_ID=my-repo
DEFAULT_REPO_PATH=/repos/project   # server-side path; used when graph_rebuild omits repo_path
WORKSPACE_BASE=workspace
LOG_LEVEL=INFO
MCP_CONFIG_FILE=                  # optional path to YAML config
```
