# CodeGraph Context Builder Specification

## 1. Overview

The ContextBuilder combines FTS5 full-text search with graph traversal to produce rich context for AI agents. It's the bridge between raw graph queries and agent-readable output.

## 2. Public API

```typescript
class ContextBuilder {
  constructor(
    private projectRoot: string,
    private queries: QueryBuilder,
    private traverser: GraphTraverser
  ) {}

  // Get source code for a node
  async getCode(nodeId: string): Promise<string | null>

  // Find relevant subgraph for a natural language query
  async findRelevantContext(
    query: string,
    options?: FindRelevantContextOptions
  ): Promise<Subgraph>

  // Full task context builder
  async buildContext(
    input: TaskInput,           // string | { title, description }
    options?: BuildContextOptions
  ): Promise<TaskContext | string>  // returns TaskContext or formatted string
}
```

## 3. FTS5 Search

### SearchQueryParser

Parses user queries into FTS5-compatible search terms:

```typescript
function extractSearchTerms(query: string): string[] {
  // 1. Extract CamelCase identifiers (e.g., "UserService", "calculateTotal")
  // 2. Extract snake_case identifiers (e.g., "user_service")
  // 3. Extract SCREAMING_SNAKE_CASE (e.g., "MAX_RETRIES")
  // 4. Extract ALL_CAPS acronyms (e.g., "API", "HTTP")
  // 5. Extract dot.notation (e.g., "app.isPackaged" → "app", "isPackaged")
  // 6. Extract single-word identifiers (2+ chars, not common English)
  // 7. Filter stopwords (the, and, how, does, what, etc.)
  // 8. Prefix-match each term with * for FTS5 (e.g., "User*", "Service*")
}

function isCommonWord(word: string): boolean {
  // Return true for: the, a, an, in, on, at, to, for, of, with,
  // by, from, as, is, was, are, were, be, been, being, have, has,
  // had, do, does, did, but, or, if, then, else, when, where, how,
  // what, why, who, which, this, that, these, those, it, its, and, not, no
}
```

### FTS5 Query Building

```typescript
function buildFTSQuery(terms: string[]): string {
  // Join terms with space for BM25 ranking
  // Each term is prefix-matched with *
  // Example: "calculate* total* user* service*"
  return terms.map(t => `${t.toLowerCase()}*`).join(' ');
}
```

### Scoring

FTS5 returns BM25-ranked results. Additional scoring factors:

1. **Exact match bonus**: name matches query term exactly (not just prefix) gets +0.2
2. **File reputation**: files with more total matches get priority
3. **Project name downweight**: if a term matches the project name (from package.json, go.mod, or directory name), it's downweighted — the project name is not a discriminative symbol
4. **PascalCase disambiguation bias**: when multiple terms match and one is PascalCase (type-like), that term biases toward definitions in files where the other terms also appear

```typescript
interface SearchResult {
  node: Node;
  score: number;       // 0-1 normalized
  highlights?: string[];  // matched text snippets
}

async function searchNodes(
  query: string,
  options?: { kinds?, languages?, includePatterns?, excludePatterns?, limit?, offset?, caseSensitive? }
): Promise<SearchResult[]>
```

## 4. Symbol Extraction from Natural Language

```typescript
function extractSymbolsFromQuery(query: string): string[] {
  // Extract likely code symbol names from natural language:
  // - CamelCase: "UserService", "signInWithGoogle"
  // - snake_case: "user_service", "sign_in"  
  // - SCREAMING_SNAKE: "MAX_RETRIES"
  // - Dot notation: "app.isPackaged" (extracts segments)
  // - PascalCase acronyms: "API", "HTTP"
  // - Single words 2+ chars, not common English words
  
  // Returns deduplicated array
}
```

## 5. findRelevantContext

```typescript
async function findRelevantContext(
  query: string,
  options?: FindRelevantContextOptions
): Promise<Subgraph> {
  // 1. Extract symbols from the query
  // 2. Run FTS5 search with extracted terms
  // 3. Filter results by minScore (default: 0.3)
  // 4. For top results (default: 5), expand the graph:
  //    - Traverse outgoing edges (calls, references, extends) up to traversalDepth
  //    - Follow contains edges to get children
  //    - Follow incoming reference edges to get usages
  // 5. Deduplicate and cap at maxNodes (default: 50)
  // 6. Return Subgraph with confidence marker
}
```

