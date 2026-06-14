# CodeGraph Rebuild — Specification Index

These specs are designed to be fed to opencode (or any AI coding tool) to rebuild CodeGraph in a fresh workspace. The specs contain everything needed — no need to reference the original source code.

## How to use

1. Start a new workspace (empty directory)
2. Initialize opencode with `opencode /init`
3. Feed the specs to opencode in dependency order:

```
Step 1: types + data model → SPEC.md (Section 3-4)
Step 2: database schema  → DB_SCHEMA.md
Step 3: extraction       → EXTRACTION_SPEC.md
Step 4: resolution       → RESOLUTION_SPEC.md
Step 5: graph traversal  → GRAPH_SPEC.md
Step 6: context builder  → CONTEXT_SPEC.md
Step 7: CLI + tools      → TOOLS_SPEC.md, INSTALLER_SPEC.md
```

## Spec files

| File | What it covers |
|------|---------------|
| `SPEC.md` | High-level architecture, data model (NodeKind, EdgeKind, interfaces), system pipeline |
| `DB_SCHEMA.md` | SQLite DDL (nodes, edges, files, FTS5, indexes, migrations) |
| `EXTRACTION_SPEC.md` | Tree-sitter extraction contract, grammar loading, per-language extractor spec, file scanning |
| `RESOLUTION_SPEC.md` | Reference resolution pipeline, import resolution, name matching, framework detection, callback synthesis |
| `GRAPH_SPEC.md` | Graph traversal (BFS/DFS, callers, callees, impact radius, type hierarchy, path finding) |
| `CONTEXT_SPEC.md` | Context building (FTS5 search, symbol extraction, traversal, formatter) |
| `TOOLS_SPEC.md` | opencode Custom Tools definitions (.opencode/tools/) |
| `INSTALLER_SPEC.md` | Multi-agent installer for opencode |

## Key design principles

These are critical — keep them in a `AGENTS.md` / rule file:

- **Deterministic**: All extraction is AST-derived, no LLM summarization
- **Local-first**: Per-project `.codegraph/` directory, SQLite database
- **Provenance tracking**: Every edge records how it was created (tree-sitter / heuristic)
- **Agent-centric**: Tool output must be sufficient that agent stops calling Read/Grep
- **Never use `isError` for expected conditions** — symbol not found = textResult with guidance
- **Partial coverage is worse than none** — every synthesized edge must complete the flow end-to-end
