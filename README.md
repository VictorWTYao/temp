# CodeGraph Rebuild Guideline

## How to use this guide

Give these prompts to opencode in **sequential order**. Do not skip ahead — each session builds on the previous one.

**Setup:** Start with an empty directory, then run `opencode /init` in it.

**After each session:** Run `npx tsc --noEmit` to confirm the code compiles before moving to the next session.

---

## Session 1: Types, Data Model, Project Structure

**Read:** `docs/rebuild/SPEC.md`

**Prompt for opencode:**

> Read the file docs/rebuild/SPEC.md. Implement everything it specifies:
>
> 1. Initialize a TypeScript project with `npm init -y` and `npm install typescript @types/node --save-dev`
> 2. Create a `tsconfig.json` targeting ES2022, strict mode, moduleResolution node
> 3. Create the full directory structure under `src/` as specified in the Project Structure section
> 4. Implement ALL types and interfaces from the Data Model section (Node, Edge, FileRecord, Subgraph, TraversalOptions, etc.) in `src/types.ts`
> 5. Implement the NodeKind enum (22 values) and EdgeKind enum (12 values) exactly as specified
> 6. Implement the `Language` type union with all 30+ supported languages
> 7. Create the `CodeGraph` class shell in `src/index.ts` with all method signatures from the Public API section
> 8. Create `src/errors.ts` with the error hierarchy (CodeGraphError, IndexError, etc.)
> 9. Create `src/utils.ts` with Mutex, FileLock, debounce helpers
> 10. Create `src/directory.ts` with .codegraph directory management
>
> After implementing, run `npx tsc --noEmit` and fix any type errors. Do not move to Session 2 until this compiles.

---

## Session 2: Database Schema

**Read:** `docs/rebuild/DB_SCHEMA.md`

**Prompt for opencode:**

> Read the file docs/rebuild/DB_SCHEMA.md. Implement everything it specifies:
>
> 1. Install `better-sqlite3` and `@types/better-sqlite3` as optional peer dependencies
> 2. Create `src/db/schema.sql` with the exact DDL from the spec (nodes, edges, files, unresolved_refs, project_metadata tables, FTS5 virtual table, all indexes, all triggers)
> 3. Create `src/db/index.ts` with DatabaseConnection — auto-select native vs WASM backend, manage WAL mode, run migrations
> 4. Create `src/db/queries.ts` with QueryBuilder — all prepared statements listed in the QueryBuilder API section
> 5. Create `src/db/migrations.ts` — versioned migration runner with schema_versions table
>
> After implementing, run `npx tsc --noEmit` and fix any type errors.

---

## Session 3: Extraction Pipeline

**Read:** `docs/rebuild/EXTRACTION_SPEC.md`

**Prompt for opencode:**

> Read the file docs/rebuild/EXTRACTION_SPEC.md. Implement everything it specifies:
>
> 1. Install `web-tree-sitter` and `tree-sitter-wasms` as dependencies
> 2. Create `src/extraction/grammars.ts` — grammar loading, caching, parser management, file extension mapping, .h disambiguation, resetParser
> 3. Create `src/extraction/tree-sitter.ts` — the tree-sitter wrapper with extractFromSource that applies tree-sitter query patterns
> 4. Create `src/extraction/languages/` — at minimum implement extractors for: typescript, javascript, python, go, rust. Each extracts classes, functions, methods, interfaces, imports, exports, calls, extends, implements using tree-sitter queries
> 5. Create `src/extraction/parse-worker.ts` — the off-thread parsing worker with timeout, recycling, error handling
> 6. Create `src/extraction/index.ts` — ExtractionOrchestrator with indexAll, indexFiles, sync
> 7. Create custom extractors in `src/extraction/`: svelte-extractor.ts, vue-extractor.ts, astro-extractor.ts, liquid-extractor.ts, razor-extractor.ts (stub implementations — they delegate script blocks to TS/JS extractor)
> 8. Create `src/extraction/wasm/` directory (empty for now)
>
> After implementing, run `npx tsc --noEmit` and fix any type errors.

---

## Session 4: Reference Resolution

**Read:** `docs/rebuild/RESOLUTION_SPEC.md`

**Prompt for opencode:**

> Read the file docs/rebuild/RESOLUTION_SPEC.md. Implement everything it specifies:
>
> 1. Create `src/resolution/types.ts` — UnresolvedRef, ResolvedRef, ResolutionResult, ResolutionContext, ImportMapping, ReExport, FrameworkResolver interfaces
> 2. Create `src/resolution/lru-cache.ts` — LRU cache with bounded size
> 3. Create `src/resolution/import-resolver.ts` — resolveViaImport, resolveJvmImport, extractImportMappings, extractReExports, isPhpIncludePathRef
> 4. Create `src/resolution/name-matcher.ts` — matchReference, matchFunctionRef, matchDottedCallChain, matchScopedCallChain, sameLanguageFamily, crossesKnownFamily
> 5. Create `src/resolution/path-aliases.ts` — loadProjectAliases from tsconfig/jsconfig paths, cargo workspace member globs
> 6. Create `src/resolution/go-module.ts` — loadGoModule from go.mod
> 7. Create `src/resolution/workspace-packages.ts` — workspace member package detection
> 8. Create `src/resolution/callback-synthesizer.ts` — Phase 1 field-backed observer, Phase 2 EventEmitter, React re-render, JSX child synthesis, closure collection dispatch. All edges get provenance: 'heuristic'
> 9. Create `src/resolution/frameworks/index.ts` — framework resolver registry with detectFrameworks function. At minimum implement: expressResolver, reactResolver
> 10. Create `src/resolution/index.ts` — ReferenceResolver with the full 6-strategy resolve pipeline, deferred chain resolution, conformance pass, batched resolveAndPersistBatched
>
> Implement built-in/external symbol filters for: JS/TS, Python, Go, Pascal, C/C++
> Implement language gates for import/name-match strategies
> Implement edge creation with promotion rules (extends→implements, calls→instantiates)
>
> After implementing, run `npx tsc --noEmit` and fix any type errors.

