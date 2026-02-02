# PRD: Polyglot LSP Module for crewkit CLI

## Executive Summary

Build a Language Server Protocol module within the crewkit CLI that provides:
- Fast code navigation across polyglot monorepos (symbols, definitions, references)
- Semantic search using Claude Haiku (not embeddings)
- Cross-project contract tracking (API endpoints, producers, consumers)
- Domain concept mapping (User model → User serializer → User interface)
- Framework-aware indexing using definitions from crewkit API
- Persistent storage with redb (pure Rust, no C deps)

**Integration**: Auto-starts with `crewkit code`, exposed via environment variable for Claude Code to consume.

## Architecture

### System Layers

1. **LSP Interface** (tower-lsp) - JSON-RPC communication
2. **Query Router** - Dispatches to handlers based on configurable rules
3. **Index Layer** - Symbol, Graph, and Chunk indexes
4. **Storage** - redb for persistence + in-memory text index

### Key Design Decisions

**Claude Haiku as Semantic Layer**
- Index-time: Generate chunk summaries, concepts, questions_answered
- Query-time: Rank candidates, expand context, explain code
- Cost: ~$0.15 for 10k chunks at index time, ~$0.001 per query

**redb for Storage**
- Pure Rust (no SQLite/C compilation)
- ACID transactions
- Memory-mapped for instant startup
- Single file at `.crewkit/index.redb`

**crewkit API Integration**
- Fetch curated framework definitions (Rails, Next.js, Laravel, etc.)
- Cache locally with TTL
- Continuous improvement via platform feedback loop
- No software updates needed for new framework knowledge

**Phased Startup**
- Phase 1: Immediate LSP availability (stdio/TCP)
- Phase 2: Load index from redb (instant with mmap)
- Phase 3: Incremental update check (hash comparison)
- Phase 4: Background re-indexing of changed files
- Phase 5: Configuration check/regeneration
- Phase 6: Framework definition fetch from API

## Supported Languages (Phase 1)

| Language | Tree-sitter Crate | Primary Use Case |
|----------|-------------------|------------------|
| Rust | tree-sitter-rust | CLI tooling |
| TypeScript/JavaScript | tree-sitter-typescript | Frontend dashboards, Node backends |
| Ruby | tree-sitter-ruby | Rails API backends |
| Python | tree-sitter-python | Scripts, ML workloads |
| Go | tree-sitter-go | Microservices, infrastructure |

## Storage Architecture

### Database Location

`.crewkit/index.redb` within project root

### Table Definitions

**files** - path → metadata (hash, mtime, language, project, index_timestamp)
**symbols_by_name** - name:kind → locations[]
**symbols_by_file** - path → symbols[]
**refs_by_name** - name → locations[]
**refs_by_file** - path → refs[]
**chunks** - chunk_id → chunk record (code, metadata, summaries)
**chunks_by_file** - path → chunk_ids[]
**concept_index** - concept → chunk_ids[]
**graph_edges** - source:relation → target_ids[]
**graph_reverse** - target → source:relation[]
**endpoints** - method:path → endpoint definition
**endpoint_consumers** - endpoint_id → consumer locations[]
**domain_concepts** - concept_name → mappings[]
**config** - Cache framework definitions and generated configs

### In-Memory Text Index

Built on startup from chunk summaries, concepts, and questions_answered fields. Enables fast candidate retrieval for semantic search without SQLite FTS5.

## LSP Interface

### Standard Methods

- `textDocument/definition` - Go to definition
- `textDocument/references` - Find references
- `textDocument/documentSymbol` - Document outline
- `textDocument/hover` - Documentation
- `workspace/symbol` - Symbol search
- `textDocument/publishDiagnostics` - Syntax errors

### Custom Methods

- `crewkit/semanticSearch` - Natural language code search
- `crewkit/getContext` - Assemble context bundle
- `crewkit/explain` - Explain code section
- `crewkit/findSimilar` - Find similar code
- `crewkit/queryGraph` - Graph traversal
- `crewkit/queryConcept` - Domain concept lookup
- `crewkit/queryEndpoint` - API contract lookup
- `crewkit/analyzeImpact` - Impact analysis

### Custom Commands

- `crewkit-lsp/regenerateConfig` - Force config regen
- `crewkit-lsp/showConfig` - Show merged config
- `crewkit-lsp/reindex` - Force full reindex
- `crewkit-lsp/stats` - Index statistics
- `crewkit-lsp/refreshDefinitions` - Refresh framework defs
- `crewkit-lsp/showFrameworks` - Show detected frameworks
- `crewkit-lsp/symbolInfo` - Framework symbol details

## Configuration System

### File Structure

- `.crewkit/lsp.auto.toml` - Auto-generated (Haiku)
- `.crewkit/lsp.toml` - User overrides
- `.crewkit/config.toml` - Global CLI settings (API keys, etc.)

### Configuration Contents

- Project definitions (monorepo structure, languages, frameworks)
- Subproject definitions (role, contract file patterns)
- Injection rules (framework symbols to inject per context)
- Routing rules (path/content patterns for dispatch)
- Domain concepts (cross-project entity mappings)
- Documentation paths (files to index)

