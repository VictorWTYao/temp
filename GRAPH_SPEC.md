# CodeGraph Graph Traversal Specification

## 1. Overview

The graph layer provides BFS/DFS traversal, call graph analysis, type hierarchy, impact analysis, and path finding over the node-edge graph.

## 2. GraphTraverser

```typescript
class GraphTraverser {
  constructor(private queries: QueryBuilder) {}
  
  // BFS traversal — edges sorted by priority (contains > calls > references)
  traverseBFS(startId: string, options?: TraversalOptions): Subgraph
  
  // DFS traversal
  traverseDFS(startId: string, options?: TraversalOptions): Subgraph
  
  // Call graph
  getCallers(nodeId: string, maxDepth?: number): Array<{node: Node, edge: Edge}>
  getCallees(nodeId: string, maxDepth?: number): Array<{node: Node, edge: Edge}>
  getCallGraph(nodeId: string, depth?: number): Subgraph
  
  // Type hierarchy
  getTypeHierarchy(nodeId: string): Subgraph
  
  // Usage analysis
  findUsages(nodeId: string): Array<{node: Node, edge: Edge}>
  
  // Impact analysis
  getImpactRadius(nodeId: string, maxDepth?: number): Subgraph
  
  // Path finding (BFS shortest path)
  findPath(fromId: string, toId: string, edgeKinds?: EdgeKind[]): Array<{node: Node, edge: Edge | null}> | null
  
  // Containment hierarchy
  getAncestors(nodeId: string): Node[]
  getChildren(nodeId: string): Node[]
}
```

## 3. TraversalOptions

```typescript
interface TraversalOptions {
  maxDepth?: number;            // default: Infinity
  edgeKinds?: EdgeKind[];       // default: all
  nodeKinds?: NodeKind[];       // default: all
  direction?: 'outgoing' | 'incoming' | 'both';  // default: 'outgoing'
  limit?: number;               // default: 1000
  includeStart?: boolean;       // default: true
}
```

## 4. BFS Traversal (traverseBFS)

Algorithm:
```
1. Look up start node by ID. Return empty Subgraph if not found.
2. Initialize: nodes Map, edges array, visited Set, queue with {node, edge: null, depth: 0}
3. While queue is not empty AND nodes.size < limit:
   a. Dequeue step
   b. Skip if visited (visited set check)
   c. Add node to nodes map
   d. Add edge to edges (if not null)
   e. Skip if depth >= maxDepth
   f. Get adjacent edges (filtered by direction + edgeKinds)
   g. Sort edges: contains first (priority 0), calls (1), everything else (2)
   h. Batch-fetch unvisited neighbor nodes in one query
   i. Filter by nodeKinds if specified
   j. Enqueue unvisited neighbors
4. Return Subgraph { nodes, edges, roots: [startId] }
```

### Edge adjacency logic

```typescript
function getAdjacentEdges(nodeId, direction, edgeKinds?): Edge[] {
  if (direction === 'outgoing') return queries.getOutgoingEdges(nodeId, edgeKinds);
  if (direction === 'incoming') return queries.getIncomingEdges(nodeId, edgeKinds);
  // both: union of outgoing + incoming (deduplicated)
}
```

## 5. Call Graph

### getCallers

Traverses INCOMING edges of kind `calls`, `references`, `imports`, `instantiates` (includes `instantiates` because constructing a class is calling its constructor).

Edge traversal is recursive up to `maxDepth`. Uses visited set to prevent cycles.

Batch-fetches caller nodes (was N+1 — fixed).

### getCallees

Traverses OUTGOING edges of kind `calls`, `references`, `imports`, `instantiates`. Symmetric with getCallers — a class's constructor IS a callee of the construction site.

### getCallGraph

Returns BOTH callers and callees (up to `depth` in each direction) in a single Subgraph with the focal node as root.

## 6. Type Hierarchy

### getTypeHierarchy

Returns both ancestors and descendants:

- **Ancestors** (what this extends/implements): Recursively follow outgoing `extends` and `implements` edges
- **Descendants** (what extends/implements this): Recursively follow incoming `extends` and `implements` edges

## 7. Impact Radius

### getImpactRadius

Returns all nodes that could be affected by changes to the focal node:

1. Start at the focal node
2. For container nodes (class, interface, struct, trait, protocol, module, enum), also traverse INTO their children (contains edges) at the SAME depth — callers of contained methods should appear in impact
3. Traverse INCOMING edges of ALL kinds EXCEPT `contains` — a container "contains" its members but doesn't *depend* on them, so following `contains` upward would climb to the parent class and re-expand every sibling member (explosion)
4. Recursive up to `maxDepth`

## 8. Path Finding

### findPath

BFS shortest path between two nodes:

1. Look up both nodes by ID; return null if either missing
2. BFS from `fromId`, tracking path of {node, edge} objects
3. For each edge, batch-fetch unvisited targets
4. Return path when `toId` is reached; null if unreachable

## 9. Subgraph

```typescript
interface Subgraph {
  nodes: Map<string, Node>;
  edges: Edge[];
  roots: string[];
  confidence?: 'high' | 'low';  // set by search, not traversal
}
```

## 10. GraphQueryManager

Higher-level graph queries:

```typescript
class GraphQueryManager {
  constructor(private queries: QueryBuilder) {}

  getContext(nodeId: string): Context
  // Returns: focal node, ancestors, children, incoming refs, outgoing refs, types, imports

  getFileDependencies(filePath: string): string[]
  // Files this file imports from

  getFileDependents(filePath: string): string[]
  // Files that import this file

  findCircularDependencies(): string[][]
  // Detect cycles in the import graph

  findDeadCode(kinds?: NodeKind[]): Node[]
  // Unreferenced symbols (no incoming edges other than 'contains')

  getNodeMetrics(nodeId: string): {
    incomingEdgeCount: number;
    outgoingEdgeCount: number;
    callCount: number;
    callerCount: number;
    childCount: number;
    depth: number;
  }
}
```

### Context

```typescript
interface Context {
  focal: Node;
  ancestors: Node[];        // containment hierarchy (parents)
  children: Node[];          // contained nodes
  incomingRefs: Array<{node: Node, edge: Edge}>;  // who uses this
  outgoingRefs: Array<{node: Node, edge: Edge}>;  // what this uses
  types: Node[];             // type information
  imports: Node[];           // relevant imports
}
```

### Dead Code Detection

For a symbol to be "dead":
1. It has NO incoming edges (excluding `contains` edges — being contained doesn't make something "used")
2. Filter by the requested kinds (default: function, method, class)
3. Exclude symbols that are exported (exported = intentionally public API)

### Circular Dependency Detection

Use Tarjan's algorithm or iterative DFS on the file-level `imports` graph:
1. Build adjacency list: for each file, get files it imports
2. Run SCC (strongly connected components) detection
3. Return all SCCs with size > 1 as cycles