---

## Session 5: Graph Traversal

**Read:** `docs/rebuild/GRAPH_SPEC.md`

**Prompt for opencode:**

> Read the file docs/rebuild/GRAPH_SPEC.md. Implement everything it specifies:
>
> 1. Create `src/graph/traversal.ts` — GraphTraverser with:
>    - traverseBFS (edge priority: contains > calls > others)
>    - traverseDFS
>    - getCallers / getCallees (recursive, batch-fetched, handles instantiates)
>    - getCallGraph
>    - getTypeHierarchy (ancestors + descendants via extends/implements)
>    - findUsages
>    - getImpactRadius (traverses into container children, excludes contains edges)
>    - findPath (BFS shortest path)
>    - getAncestors / getChildren (containment hierarchy)
>
> 2. Create `src/graph/index.ts` — GraphQueryManager with:
>    - getContext (focal node + ancestors + children + refs + types + imports)
>    - getFileDependencies / getFileDependents
>    - findCircularDependencies (Tarjan's SCC)
>    - findDeadCode
>    - getNodeMetrics
>
> Use batch-fetching (getNodesByIds) instead of N+1 getNodeById calls throughout.
>
> After implementing, run `npx tsc --noEmit` and fix any type errors.

---

## Session 6: Context Builder

**Read:** `docs/rebuild/CONTEXT_SPEC.md`

**Prompt for opencode:**

> Read the file docs/rebuild/CONTEXT_SPEC.md. Implement everything it specifies:
>
> 1. Install `ignore` and `picomatch` as dependencies
> 2. Create `src/search/query-utils.ts` — extractSearchTerms, isCommonWord, buildFTSQuery, extractSymbolsFromQuery, searchNodes
> 3. Create a `src/context/formatter.ts` — formatContextAsMarkdown, formatContextAsJson
> 4. Create `src/context/markers.ts` — LOW_CONFIDENCE_MARKER constant
> 5. Create `src/context/index.ts` — ContextBuilder with:
>    - getCode (read source lines from disk)
>    - findRelevantContext (FTS5 search + graph expansion)
>    - buildContext (search → expand → extract code → format)
>
> After implementing, run `npx tsc --noEmit` and fix any type errors.

---

## Session 7: opencode Custom Tools

**Read:** `docs/rebuild/TOOLS_SPEC.md`

**Prompt for opencode:**

> Read the file docs/rebuild/TOOLS_SPEC.md. Implement everything it specifies:
>
> 1. Install `@opencode-ai/plugin` as a dependency
> 2. Create `.opencode/tools/explore.ts` — the PRIMARY tool that takes a bag of symbol names, finds flow paths, returns inline source with adaptive budget
> 3. Create `.opencode/tools/node.ts` — SECONDARY depth tool, returns full body + caller/callee trail, ALL overloads for ambiguous names
> 4. Create `.opencode/tools/search.ts` — FTS5 search across indexed symbols
> 5. Create `.opencode/tools/status.ts` — index state, backend, stale flags
> 6. Create `.opencode/tools/callers.ts` — caller chain analysis
> 7. Create `.opencode/tools/callees.ts` — callee chain analysis
> 8. Create `.opencode/tools/impact.ts` — impact radius analysis
>
> Each tool should import CodeGraph from the project and use the invokeCodeGraph helper (or direct import) pattern described in the spec.
>
> Implement the explore budget scaling functions and the flow builder.
> Follow the error handling rules: never use isError for expected conditions.
>
> After implementing, run `npx tsc --noEmit` and fix any type errors.

---

## Session 8: Installer CLI

**Read:** `docs/rebuild/INSTALLER_SPEC.md`

**Prompt for opencode:**

> Read the file docs/rebuild/INSTALLER_SPEC.md. Implement everything it specifies:
>
> 1. Install `commander` as a dependency
> 2. Create `src/installer/tools/` directory and create copies of all 7 tool files (explore.ts, node.ts, search.ts, status.ts, callers.ts, callees.ts, impact.ts)
> 3. Create `src/bin/codegraph.ts` — CLI entry point with commander subcommands:
>    - `install` — creates .opencode/tools/, copies tool files, runs codegraph init
>    - `uninstall` — removes tool files and .codegraph/ directory
>    - `index` — triggers full index
>    - `sync` — incremental sync
>    - `status` — print index status
>    - `init` / `uninit` — initialize/remove project
> 4. Create `src/installer/index.ts` — installer orchestrator with install/uninstall logic
> 5. Wire up the `package.json` bin field to point to the compiled CLI entry point
>
> The install command should be idempotent — re-running updates changed files but doesn't re-index.
> The tool files in src/installer/tools/ are the canonical copies that get deployed.
>
> After implementing, run `npx tsc --noEmit` and fix any type errors.

---

## Verification Checklist

After all 8 sessions:

```bash
npm run build          # tsc compiles cleanly
npx codegraph init     # initializes a test project
npx codegraph index    # indexes the codebase
npx codegraph status   # shows indexed state

# Test with opencode
codegraph_explore symbols=["CodeGraph", "indexAll", "searchNodes"]
codegraph_status
```

If any step fails, check the relevant spec file and re-read the corresponding session's implementation.
