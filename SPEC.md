# CodeGraph System Specification

## 1. Elevator Pitch

A local-first code intelligence tool that parses any codebase with tree-sitter, extracts a symbolic knowledge graph (nodes + edges), stores it in SQLite with FTS5 full-text search, and exposes the graph to AI agents via opencode Custom Tools. Deterministic, local-only, real-time file watching.

## 2. Architecture Overview

```
Filesystem → tree-sitter WASM (extraction) → SQLite DB (nodes/edges/files with FTS5)
                                          → Resolution (imports + name-matching + framework patterns)
                                          → Graph traversal (BFS/DFS, callers/callees, impact)
                                          → Context builder (search + traversal + code blocks)
                                          → opencode Custom Tools (.opencode/tools/)
```

### Layers

1. **Extraction Orchestrator** — scans files, detects languages, runs tree-sitter WASM parsing off-thread, produces nodes/edges/unresolved references
2. **Reference Resolver** — post-extraction pass: resolves imports, matches names, runs framework detectors, synthesizes dynamic dispatch edges
3. **Graph Traversal** — BFS/DFS traversals over the node-edge graph (callers, callees, impact radius, type hierarchy, path finding)
4. **Context Builder** — combines FTS5 search with graph traversal to produce markdown context for AI agents
5. **opencode Custom Tools** — wraps the above as `codegraph_explore`, `codegraph_node`, `codegraph_search`, `codegraph_status`

## 3. Data Model

### NodeKind (22 kinds — MUST match this exact list)

```
file, module, class, struct, interface, trait, protocol,
function, method, property, field, variable, constant,
enum, enum_member, type_alias, namespace, parameter,
import, export, route, component
```

### EdgeKind (12 kinds)

| Kind | Description |
|------|-------------|
| contains | Parent contains child (file→class, class→method) |
| calls | Function/method calls another |
| imports | File imports from another |
| exports | File exports a symbol |
| extends | Class/interface extends another |
| implements | Class implements interface |
| references | Generic reference to another symbol |
| type_of | Variable/parameter has type |
| returns | Function returns type |
| instantiates | Creates instance of class |
| overrides | Method overrides parent method |
| decorates | Decorator applied to symbol |

### Node Interface

```typescript
interface Node {
  id: string;                // hash(filePath + qualifiedName)
  kind: NodeKind;
  name: string;              // simple name (e.g. "calculateTotal")
  qualifiedName: string;     // e.g. "src/utils.ts::MathHelper.calculateTotal"
  filePath: string;          // relative to project root
  language: Language;
  startLine: number;         // 1-indexed
  endLine: number;           // 1-indexed
  startColumn: number;       // 0-indexed
  endColumn: number;
  docstring?: string;
  signature?: string;
  visibility?: 'public' | 'private' | 'protected' | 'internal';
  isExported?: boolean;
  isAsync?: boolean;
  isStatic?: boolean;
  isAbstract?: boolean;
  decorators?: string[];
  typeParameters?: string[];
  returnType?: string;       // normalized return type (for C++ receiver inference)
  updatedAt: number;
}
```

### Edge Interface

```typescript
interface Edge {
  source: string;            // source node ID
  target: string;            // target node ID
  kind: EdgeKind;
  metadata?: Record<string, unknown>;  // e.g. { confidence, resolvedBy }
  line?: number;
  column?: number;
  provenance?: 'tree-sitter' | 'scip' | 'heuristic';
}
```

### FileRecord Interface

```typescript
interface FileRecord {
  path: string;
  contentHash: string;       // SHA256
  language: Language;
  size: number;
  modifiedAt: number;
  indexedAt: number;
  nodeCount: number;
  errors?: ExtractionError[];
}
```

### Provenance rules

- `tree-sitter` — statically extracted from AST (ground truth)
- `heuristic` — synthesized by resolver (callback synthesis, framework patterns)
- `scip` — imported from SCIP index (future use)

## 4. Supported Languages (30)

Extraction via tree-sitter WASM grammars. Languages with custom extractors (no tree-sitter grammar) use regex/parsing:

### Languages with tree-sitter grammars (from `tree-sitter-wasms` package)