### Confidence marker

- `'high'`: 2+ distinct query terms produced search results with corroborating entry points
- `'low'`: Only isolated common-word matches, no corroboration

## 6. buildContext

```typescript
async function buildContext(
  input: TaskInput,
  options?: BuildContextOptions
): Promise<TaskContext | string> {
  // Options:
  //   maxNodes: 50
  //   maxCodeBlocks: 10
  //   maxCodeBlockSize: 2000
  //   includeCode: true
  //   format: 'markdown' | 'json'
  //   searchLimit: 5
  //   traversalDepth: 2
  //   minScore: 0.3

  // 1. Parse input (string → extract query, {title, description} → extract from title + description)
  // 2. Find relevant context (FTS5 search + graph expansion)
  // 3. Extract code blocks for key nodes
  // 4. Build summary
  // 5. Format as markdown or JSON

  // Output:
  const result: TaskContext = {
    query: string,           // original query
    subgraph: Subgraph,      // relevant nodes + edges
    entryPoints: Node[],     // top search results
    codeBlocks: CodeBlock[], // extracted source code
    relatedFiles: string[],  // files involved
    summary: string,         // brief description of context
    stats: {
      nodeCount: number,
      edgeCount: number,
      fileCount: number,
      codeBlockCount: number,
      totalCodeSize: number,
    },
  };
}
```

### CodeBlock

```typescript
interface CodeBlock {
  content: string;      // the code text
  filePath: string;     // file path
  startLine: number;    // starting line (1-indexed)
  endLine: number;      // ending line
  language: Language;   // for syntax highlighting
  node?: Node;          // associated node if extracted
}
```

## 7. Code Extraction (getCode)

```typescript
async function getCode(nodeId: string): Promise<string | null> {
  // 1. Look up the node
  // 2. Read the file from disk
  // 3. Extract lines between startLine and endLine
  // 4. Return with line numbers (optional)
}
```

## 8. Output Formatting

### Markdown formatter

```typescript
function formatContextAsMarkdown(context: TaskContext): string {
  // Output structure:
  // 
  // # Task Context: {query}
  // 
  // ## Summary
  // {summary}
  // 
  // ## Entry Points
  // - **{name}** (`{filePath}:{line}`) — {kind} — {docstring snippet}
  // 
  // ## Related Files
  // - {filePath} ({nodeCount} nodes)
  // 
  // ## Code Blocks
  // ### {name} ({filePath}:{startLine}-{endLine})
  // ```{language}
  // {code content}
  // ```
  // 
  // ## Dependency Graph
  // {name} → {callee name} ({kind})
  // 
  // ## Stats
  // - Nodes: {n}
  // - Edges: {e}
  // - Files: {f}
}
```

### JSON formatter

```typescript
function formatContextAsJson(context: TaskContext): string {
  return JSON.stringify(context, null, 2);
}
```

## 9. FindRelevantContextOptions

```typescript
interface FindRelevantContextOptions {
  searchLimit?: number;        // default: 5
  traversalDepth?: number;     // default: 2
  maxNodes?: number;           // default: 50
  minScore?: number;           // default: 0.3
  edgeKinds?: EdgeKind[];      // default: all
  nodeKinds?: NodeKind[];      // default: all
}
```

## 10. BuildContextOptions

```typescript
interface BuildContextOptions {
  maxNodes?: number;           // default: 50
  maxCodeBlocks?: number;      // default: 10
  maxCodeBlockSize?: number;   // default: 2000
  includeCode?: boolean;       // default: true
  format?: 'markdown' | 'json'; // default: 'markdown'
  searchLimit?: number;        // default: 5
  traversalDepth?: number;     // default: 2
  minScore?: number;           // default: 0.3
}
```

## 11. TaskInput

```typescript
type TaskInput = string | { title: string; description?: string };
```

When a string is passed, it's used directly as the query. When an object with `{title, description}` is passed, symbols are extracted from both and combined for FTS5 search.

## 12. TaskContext

```typescript
interface TaskContext {
  query: string;
  subgraph: Subgraph;
  entryPoints: Node[];
  codeBlocks: CodeBlock[];
  relatedFiles: string[];
  summary: string;
  stats: {
    nodeCount: number;
    edgeCount: number;
    fileCount: number;
    codeBlockCount: number;
    totalCodeSize: number;
  };
}
```
