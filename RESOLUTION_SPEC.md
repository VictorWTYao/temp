# CodeGraph Resolution Specification

## 1. Overview

After extraction, the graph has nodes + static edges + unresolved references. The Reference Resolver runs as a post-extraction pass to resolve those references and create additional edges.

## 2. Architecture

```
ReferenceResolver
├── ImportResolver — resolves import statements to file-level edges
├── NameMatcher — matches names across files using qualified name heuristics
├── FrameworkResolvers (22) — emit route nodes + references for framework patterns
├── CallbackSynthesizer — synthesizes edges for dynamic dispatch
├── PathAliasResolver — tsconfig paths / Cargo workspace aliases
├── GoModuleResolver — Go module path resolution
└── WorkspacePackages — monorepo workspace member discovery

Resolution runs in batches (5000 refs/batch) to keep memory bounded.
After resolution, callback synthesis creates dynamic-dispatch edges.
A second conformance pass resolves chained calls on supertypes.
```

## 3. Resolution Strategy (Priority Order)

For each unresolved reference, try strategies in order:

### Strategy 0: Built-in/external filter

Skip if the reference name matches a known built-in (language standard library, etc.):

- **JS/TS**: console, window, document, Promise, Array, Object, Math, JSON, setTimeout, fetch, require, module, React hooks (useState, useEffect, etc.)
- **Python**: print, len, range, str, int, float, list, dict, set, open, type, isinstance, super, None, True, False, + built-in methods (append, extend, join, split, etc.)
- **Go**: stdlib packages (fmt, os, http, etc.), built-ins (make, new, len, append, etc.)
- **Pascal**: System.*, Vcl.*, Fmx.*, etc.; common units (SysUtils, Classes, Math, Forms, etc.)
- **C/C++**: std:: prefix, stdlib functions (printf, malloc, memcpy, etc. — only if no user symbol with same name exists)
- Built-in filter is ONLY applied when there's no user-defined symbol with that name in the codebase

### Strategy 1: JVM FQN import resolution

For JVM languages: `import com.example.Bar` resolves directly through the `qualified_name` index. Unambiguous even when multiple `Bar` classes exist in different packages.

### Strategy 2: Razor/Blazor @using resolution

For `.razor`/`.cshtml` files, collect `@using` namespaces (own + cascade from `_Imports.razor` up to project root). A simple type name + `@using Namespace` → `Namespace::TypeName` qualified name match.

### Strategy 3: Framework-specific resolution (22 resolvers)

Each framework resolver has:
- `detect(context): boolean` — scans project files/imports for framework markers
- `resolve(ref, context): ResolvedRef | null` — tries framework-specific patterns
- `claimsReference?(name): boolean` — optional pre-filter for framework-specific names
- `postExtract?(context): Node[]` — optional cross-file finalization

Framework resolvers run for every unresolved reference. High-confidence results (≥0.9) return immediately.

### Strategy 4: Import-based resolution

Resolve import bindings:
1. Parse the file's import statements
2. Match the reference name to a local import binding
3. Follow the import to find the target file/module
4. Handle re-exports (barrel files: `export { foo } from './bar'`)

For JS/TS: resolve named imports, default imports, require(), dynamic imports
For Python: resolve from/import statements
For Go: resolve import paths
For Rust: resolve use statements
For C/C++: resolve #include paths
For PHP: resolve include/require paths (file-only, never falls through to name-matching)
For Pascal: resolve uses clauses
For Java/Kotlin: resolve import statements (JVM FQN already handled above)

### Strategy 5: Name matching

When imports don't resolve, try heuristic name matching:

```typescript
function matchReference(ref, context): ResolvedRef | null {
  // 1. Exact name match — the name exists as a symbol somewhere
  // 2. Same-file first — prefer symbols in the same file
  // 3. Exported over private — prefer exported symbols
  // 4. Same language family — prefer symbols in same or compatible language
  // 5. Same directory — prefer symbols in nearby files
  // 6. Closest line range — prefer symbol defined closest to the reference
}
```

### Strategy 6: Chained call resolution (deferred)