typescript, tsx, javascript, jsx, python, go, rust, java, c, cpp, csharp, php, ruby, swift, kotlin, dart, pascal, scala, lua, luau, objc, r

### Vendored WASM grammars (bundled locally, not from tree-sitter-wasms)

pascal, scala, lua, luau, csharp, r

### Custom extractor languages (no tree-sitter grammar — regex/parse)

svelte, vue, astro, liquid, razor

### File-level-only languages (tracked but no symbol extraction)

yaml, twig, xml, properties

### .h file detection

`.h` files default to C. Source content is checked:
- C++ keywords (namespace, class, template, virtual, etc.) → cpp
- Objective-C constructs (@interface, @implementation, @protocol) → objc

## 5. Framework Resolvers (22)

Each framework has a `detect(context)` → boolean and `resolve(ref, context)` → ResolvedRef method.

| Framework | File | Language | What it detects/resolves |
|-----------|------|----------|--------------------------|
| Laravel | `laravel.ts` | PHP | Routes, controllers |
| Drupal | `drupal.ts` | PHP | Routing YAML → PHP controllers |
| Express | `express.ts` | JS/TS | Router.get/post → route nodes |
| NestJS | `nestjs.ts` | JS/TS | @Controller/@Module decorators → route nodes |
| React | `react.ts` | JS/TS | Components, hooks |
| Svelte | `svelte.ts` | Svelte | Components, routes |
| Vue | `vue.ts` | Vue | Components, event handlers |
| Astro | `astro.ts` | Astro | Components, routes |
| Django | `python.ts` | Python | URL patterns → views |
| Flask | `python.ts` | Python | Route decorators → handlers |
| FastAPI | `python.ts` | Python | Route decorators → handlers |
| Rails | `ruby.ts` | Ruby | Routes → controllers |
| Spring | `java.ts` | Java | @RequestMapping → handlers |
| Play | `play.ts` | Scala/Java | conf/routes → controllers |
| Gin | `go.ts` | Go | router.GET → handlers |
| Axum | `rust.ts` | Rust | Router::route → handlers |
| ASP.NET | `csharp.ts` | C# | [Route] → controllers |
| Vapor | `swift.ts` | Swift | router.get → handlers |
| SwiftUI | `swift.ts` | Swift | View protocols |
| UIKit | `swift.ts` | Swift | @objc, storyboard references |
| React Native | `react-native.ts` | TS/ObjC | Bridge → native |
| Cargo workspace | `cargo-workspace.ts` | Rust | Workspace members, path aliases |

Each resolver `detect()` should return true only when the framework is actually present (e.g., detect `require('express')` or `from flask import`). Detection runs during `indexAll` and before resolution, using the scanned file list.

## 6. Key Design Principles

### Dynamic dispatch synthesis

Static tree-sitter extraction misses computed/indirect calls. The system bridges these dynamically:

- **Callback/observer**: `onX(cb)` registrations paired with `triggerX()` dispatchers
- **EventEmitter**: `this.on('event', handler)` + `this.emit('event')` → synthesized edge from emit to handler
- **React re-render**: `setState()` → `render()` method → JSX child component
- **Django ORM**: descriptor access → model field references

All synthesized edges get `provenance: 'heuristic'` with `metadata.synthesizedBy` and `metadata.registeredAt`.

### Agent-centric tool design

- `codegraph_explore` is the PRIMARY tool — accepts a bag of symbol names, returns flow path among them with inline source
- `codegraph_node` is SECONDARY depth tool — returns full body + caller/callee trail for a symbol
- Never use `isError` for expected conditions (symbol not found = textResult with guidance)
- Unindexed workspace → empty tool list + inactive message

### Explore budget scaling

| Repo size (files) | Explore calls | Output chars |
|---|---|---|
| <500 | 1 | 18K |
| <5000 | 2 | 28K |
| <15000 | 3 | 35K |
| <25000 | 4 | 38K |
| >=25000 | 5 | 38K |

Per-file allocation must be monotonic with repo size (never less for larger repos).

## 7. CodeGraph Class — Public API

