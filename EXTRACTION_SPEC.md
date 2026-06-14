# CodeGraph Extraction Specification

## 1. Grammar Loading

### Dependencies

- `web-tree-sitter` — WASM-based tree-sitter runtime
- `tree-sitter-wasms` — prebuilt WASM grammars for most languages

### Initialization

```typescript
// Called once at startup, idempotent
async function initGrammars(): Promise<void> {
  if (initialized) return;
  await Parser.init();  // web-tree-sitter init
  initialized = true;
}
```

### Grammar loading (lazy, per-language)

```typescript
const WASM_GRAMMAR_FILES = {
  typescript: 'tree-sitter-typescript.wasm',
  tsx: 'tree-sitter-tsx.wasm',
  javascript: 'tree-sitter-javascript.wasm',
  jsx: 'tree-sitter-javascript.wasm',
  python: 'tree-sitter-python.wasm',
  go: 'tree-sitter-go.wasm',
  rust: 'tree-sitter-rust.wasm',
  java: 'tree-sitter-java.wasm',
  c: 'tree-sitter-c.wasm',
  cpp: 'tree-sitter-cpp.wasm',
  csharp: 'tree-sitter-c_sharp.wasm',
  php: 'tree-sitter-php.wasm',
  ruby: 'tree-sitter-ruby.wasm',
  swift: 'tree-sitter-swift.wasm',
  kotlin: 'tree-sitter-kotlin.wasm',
  dart: 'tree-sitter-dart.wasm',
  pascal: 'tree-sitter-pascal.wasm',
  scala: 'tree-sitter-scala.wasm',
  lua: 'tree-sitter-lua.wasm',
  r: 'tree-sitter-r.wasm',
  luau: 'tree-sitter-luau.wasm',
  objc: 'tree-sitter-objc.wasm',
};

async function loadGrammarsForLanguages(languages: Language[]): Promise<void> {
  // 1. If any SFC language (svelte/vue/astro) is in the set, also add typescript + javascript
  // 2. Deduplicate, filter to those with WASM files, not already cached
  // 3. Load each WASM grammar sequentially (Node 20+ race condition on parallel loads)
  // 4. Cache the loaded Language object for reuse
}
```

### WASM path resolution

- Most grammars resolve from `tree-sitter-wasms/out/{filename}`
- Some languages (pascal, scala, lua, luau, csharp, r) use **vendored WASM files** from a local `wasm/` directory because the tree-sitter-wasms build is too old or buggy. Load from `__dirname + '/wasm/' + filename`.

### Parser cache

```typescript
const parserCache = new Map<Language, Parser>();

function getParser(language: Language): Parser | null {
  if (parserCache.has(language)) return parserCache.get(language);
  const lang = languageCache.get(language);
  if (!lang) return null;
  const parser = new Parser();
  parser.setLanguage(lang);
  parserCache.set(language, parser);
  return parser;
}

// Reclaim WASM heap memory by destroying + recreating parser
// WASM linear memory grows but NEVER shrinks
function resetParser(language: Language): void {
  const old = parserCache.get(language);
  if (old) { old.delete(); parserCache.delete(language); }
}
```

## 2. File Extension → Language Mapping

```typescript
const EXTENSION_MAP = {
  '.ts': 'typescript', '.tsx': 'tsx',
  '.mts': 'typescript', '.cts': 'typescript',
  '.js': 'javascript', '.mjs': 'javascript', '.cjs': 'javascript',
  '.xsjs': 'javascript', '.xsjslib': 'javascript',
  '.jsx': 'jsx',
  '.py': 'python', '.pyw': 'python',
  '.go': 'go',
  '.rs': 'rust',
  '.java': 'java',
  '.c': 'c', '.h': 'c',
  '.cpp': 'cpp', '.cc': 'cpp', '.cxx': 'cpp', '.hpp': 'cpp', '.hxx': 'cpp',
  '.cs': 'csharp',
  '.cshtml': 'razor', '.razor': 'razor',
  '.php': 'php',
  '.module': 'php', '.install': 'php', '.theme': 'php', '.inc': 'php',
  '.yml': 'yaml', '.yaml': 'yaml',
  '.twig': 'twig',
  '.rb': 'ruby', '.rake': 'ruby',
  '.swift': 'swift',
  '.kt': 'kotlin', '.kts': 'kotlin',
  '.dart': 'dart',
  '.liquid': 'liquid',
  '.svelte': 'svelte',
  '.vue': 'vue',
  '.astro': 'astro',
  '.r': 'r',
  '.pas': 'pascal', '.dpr': 'pascal', '.dpk': 'pascal',
  '.lpr': 'pascal', '.dfm': 'pascal', '.fmx': 'pascal',
  '.scala': 'scala', '.sc': 'scala',
  '.lua': 'lua', '.luau': 'luau',
  '.m': 'objc', '.mm': 'objc',
  '.xml': 'xml',
  '.properties': 'properties',
};
```