### Merging Strategy

Built-in defaults → crewkit API definitions → auto-generated → user overrides

## crewkit API Integration

### Framework Definitions

Comprehensive injection rules, symbol catalogs, meta-patterns, tree-sitter queries, and pattern recognition rules for Rails, Next.js, Laravel, Express, Django, FastAPI, and more.

### API Endpoints

- `GET /api/v1/frameworks` - List supported frameworks
- `GET /api/v1/frameworks/:name/definitions` - Fetch definition set
- `GET /api/v1/frameworks/:name/definitions?version=X` - Version-specific

### Fetch Strategy

- On framework detection, fetch definitions from API
- Cache locally with TTL
- Serve from cache during normal operation
- Background refresh for stale definitions
- Offline fallback to cached or built-in defaults

### Authentication

Uses CLI's existing authentication (API key or OAuth token).

## Implementation Phases

### Phase 1: Module Scaffold ✅ COMPLETE

**Goal**: Basic LSP server that responds to initialize/shutdown

**Files created:**
- `cli/src/commands/lsp.rs` - Plugin management commands
- `cli/src/lsp/mod.rs` - Module root
- `cli/src/lsp/server.rs` - LSP server implementation
- `cli/src/lsp/backend.rs` - LSP backend handlers

**Dependencies added:**
- tower-lsp = "0.20"
- lsp-types = "0.95"

**Deliverables completed:**
- `crewkit lsp start --stdio` starts server in stdio mode
- `crewkit lsp start --port PORT` starts on TCP
- `crewkit lsp install --project` installs Claude Code plugin
- `crewkit lsp uninstall --project` removes plugin
- `crewkit lsp status --project` checks installation
- Responds to LSP initialize/shutdown
- Automatic plugin check on `crewkit code` startup
- Conflicting plugin detection and inline disable

### Phase 2: Storage + Symbol Index ✅ COMPLETE

**Goal**: Parse files, extract symbols, store in redb, basic navigation

**Files created:**
- `cli/src/lsp/storage/` (database.rs, tables.rs)
- `cli/src/lsp/index.rs` - IndexManager for symbol extraction
- `cli/src/lsp/languages/` (rust.rs, typescript.rs, ruby.rs)

**Dependencies added:**
- redb, bincode
- tree-sitter, tree-sitter-rust, tree-sitter-typescript, tree-sitter-ruby

**Deliverables completed:**
- Parse with tree-sitter (Rust, TypeScript, JavaScript, Ruby)
- Extract symbols (functions, classes, methods, modules, traits, interfaces)
- Store in redb with SWMR for concurrent access
- LSP methods: goToDefinition, documentSymbol, workspaceSymbol, findReferences
- Incremental update on file change (via didOpen/didChange)
- hover support with symbol information

### Phase 3: References + Graph Index (Week 3)

**Goal**: Find references, track relationships

**New files:**
- `cli/src/lsp/storage/graph.rs`
- `cli/src/lsp/index/graph_index.rs`

**Deliverables:**
- Extract references
- Extract relationships (imports, calls, inherits)
- LSP method: findReferences
- Custom method: queryGraph

### Phase 4: Chunking + Text Search (Week 4)

**Goal**: Semantic chunking, text-based search

**New files:**
- `cli/src/lsp/parser/chunker.rs`
- `cli/src/lsp/storage/chunks.rs`
- `cli/src/lsp/index/text_index.rs`

**Deliverables:**
- Chunk code into logical units
- Store structural metadata
- Build in-memory text index
- Basic semantic search (text-based, no Haiku)

### Phase 5: Claude Integration (Week 5)

**Goal**: Haiku-powered semantic features

**New files:**
- `cli/src/lsp/semantic/` (summarizer, ranker, config_gen)

**Deliverables:**
- Batch chunk summarization
- Query-time ranking with Haiku
- Custom methods: semanticSearch, getContext, explain

### Phase 6: Configuration System (Week 6)

**Goal**: Auto-config generation, framework detection

**New files:**
- `cli/src/lsp/router/` (config, loader, matcher, injection)

**Deliverables:**
- Detect project structure and frameworks
- Generate config via Haiku
- Load and merge configs
- Apply injection rules
- Hot reload

### Phase 7: crewkit API Integration (Week 7)

**Goal**: Fetch framework definitions from API

**New files:**
- `cli/src/lsp/router/definitions.rs`

**Deliverables:**
- Framework detection
- Fetch definitions from crewkit API
- Cache in redb
- Background refresh
- Custom commands for definition management

### Phase 8: Contracts + Domain Index (Week 8)

**Goal**: API contract tracking, domain concepts

**New files:**
- `cli/src/lsp/contracts/` (routes, consumers, domain)

**Deliverables:**
- Extract routes
- Detect API consumers
- Link endpoints to producers/consumers
- Domain concept mapping
- Custom methods: queryEndpoint, queryConcept, analyzeImpact

### Phase 9: Polish + Integration (Week 9)