For `receiver().method()` patterns where the first pass couldn't resolve:
- Deferred to second pass (after implements/extends edges exist)
- Dotted chain: matchDottedCallChain — resolves by walking the receiver type's methods + supertypes
- Scoped chain (Rust `::`): matchScopedCallChain — resolves module paths

### Language gates

Apply to Strategies 4-5 only:
- `references` (type usage): STRICT — target must be same language family
- `imports`: both-known — only filter when both source and target are in known families
- `calls`: no gate — cross-language bridges preserved intentionally

Language families: js-like (ts/tsx/js/jsx), python-like, go, rust, jvm (java/kotlin/scala), dotnet (csharp), php, ruby, swift, objc, dart, pascal

## 4. Resolution types

```typescript
interface UnresolvedRef {
  fromNodeId: string;
  referenceName: string;
  referenceKind: EdgeKind | 'function_ref';
  line: number;
  column: number;
  filePath: string;
  language: Language;
}

interface ResolvedRef {
  original: UnresolvedRef;
  targetNodeId: string;
  confidence: number;  // 0-1
  resolvedBy: string;  // 'import' | 'name-match' | 'framework-*' | 'function-ref'
}

interface ResolutionResult {
  resolved: ResolvedRef[];
  unresolved: UnresolvedRef[];
  stats: {
    total: number;
    resolved: number;
    unresolved: number;
    byMethod: Record<string, number>;
  };
}
```

### Confidence levels

- 0.95-1.0: Exact import match, JVM FQN, this-member fn ref
- 0.9: Framework-specific match, Razor @using match
- 0.8-0.89: Import-based (re-export chain)
- 0.7-0.79: Name matching — same file, exported
- 0.6-0.69: Name matching — same language family, nearby directory
- 0.5-0.59: Name matching — cross-language, loose heuristic

## 5. ReferenceKind: Function Ref (`function_ref`)