### .h file disambiguation

`.h` files default to `c`. When source is available, check content:

- **C++** patterns: `\bnamespace\b`, `\bclass\s+\w+\s*[:{]`, `\btemplate\s*<`, `(public|private|protected)\s*:`, `\bvirtual\b`, `\busing\s+(namespace\b|\w+\s*=)`
- **Objective-C** patterns: `@(interface|implementation|protocol|synthesize)\b`

### Shopify Liquid JSON

Files matching `(^|\/)(templates|sections)\/.+\.json$` are treated as `liquid` (the Liquid extractor links section type references to their `.liquid` files).

### Play Framework routes

The extensionless `conf/routes` (and `*.routes` includes) are treated as `yaml` (file-level only, route extraction by the Play framework resolver).

## 3. Extractors

### Per-language extractors

Each export a `LanguageExtractor` object:

```typescript
interface LanguageExtractor {
  language: Language;
  extract(source: string, config?: ExtractorConfig): ExtractionResult;
}
```

Languages with tree-sitter extractors (20): typescript, tsx, javascript, jsx, python, go, rust, java, c, cpp, csharp, php, ruby, swift, kotlin, dart, pascal, scala, lua, luau, objc, r

### Custom extractors (no tree-sitter grammar)

These parse files using regex or custom parsers:

1. **SvelteExtractor**: Extracts `<script>` blocks → delegates to TS/JS extractor. Detects component exports, store subscriptions.
2. **VueExtractor**: Extracts `<script>` blocks from SFC → delegates to TS/JS extractor. Detects component options.
3. **AstroExtractor**: Extracts frontmatter (`---`) and `<script>` blocks → delegates. Detects Astro components.
4. **LiquidExtractor**: Shopify Liquid templates — regex-based extraction of sections, snippets, and their reference links.
5. **RazorExtractor**: ASP.NET Razor/Blazor — extracts `@model`, `@inject`, `@using`, component tags, event handler bindings.
6. **MyBatisExtractor**: XML mapper files — extracts `<mapper namespace="...">` → produces nodes for SQL statements.
7. **DFMExtractor**: Delphi DFM form files — extracts component class references.

### ExtractionResult format

```typescript
interface ExtractionResult {
  nodes: Node[];
  edges: Edge[];
  unresolvedReferences: UnresolvedReference[];
  errors: ExtractionError[];
  durationMs: number;
}
```

### Node ID generation

```typescript
function generateNodeId(filePath: string, qualifiedName: string): string {
  return crypto.createHash('sha256')
    .update(`${filePath}:${qualifiedName}`)
    .digest('hex');
}
```

### Qualified name format

```
<filePath>::<parent1>.<parent2>.<name>
```

Examples:
- `src/utils.ts::calculateTotal`
- `src/models.ts::UserService.createUser`
- `src/components.ts::App.Header.render`

## 4. Tree-sitter Query Architecture

Each language extractor defines tree-sitter queries as nested capture patterns. The general approach:

### Common capture names across all languages

| Capture | Maps to |
|---------|---------|
| `@class.definition` | NodeKind: class |
| `@function.definition` | NodeKind: function |
| `@method.definition` | NodeKind: method (qualified under class) |
| `@interface.definition` | NodeKind: interface |
| `@struct.definition` | NodeKind: struct |
| `@enum.definition` | NodeKind: enum |
| `@enum_member` | NodeKind: enum_member |
| `@variable` | NodeKind: variable |
| `@property` | NodeKind: property |
| `@parameter` | NodeKind: parameter |
| `@type` | EdgeKind: type_of |
| `@call` | EdgeKind: calls (unresolved initially) |
| `@import` | EdgeKind: imports |
| `@export` | EdgeKind: exports |
| `@extends` | EdgeKind: extends |
| `@implements` | EdgeKind: implements |
| `@reference` | EdgeKind: references |
| `@decorator` | EdgeKind: decorates |
| `@doc` | Extracted as docstring |
| `@comment` | Extracted as docstring |

