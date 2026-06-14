# CodeGraph opencode Custom Tools Specification

## 1. Overview

CodeGraph exposes its graph as opencode Custom Tools — TypeScript files in `.opencode/tools/`. Each file becomes a tool the LLM can call, auto-discovered by opencode at startup.

**The filename is the tool name.** Example: `explore.ts` → the agent calls `codegraph_explore`.

## 2. Tool Definitions

### explore.ts — PRIMARY tool

The most important tool. The agent calls this for structural/flow questions.

```typescript
// .opencode/tools/explore.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Explore the code graph to find flow paths between named symbols. " +
    "Pass a precise bag of symbol names (including qualified names like Class.method). " +
    "Returns the call/import/extends path among those symbols with inline source code. " +
    "This is the PRIMARY code intelligence tool — use it for any structural/flow question.",

  args: {
    symbols: tool.schema
      .array(tool.schema.string())
      .describe("Symbol names to explore, e.g. ['mutateElement', 'renderStaticScene', 'App']"),
  },

  async execute(args, context) {
    // 1. Get the CodeGraph instance for this project
    const cg = await getCodeGraph(context.worktree)

    // 2. For each symbol, search by name (getNodesByName — returns ALL overloads)
    //    Also try FTS5 search for partial matches
    const allFoundNodes: Node[] = []
    for (const sym of args.symbols) {
      const exact = cg.getNodesByName(sym)
      const fuzzy = cg.searchNodes(sym, { limit: 5 })
      allFoundNodes.push(...exact, ...fuzzy.map(r => r.node))
    }

    // 3. Deduplicate nodes by ID
    const nodes = deduplicate(allFoundNodes)

    // 4. Build the flow: find paths between the named symbols
    //    - For each pair of nodes, try findPath
    //    - Collect all callers/callees for each node (depth 1-3 depending on set size)
    //    - Synthesize: walk edges, find connections, return the flow
    const flow = buildFlowBetweenSymbols(cg, nodes)

    // 5. Adapt output budget to repo size
    const fileCount = cg.getFiles().length
    const budget = getExploreOutputBudget(fileCount)

    // 6. Format output:
    //    - Lead with the flow path (if found)
    //    - Then inline source for key symbols
    //    - Caller/callee trail
    //    - "Stale file" warning if watcher has pending changes

    return formatExploreOutput(flow, nodes, budget)
  },
})
```

### node.ts — SECONDARY tool (depth tool)

Called for depth on a specific symbol after explore found it.

```typescript
// .opencode/tools/node.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Get detailed information about a named symbol. Returns the full " +
    "source body, caller trail (who calls this), callee trail (what this calls), " +
    "type hierarchy, and usage locations. For AMBIGUOUS names, returns ALL " +
    "overloads' bodies in one response so the agent never needs to Read a file " +
    "to find the right overload. This is SECONDAY tool — use explore first, " +
    "then node for depth on a specific symbol.",

  args: {
    name: tool.schema.string().describe("Symbol name (exact or qualified, e.g. 'render' or 'App.render')"),
  },

  async execute(args, context) {
    const cg = await getCodeGraph(context.worktree)
    const nodes = cg.getNodesByName(args.name)

    if (nodes.length === 0) {
      // FTS5 fallback
      const results = cg.searchNodes(args.name, { limit: 5 })
      if (results.length === 0) {
        return `Symbol "${args.name}" not found in the code graph.`
      }
      nodes.push(...results.map(r => r.node))
    }

    // For each matching node, build context
    const parts = nodes.map(node => {
      const ctx = cg.getContext(node.id)
      const code = await cg.getCode(node.id)
      const callers = cg.getCallers(node.id, 2)
      const callees = cg.getCallees(node.id, 2)

      return formatNodeOutput(node, ctx, code, callers, callees)
    })

    return parts.join('\n\n---\n\n')
  },
})
```

### search.ts — Full-text search tool