**Goal**: Production-ready, Claude Code integration

**Deliverables:**
- Optimize performance
- Progress reporting
- Error handling
- Stats and debugging
- Integration with `crewkit code`
- Claude Code testing
- Documentation

## Claude Code Integration

### Plugin-Based Integration (Implemented)

The crewkit LSP integrates with Claude Code via the **plugin system**, not environment variables. This provides a cleaner, more reliable integration.

#### Plugin Installation

```bash
# Install plugin for current project
crewkit lsp install --project

# Check plugin status
crewkit lsp status --project

# Uninstall plugin
crewkit lsp uninstall --project
```

The plugin is installed to Claude Code's plugin system:
- Creates a local marketplace at `~/.claude/plugins/marketplaces/crewkit-local/`
- Registers the marketplace via `claude plugin marketplace add`
- Installs the plugin via `claude plugin install --scope project`

#### Plugin Configuration

The plugin configures Claude Code to spawn `crewkit lsp start --stdio` for code intelligence:

```json
{
  "lspServers": {
    "crewkit": {
      "command": "/path/to/crewkit",
      "args": ["lsp", "start", "--stdio"],
      "extensionToLanguage": {
        ".rs": "rust",
        ".ts": "typescript",
        ".tsx": "typescriptreact",
        ".js": "javascript",
        ".jsx": "javascriptreact",
        ".rb": "ruby"
      }
    }
  }
}
```

#### Automatic Startup Check

When running `crewkit code`, the CLI automatically:

1. **Checks plugin status** - Is crewkit-lsp installed and enabled?
2. **Detects conflicts** - Are rust-analyzer-lsp or typescript-lsp enabled?
3. **Prompts for action** - Offers to install plugin or disable conflicts

```
ℹ Enhance Claude Code with code intelligence
  · Jump to definitions and find references
  · Navigate codebases faster
Install crewkit-lsp? [Y/n]
```

#### Conflicting Plugin Handling

If conflicting LSP plugins are detected (e.g., rust-analyzer-lsp), the CLI offers to disable them:

```
⚠ Conflicting LSP plugins detected:
  · rust-analyzer-lsp@claude-plugins-official
Disable conflicting plugins? [Y/n]
```

#### Multiple Instance Support

When multiple Claude Code sessions run in the same project:
- Each session spawns its own `crewkit lsp start --stdio` process
- The shared index uses redb's SWMR (Single-Writer-Multiple-Reader) mode
- Index is stored at `.crewkit/index.redb`

### Failsafe Mechanisms

1. **Stdio transport**: Plugin uses stdio (not TCP) for reliability
2. **Graceful degradation**: If LSP unavailable, Claude Code still works
3. **One-time prompt**: Users are only prompted once for installation
4. **Retry on failure**: If installation fails, prompts again on next run

### Legacy Support

For environments where plugins aren't available, the TCP + .lsp.json approach is still supported:

```json
{
  "crewkit": {
    "command": "nc",
    "args": ["127.0.0.1", "9257"],
    "extensionToLanguage": {
      ".rs": "rust",
      ".ts": "typescript",
      ".tsx": "typescriptreact",
      ".js": "javascript",
      ".jsx": "javascriptreact",
      ".rb": "ruby"
    }
  }
}
```

## Success Criteria

### Performance

- Warm startup < 1s
- Go-to-definition < 50ms
- Semantic search < 2s (including Haiku)
- Incremental re-index < 200ms per file

### Functionality

- All Phase 1 languages supported (Rust, TS/JS, Ruby, Python, Go)
- Cross-project contract tracking (Rails + Next.js + Rust CLI)
- Domain concept linking works
- Semantic search returns relevant results
- Zero-config for Claude Code integration

### Quality

- No crashes on malformed input
- Graceful degradation when Claude/crewkit API unavailable
- Clear error messages
- Index recovery via `crewkit lsp --reindex`

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| tree-sitter parsing edge cases | Fallback to text-based search on parse errors |
| Performance on large repos | Lazy indexing, LRU eviction, incremental updates |
| nc dependency for TCP-to-stdio | Bundle Rust TCP wrapper if needed |
| Haiku rate limits | Batching, caching, backoff |
| Framework definition schema changes | Versioning, backward compatibility |
| LSP startup failures | Health checks, automatic retry, fallback modes |
| Port conflicts | Ephemeral port allocation, retry logic |

## Dependencies

### New Crates

- tower-lsp, lsp-types (LSP protocol)
- redb (storage)
- bincode (serialization)
- tree-sitter, tree-sitter-{rust,typescript,ruby,python,go} (parsing)
- dashmap (concurrent caching)
- globset (path matching)

### Existing Crates (Reuse)

- tokio (async runtime)
- serde, serde_json (serialization)
- toml (config parsing)
- reqwest (crewkit API client)
- tracing (logging)
- anyhow, thiserror (errors)
- notify (file watching)

## Future Enhancements

- Shared index caching for common dependencies
- Multi-root workspace support
- Language server federation (delegate to rust-analyzer when available)
- Streaming responses for large context bundles
- Collaborative features (shared index across team)