### Extractor pattern (TypeScript example)

```typescript
const typescriptExtractor: LanguageExtractor = {
  language: 'typescript',
  extract(source, config) {
    const parser = getParser('typescript');
    const tree = parser.parse(source);
    const root = tree.rootNode;
    
    // Apply query patterns:
    // 1. Class declarations → class node
    // 2. Method definitions → method nodes (under class)
    // 3. Function declarations → function nodes
    // 4. Variable declarations → variable/const nodes
    // 5. Interface declarations → interface nodes
    // 6. Type alias → type_alias nodes
    // 7. Import/export → import/export nodes + edges
    // 8. Call expressions → unresolved calls edges
    // 9. Extends/implements → extends/implements edges
    // 10. Decorators → decorates edges
    // 11. JSDoc comments → docstring on nodes
  }
};
```

### Chained call encoding

For chained method calls like `foo().bar()`, the extractor encodes the receiver as `<inner>().method_name` in the unresolved reference. This lets the resolver later match `bar` on the type that `foo()` returns.

Pattern: `CHAIN_SHAPE = /^(.+)\(\)\.(\w+)$/`

Languages with chain encoding: java, kotlin, csharp, swift, rust, go, scala, dart, objc, pascal
Languages with `::` scoped chains (Rust): scoped_chain_languages = ['rust']

## 5. File Scanning

### Git-based fast path

In git repos, use `git ls-files -z -c --recurse-submodules` (tracked) + `git ls-files -z -o --exclude-standard` (untracked) to enumerate visible files. Then apply default ignore patterns.

### Fallback filesystem walk

For non-git projects, walk the filesystem respecting `.gitignore` at every level (nested directories can have their own `.gitignore`).

### Default ignore directories

node_modules, bower_components, jspm_packages, web_modules, .yarn, .pnpm-store,
.next, .nuxt, .svelte-kit, .turbo, .vite, .parcel-cache, .angular, .docusaurus,
storybook-static, .vinxi, .nitro, out-tsc, .vercel, .netlify, .wrangler,
dist, build, out, .output, coverage, .nyc_output, __pycache__, __pypackages__,
.venv, venv, .pixi, .pdm-build, .mypy_cache, .pytest_cache, .ruff_cache, .tox,
.nox, .hypothesis, .ipynb_checkpoints, .eggs, target, .gradle, obj, vendor,
.build, Pods, Carthage, DerivedData, .swiftpm, .dart_tool, .pub-cache, .cxx,
.externalNativeBuild, vcpkg_installed, .bloop, .metals, lua_modules, .luarocks,
__history, __recovery, .cache

### Change detection (sync)

Use `git status --porcelain --no-renames` to detect:
- `??` → added (new untracked files)
- `D` → deleted
- Everything else → modified

### Sync fallback

When git is unavailable, hash-compare all file content hashes against stored `content_hash` in the `files` table.

## 6. ExtractionOrchestrator

```typescript
class ExtractionOrchestrator {
  constructor(private rootDir: string, private queries: QueryBuilder) {}

  async indexAll(onProgress?, signal?, verbose?): Promise<IndexResult>
  async indexFiles(filePaths: string[]): Promise<IndexResult>
  async sync(onProgress?): Promise<SyncResult>

  // Internals:
  // 1. Scan directory → get file list
  // 2. Detect framework names from file list
  // 3. Load needed WASM grammars
  // 4. Spawn parse worker thread
  // 5. Batch-read files (10 at a time), send to worker for parsing
  // 6. Store results in DB on main thread (SQLite not thread-safe)
  // 7. Worker recycles every 250 files to reclaim WASM memory
  // 8. Each parse has a timeout (10s base + 10s per 100KB)
  // 9. Retry WASM-crashed files once on a fresh worker
  // 10. Max file size: 1MB
  // 11. After indexing, run reference resolver
}
```

### Worker thread lifecycle

- Parse worker runs in a Node.js `Worker` thread to keep the main thread responsive
- Worker receives `{ type: 'parse', id, filePath, content, frameworkNames }` messages
- Worker sends back `{ type: 'parse-result', id, result }`
- Worker is recycled every 250 parses (WASM memory leak mitigation)
- Worker is killed on timeout (10s + 10s/100KB) — WASM can hang on pathological files
- Fallback to in-process parsing when worker module is unavailable (e.g., tests)
- Grammars are loaded in the worker via `{ type: 'load-grammars', languages }`