```typescript
// .opencode/tools/search.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Full-text search across all indexed symbols. Searches symbol names, " +
    "qualified names, docstrings, and signatures. Returns ranked results with " +
    "relevance scores. Use when you don't know the exact symbol name.",

  args: {
    query: tool.schema.string().describe("Search query (natural language or symbol fragment)"),
    limit: tool.schema.number().optional().describe("Max results (default: 20)"),
    kinds: tool.schema.array(tool.schema.string()).optional()
      .describe("Filter by node kinds: class, function, method, etc."),
  },

  async execute(args, context) {
    const cg = await getCodeGraph(context.worktree)
    const results = cg.searchNodes(args.query, {
      limit: args.limit ?? 20,
      kinds: args.kinds as any,
    })

    if (results.length === 0) {
      return `No results found for "${args.query}".`
    }

    return results.map(r =>
      `[${r.score.toFixed(2)}] ${r.node.name} — ${r.node.kind} (${r.node.filePath}:${r.node.startLine})\n` +
      `  Qualified: ${r.node.qualifiedName}\n` +
      (r.node.signature ? `  Signature: ${r.node.signature}\n` : '')
    ).join('\n')
  },
})
```

### status.ts — Index status tool

```typescript
// .opencode/tools/status.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Get the current status of the code graph index. Returns whether " +
    "the project is indexed, how many files/nodes/edges, and whether a re-index " +
    "is recommended due to a newer extraction engine.",

  args: {},

  async execute(args, context) {
    const cg = await getCodeGraph(context.worktree)
    if (!cg) {
      return "CodeGraph is not initialized for this project. Run `codegraph init` first."
    }

    const stats = cg.getStats()
    const stale = cg.isIndexStale()
    const watcher = cg.isWatching()
    const backend = cg.getBackend()
    const journal = cg.getJournalMode()

    return [
      `**CodeGraph Status**`,
      ``,
      `Project: ${cg.getProjectRoot()}`,
      `Files: ${stats.fileCount}`,
      `Nodes: ${stats.nodeCount}`,
      `Edges: ${stats.edgeCount}`,
      `Database: ${formatSize(stats.dbSizeBytes)}`,
      `Backend: ${backend} (${journal})`,
      `Watching: ${watcher ? 'active' : 'inactive'}`,
      `Stale index: ${stale ? '⚠️ yes — run `codegraph index` to upgrade' : 'no'}`,
      `Last updated: ${new Date(stats.lastUpdated).toISOString()}`,
      `Languages: ${Object.entries(stats.filesByLanguage)
        .filter(([, count]) => count > 0)
        .map(([lang, count]) => `${lang} (${count})`)
        .join(', ')}`,
    ].join('\n')
  },
})
```

### callers.ts — Caller analysis tool

```typescript
// .opencode/tools/callers.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Find who calls a given function or method. Returns call chain " +
    "up to the specified depth. Useful for understanding impact of changes.",

  args: {
    name: tool.schema.string().describe("Function/method name"),
    depth: tool.schema.number().optional().describe("Max call chain depth (default: 2)"),
  },

  async execute(args, context) {
    const cg = await getCodeGraph(context.worktree)
    const nodes = cg.getNodesByName(args.name)
    if (nodes.length === 0) return `Symbol "${args.name}" not found.`

    const results = nodes.flatMap(node => {
      const callers = cg.getCallers(node.id, args.depth ?? 2)
      return callers.map(({ node: caller, edge }) =>
        `${caller.name} (${caller.filePath}:${edge.line}) — via ${edge.kind}`
      )
    })

    if (results.length === 0) return `No callers found for "${args.name}".`
    return results.join('\n')
  },
})
```

### callees.ts — Callee analysis tool

```typescript
// .opencode/tools/callees.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Find what a given function or method calls. Returns call chain " +
    "down to the specified depth.",

  args: {
    name: tool.schema.string().describe("Function/method name"),
    depth: tool.schema.number().optional().describe("Max call chain depth (default: 2)"),
  },

  async execute(args, context) {
    const cg = await getCodeGraph(context.worktree)
    const nodes = cg.getNodesByName(args.name)
    if (nodes.length === 0) return `Symbol "${args.name}" not found.`

    const results = nodes.flatMap(node => {
      const callees = cg.getCallees(node.id, args.depth ?? 2)
      return callees.map(({ node: callee, edge }) =>
        `${callee.name} (${callee.filePath}:${edge.line}) — via ${edge.kind}`
      )
    })

    if (results.length === 0) return `No callees found for "${args.name}".`
    return results.join('\n')
  },
})
```

### impact.ts — Impact analysis tool