A special internal-only kind (#756). A function name used as a VALUE (callback registration, event handler):

```javascript
btn.on('click', this.handleClick)  // 'this.handleClick' is a function_ref
```

Rules:
- `this.<member>` refs resolve ONLY to the enclosing class's own members (same file)
- Non-this refs: import-based resolution first, then matchFunctionRef (same-file first, function/method targets only)
- Never reaches the framework or fuzzy strategies
- Stored as `references` edges with `metadata.fnRef: true`

Deferred pass (#808): `this.<member>` refs whose member is NOT on the enclosing class itself — retry in the conformance pass once implements/extends edges exist, resolving the member on the nearest supertype that declares it.

## 6. Callback Edge Synthesis

Runs AFTER base resolution (all `calls` edges persist). Synthesizes dynamic dispatch edges that static parsing can't detect.

### Phase 1: Field-backed observer

```typescript
// Common pattern:
class Subject {
  private callbacks = new Set<Function>();
  onUpdate(cb) { this.callbacks.add(cb); }         // registrar
  triggerUpdate() { this.callbacks.forEach(cb => cb()); }  // dispatcher
}
scene.onUpdate(this.triggerRender)                  // registration
// → synthesize triggerUpdate → triggerRender edge
```

Detection: registrar method names matching `on[A-Z]\w*|subscribe|addListener|addEventListener|register|watch|listen|addCallback`. Dispatcher method names containing `emit|trigger|notify|dispatch|fire|publish|flush`.

### Phase 2: String-keyed EventEmitter

```typescript
// EventEmitter pattern:
this.on('mount', function onmount() { ... });       // registration
fn.emit('mount', this);                              // dispatch
// → synthesize emit → onmount edge
```

Capped at `EVENT_FANOUT_CAP = 6` per event name (generic names like 'error' have too many handlers).

### React re-render

`this.setState(...)` → synthesize edge to the containing component's `render()` method.

### JSX child

`<MyComponent>` in a render function → synthesize edge from the parent's render to the child component.

### Closure collection pattern

A method appends a closure to a collection; another method iterates that collection invoking each element (`coll.forEach { $0() }` / `{ it() }`). Pairing dispatcher to same-named registrars (`.append`/`.add`/`.push`/`.insert`).

All synthesized edges: `provenance: 'heuristic'`, with `metadata.synthesizedBy` set.

## 7. Edge Creation from Resolution

```typescript
function createEdge(ref: ResolvedRef): Edge {
  let kind = ref.original.referenceKind === 'function_ref' ? 'references' : ref.original.referenceKind;

  // Promote 'extends' to 'implements' when class extends interface
  if (kind === 'extends') {
    const target = queries.getNodeById(ref.targetNodeId);
    if (target && (target.kind === 'interface' || target.kind === 'protocol')) {
      const source = queries.getNodeById(ref.original.fromNodeId);
      if (source && source.kind !== 'interface' && source.kind !== 'protocol') {
        kind = 'implements';
      }
    }
  }

  // Promote 'calls' to 'instantiates' when target is a class/struct
  if (kind === 'calls') {
    const target = queries.getNodeById(ref.targetNodeId);
    if (target && (target.kind === 'class' || target.kind === 'struct')) {
      kind = 'instantiates';
    }
  }

  return {
    source: ref.original.fromNodeId,
    target: ref.targetNodeId,
    kind,
    line: ref.original.line,
    column: ref.original.column,
    metadata: { confidence: ref.confidence, resolvedBy: ref.resolvedBy },
    provenance: 'heuristic',
  };
}
```

## 8. Caching

All per-resolver caches are LRU-bounded (default limit: 5000 entries, configurable via `CODEGRAPH_RESOLVER_CACHE_SIZE`):

- `nodeCache`: filePath → Node[] (nodes per file)
- `fileCache`: filePath → string | null (file content)
- `nameCache`: name → Node[] (nodes by name)
- `lowerNameCache`: lowercase(name) → Node[]
- `qualifiedNameCache`: qualifiedName → Node[]
- `importMappingCache`: filePath → ImportMapping[]
- `reExportCache`: filePath → ReExport[]

Caches are cleared when new edges are inserted (resolution produces new data).

## 9. ResolutionContext

```typescript
interface ResolutionContext {
  getNodesInFile(filePath: string): Node[];
  getNodesByName(name: string): Node[];
  getNodesByQualifiedName(qualifiedName: string): Node[];
  getNodesByKind(kind: NodeKind): Node[];
  getNodesByLowerName(lowerName: string): Node[];
  getNodeById(id: string): Node | null;
  getImportMappings(filePath: string, language: Language): ImportMapping[];
  getReExports(filePath: string, language: Language): ReExport[];
  getSupertypes(typeName: string, language: Language): string[];
  getProjectAliases(): AliasMap | null;
  getGoModule(): GoModule | null;
  getWorkspacePackages(): WorkspacePackages | null;
  getCppIncludeDirs(): string[];
  fileExists(filePath: string): boolean;
  readFile(filePath: string): string | null;
  getAllFiles(): string[];
  getProjectRoot(): string;
  listDirectories(relativePath: string): string[];
}
```

## 10. Framework Resolver Contract

```typescript
interface FrameworkResolver {
  name: string;                            // unique framework name
  detect(context: ResolutionContext): boolean;  // is this framework present?
  resolve(ref: UnresolvedRef, context: ResolutionContext): ResolvedRef | null;
  claimsReference?(name: string): boolean;      // optional pre-filter
  postExtract?(context: ResolutionContext): Node[];  // optional cross-file finalization
}
```

### Detection approach

Each resolver's `detect()` should check for framework markers:
- **Express**: `require('express')` or `from 'express'` imports
- **React**: `from 'react'` imports, JSX usage
- **Django**: `django` in installed apps, `urlpatterns` in Python files
- **Laravel**: artisan files, `routes/web.php`
- etc.

### Route node emission

Framework resolvers emit `route` nodes with:
- `kind: 'route'`
- `name`: URL pattern (e.g., `/api/users/:id`)
- Metadata includes HTTP method, handler name, handler file
- Outgoing `references` edge links route → handler function

After all resolution, expose:
```typescript
getRoutingManifest(): { entries: Array<{url, handler, handlerFile, handlerLine, handlerKind}>, topHandlerFile, totalRoutes } | null
getTopRouteFile(): { filePath, routeCount, totalRoutes } | null  // file with densest route concentration
```