```typescript
class CodeGraph {
  // Lifecycle
  static async init(projectRoot, options?): CodeGraph
  static async open(projectRoot, options?): CodeGraph
  close(): void
  uninitialize(): void

  // Indexing
  async indexAll(options?): IndexResult
  async indexFiles(filePaths): IndexResult
  async sync(options?): SyncResult

  // File watching
  watch(options?): boolean
  unwatch(): void
  isWatching(): boolean
  getPendingFiles(): PendingFile[]
  getChangedFiles(): { added, modified, removed }

  // Graph queries
  getNode(id): Node | null
  getNodesInFile(filePath): Node[]
  getNodesByKind(kind): Node[]
  getNodesByName(name): Node[]
  searchNodes(query, options?): SearchResult[]
  getOutgoingEdges(nodeId): Edge[]
  getIncomingEdges(nodeId): Edge[]

  // Traversal
  getContext(nodeId): Context
  traverse(startId, options?): Subgraph
  getCallGraph(nodeId, depth?): Subgraph
  getTypeHierarchy(nodeId): Subgraph
  getCallers(nodeId, maxDepth?): Array<{node, edge}>
  getCallees(nodeId, maxDepth?): Array<{node, edge}>
  getImpactRadius(nodeId, maxDepth?): Subgraph
  findPath(fromId, toId, edgeKinds?): Array<{node, edge}> | null
  getAncestors(nodeId): Node[]
  getChildren(nodeId): Node[]
  findUsages(nodeId): Array<{node, edge}>
  findDeadCode(kinds?): Node[]

  // File dependencies
  getFileDependencies(filePath): string[]
  getFileDependents(filePath): string[]
  findCircularDependencies(): string[][]

  // Context
  async getCode(nodeId): string | null
  async findRelevantContext(query, options?): Subgraph
  async buildContext(input, options?): TaskContext | string

  // Stats
  getStats(): GraphStats
  optimize(): void
  clear(): void
}
```

## 8. Project Structure

```
src/
├── index.ts           # CodeGraph class (re-export types)
├── types.ts           # All type definitions
├── directory.ts       # .codegraph directory management
├── errors.ts          # Error hierarchy
├── utils.ts           # Mutex, FileLock, debounce, etc.
├── bin/
│   └── codegraph.ts   # CLI entry point
├── db/
│   ├── index.ts       # DatabaseConnection (native/WASM adapter)
│   ├── queries.ts     # QueryBuilder — all SQL prepared statements
│   ├── schema.sql     # DDL
│   └── migrations.ts  # Schema migrations
├── extraction/
│   ├── index.ts       # ExtractionOrchestrator
│   ├── grammars.ts    # Grammar loading + caching
│   ├── tree-sitter.ts # Tree-sitter wrapper + extractFromSource
│   ├── parse-worker.ts # Off-thread parsing worker
│   ├── languages/     # Per-language extractors (20)
│   ├── wasm/          # Vendored WASM grammars
│   └── *-extractor.ts # Custom extractors (Svelte, Vue, Astro, Liquid, Razor)
├── resolution/
│   ├── index.ts       # ReferenceResolver
│   ├── types.ts       # Resolution types
│   ├── import-resolver.ts  # Import resolution
│   ├── name-matcher.ts     # Symbol name matching
│   ├── callback-synthesizer.ts  # Dynamic dispatch synthesis
│   ├── path-aliases.ts     # tsconfig/Cargo aliases
│   ├── go-module.ts        # Go module resolution
│   ├── workspace-packages.ts # Workspace detection
│   ├── lru-cache.ts        # LRU cache for resolution
│   └── frameworks/   # 22 framework detectors
├── graph/
│   ├── index.ts       # GraphQueryManager
│   ├── queries.ts     # High-level graph queries
│   └── traversal.ts   # GraphTraverser
├── context/
│   └── index.ts       # ContextBuilder + formatter
├── search/
│   └── query-utils.ts # FTS5 query parsing
├── sync/
│   └── index.ts       # FileWatcher
├── installer/
│   ├── index.ts       # Installer orchestrator
│   └── targets/       # Agent target configs
└── ui/                # Terminal UI (spinner, progress bar)
```