```typescript
// .opencode/tools/impact.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Calculate the impact radius of a symbol — what could break if " +
    "this symbol changes. Traverses incoming edges to find all dependents.",

  args: {
    name: tool.schema.string().describe("Symbol name"),
    depth: tool.schema.number().optional().describe("Max traversal depth (default: 3)"),
  },

  async execute(args, context) {
    const cg = await getCodeGraph(context.worktree)
    const nodes = cg.getNodesByName(args.name)
    if (nodes.length === 0) return `Symbol "${args.name}" not found.`

    const parts = nodes.map(node => {
      const impact = cg.getImpactRadius(node.id, args.depth ?? 3)
      return [
        `**${node.name}** (${node.filePath}:${node.startLine})`,
        `Impact radius: ${impact.nodes.size} nodes, ${impact.edges.length} edges`,
        ...Array.from(impact.nodes.values())
          .filter(n => n.id !== node.id)
          .map(n => `  - ${n.name} (${n.kind}) — ${n.filePath}:${n.startLine}`),
      ].join('\n')
    })

    return parts.join('\n\n---\n\n')
  },
})
```

## 3. Shared Utilities

### getCodeGraph

```typescript
// Shared utility to get or create a CodeGraph instance for the current project
import { CodeGraph } from '../path/to/codegraph'

let instance: CodeGraph | null = null

async function getCodeGraph(worktree: string): Promise<CodeGraph> {
  if (instance && instance.getProjectRoot() === worktree) return instance

  if (CodeGraph.isInitialized(worktree)) {
    instance = await CodeGraph.open(worktree, { sync: true })
  } else {
    instance = await CodeGraph.init(worktree, { index: true })
  }
  return instance
}
```

### Explore budget

```typescript
function getExploreBudget(fileCount: number): number {
  if (fileCount < 500) return 1
  if (fileCount < 5000) return 2
  if (fileCount < 15000) return 3
  if (fileCount < 25000) return 4
  return 5  // >= 25000
}

function getExploreOutputBudget(fileCount: number): {
  maxChars: number;
  maxFiles: number;
  maxCharsPerFile: number;
} {
  if (fileCount < 500) return { maxChars: 18000, maxFiles: 15, maxCharsPerFile: 3800 }
  if (fileCount < 5000) return { maxChars: 28000, maxFiles: 20, maxCharsPerFile: 4500 }
  if (fileCount < 15000) return { maxChars: 35000, maxFiles: 25, maxCharsPerFile: 5000 }
  if (fileCount < 25000) return { maxChars: 38000, maxFiles: 30, maxCharsPerFile: 5500 }
  return { maxChars: 38000, maxFiles: 35, maxCharsPerFile: 7000 }
  // Invariant: larger repos must never get smaller maxCharsPerFile
}
```

### Flow builder (buildFlowBetweenSymbols)

```typescript
function buildFlowBetweenSymbols(cg: CodeGraph, nodes: Node[]): FlowResult {
  // 1. For each node, collect callers + callees (depth 1-3)
  // 2. Build adjacency: which nodes connect to which via calls/imports/extends
  // 3. For each pair, try findPath
  // 4. Return the connected flow path with edges labeled
  // 5. Mark heuristic edges with [synthesized] tag
  // 6. Include "wiring site" for callback edges (where registration happens)
}
```

### Formatting helpers

```typescript
function formatExploreOutput(flow, nodes, budget): string {
  // 1. Priority: flow path first (if connected)
  // 2. Then named symbol source (sorted by file)
  // 3. Caller/callee trails
  // 4. Stale file warnings
  // 5. Never output "use Read" — tell agent "treat returned source as already Read"
}

function formatNodeOutput(node, ctx, code, callers, callees): string {
  // Format: name, kind, file:line, signature, docstring
  // Then full source body in code block
  // Then caller trail (depth-limited)
  // Then callee trail (depth-limited)
  // Then type info (implements/extends)
}
```

## 4. Error handling rules

- **Never use `isError: true` for expected conditions**: Symbol not found, project not indexed, file not in index → return a textResult with guidance
- `isError` is reserved for: security violations (path traversal), genuine malfunctions (DB corruption)
- Unindexed workspace: explore/node/search all detect this and return a short "not indexed" message rather than failing

## 5. Agent-facing guidance

This should be in a rule file or `AGENTS.md`:

```
## CodeGraph Tools

- `codegraph_explore` is the PRIMARY tool for structural/flow questions
- Pass a precise bag of symbol names (including qualified Class.method) — NOT natural language questions
- `codegraph_node` is for depth on a specific symbol after explore found it
- For ambiguous names, explore/node return ALL overloads — no need to Read
- Treat returned source code as already Read — the graph is the source of truth
- Explore budgets scale with repo size (1-5 calls); use the budget
