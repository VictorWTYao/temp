# CodeGraph Database Schema

## Technology

SQLite via `better-sqlite3` (native) with transparent fallback to `node-sqlite3-wasm`. WAL journal mode for concurrent reads. FTS5 for full-text search.

## DDL

Execute this on `init()` and run schema migrations via a `schema_versions` table.

### Schema version tracking

```sql
CREATE TABLE schema_versions (
    version INTEGER PRIMARY KEY,
    applied_at INTEGER NOT NULL,
    description TEXT
);

INSERT INTO schema_versions (version, applied_at, description)
VALUES (1, strftime('%s', 'now') * 1000, 'Initial schema');
```

### Nodes table

```sql
CREATE TABLE nodes (
    id TEXT PRIMARY KEY,
    kind TEXT NOT NULL,                   -- NodeKind value
    name TEXT NOT NULL,                   -- simple name
    qualified_name TEXT NOT NULL,         -- e.g. "src/util.ts::Helper.run"
    file_path TEXT NOT NULL,             -- relative path
    language TEXT NOT NULL,
    start_line INTEGER NOT NULL,          -- 1-indexed
    end_line INTEGER NOT NULL,
    start_column INTEGER NOT NULL,        -- 0-indexed
    end_column INTEGER NOT NULL,
    docstring TEXT,
    signature TEXT,
    visibility TEXT,                      -- public|private|protected|internal
    is_exported INTEGER DEFAULT 0,
    is_async INTEGER DEFAULT 0,
    is_static INTEGER DEFAULT 0,
    is_abstract INTEGER DEFAULT 0,
    decorators TEXT,                      -- JSON array
    type_parameters TEXT,                 -- JSON array
    return_type TEXT,                     -- normalized return type
    updated_at INTEGER NOT NULL
);

CREATE INDEX idx_nodes_kind ON nodes(kind);
CREATE INDEX idx_nodes_name ON nodes(name);
CREATE INDEX idx_nodes_qualified_name ON nodes(qualified_name);
CREATE INDEX idx_nodes_file_path ON nodes(file_path);
CREATE INDEX idx_nodes_language ON nodes(language);
CREATE INDEX idx_nodes_file_line ON nodes(file_path, start_line);
CREATE INDEX idx_nodes_lower_name ON nodes(lower(name));
```

### Edges table

```sql
CREATE TABLE edges (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source TEXT NOT NULL,                  -- source node ID
    target TEXT NOT NULL,                  -- target node ID
    kind TEXT NOT NULL,                    -- EdgeKind value
    metadata TEXT,                         -- JSON object (confidence, resolvedBy, etc.)
    line INTEGER,                          -- source line
    col INTEGER,                           -- source column
    provenance TEXT DEFAULT NULL,          -- tree-sitter|heuristic|scip
    FOREIGN KEY (source) REFERENCES nodes(id) ON DELETE CASCADE,
    FOREIGN KEY (target) REFERENCES nodes(id) ON DELETE CASCADE
);

CREATE INDEX idx_edges_kind ON edges(kind);
CREATE INDEX idx_edges_source_kind ON edges(source, kind);
CREATE INDEX idx_edges_target_kind ON edges(target, kind);
CREATE INDEX idx_edges_provenance ON edges(provenance);
```

### Files table

```sql
CREATE TABLE files (
    path TEXT PRIMARY KEY,                 -- relative path
    content_hash TEXT NOT NULL,            -- SHA256
    language TEXT NOT NULL,
    size INTEGER NOT NULL,
    modified_at INTEGER NOT NULL,
    indexed_at INTEGER NOT NULL,
    node_count INTEGER DEFAULT 0,
    errors TEXT                            -- JSON array
);

CREATE INDEX idx_files_language ON files(language);
CREATE INDEX idx_files_modified_at ON files(modified_at);
```

### Unresolved references table

```sql
CREATE TABLE unresolved_refs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    from_node_id TEXT NOT NULL,
    reference_name TEXT NOT NULL,
    reference_kind TEXT NOT NULL,          -- EdgeKind | 'function_ref'
    line INTEGER NOT NULL,
    col INTEGER NOT NULL,
    candidates TEXT,                       -- JSON array of possible qualified names
    file_path TEXT NOT NULL DEFAULT '',
    language TEXT NOT NULL DEFAULT 'unknown',
    FOREIGN KEY (from_node_id) REFERENCES nodes(id) ON DELETE CASCADE
);

CREATE INDEX idx_unresolved_from_node ON unresolved_refs(from_node_id);
CREATE INDEX idx_unresolved_name ON unresolved_refs(reference_name);
CREATE INDEX idx_unresolved_file_path ON unresolved_refs(file_path);
CREATE INDEX idx_unresolved_from_name ON unresolved_refs(from_node_id, reference_name);
```

### Project metadata table

```sql
CREATE TABLE project_metadata (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at INTEGER NOT NULL
);
```

Used for storing: `indexed_with_version`, `indexed_with_extraction_version`, etc.

### Full-text search (FTS5)

```sql
CREATE VIRTUAL TABLE nodes_fts USING fts5(
    id,
    name,
    qualified_name,
    docstring,
    signature,
    content='nodes',
    content_rowid='rowid'
);

-- Sync triggers
CREATE TRIGGER nodes_ai AFTER INSERT ON nodes BEGIN
    INSERT INTO nodes_fts(rowid, id, name, qualified_name, docstring, signature)
    VALUES (NEW.rowid, NEW.id, NEW.name, NEW.qualified_name, NEW.docstring, NEW.signature);
END;

CREATE TRIGGER nodes_ad AFTER DELETE ON nodes BEGIN
    INSERT INTO nodes_fts(nodes_fts, rowid, id, name, qualified_name, docstring, signature)
    VALUES ('delete', OLD.rowid, OLD.id, OLD.name, OLD.qualified_name, OLD.docstring, OLD.signature);
END;

CREATE TRIGGER nodes_au AFTER UPDATE ON nodes BEGIN
    INSERT INTO nodes_fts(nodes_fts, rowid, id, name, qualified_name, docstring, signature)
    VALUES ('delete', OLD.rowid, OLD.id, OLD.name, OLD.qualified_name, OLD.docstring, OLD.signature);
    INSERT INTO nodes_fts(rowid, id, name, qualified_name, docstring, signature)
    VALUES (NEW.rowid, NEW.id, NEW.name, NEW.qualified_name, NEW.docstring, NEW.signature);
END;
```

### FTS5 Search queries

```sql
-- Prefix matching with *:
SELECT id, name, qualified_name, file_path, language,
       rank FROM nodes_fts
WHERE nodes_fts MATCH ?          -- e.g. "calculate* total*"
ORDER BY rank
LIMIT ?

-- External content table join to get full node data:
SELECT n.*, fts.rank
FROM nodes_fts fts
JOIN nodes n ON n.id = fts.id
WHERE nodes_fts MATCH ?
ORDER BY fts.rank
LIMIT ?
```

## DatabaseConnection API

```typescript
class DatabaseConnection {
  static initialize(dbPath: string): DatabaseConnection   // create + migrate
  static open(dbPath: string): DatabaseConnection         // open existing
  close(): void
  getDb(): DatabaseConnection                             // returns underlying driver
  getBackend(): 'node-sqlite' | 'better-sqlite3' | 'sqlite3-wasm'
  getJournalMode(): string
  getSize(): number
  runMaintenance(): void                                  // ANALYZE + checkpoint WAL
  optimize(): void                                        // VACUUM + ANALYZE
}
```

## QueryBuilder — Key Queries

The QueryBuilder wraps prepared statements for all CRUD operations. Key queries:

```typescript
class QueryBuilder {
  // Node queries
  getNodeById(id: string): Node | null
  getNodesByIds(ids: string[]): Map<string, Node>
  getNodesByFile(filePath: string): Node[]
  getNodesByKind(kind: NodeKind): Node[]
  getNodesByName(name: string): Node[]                    // exact name match (ALL overloads)
  getNodesByLowerName(name: string): Node[]
  getNodesByQualifiedNameExact(qn: string): Node[]
  searchNodes(query: string, options?: SearchOptions): SearchResult[]  // FTS5
  getLastIndexedAt(): number | null
  getAllNodeNames(): string[]

  // Edge queries
  getOutgoingEdges(nodeId: string, kinds?: EdgeKind[]): Edge[]
  getIncomingEdges(nodeId: string, kinds?: EdgeKind[]): Edge[]
  insertEdges(edges: Edge[]): void

  // File queries
  getFileByPath(path: string): FileRecord | null
  getAllFiles(): FileRecord[]
  getAllFilePaths(): string[]
  upsertFile(file: FileRecord): void
  removeFile(path: string): void

  // Unresolved ref queries
  getUnresolvedReferences(): UnresolvedReference[]
  getUnresolvedReferencesBatch(offset: number, limit: number): UnresolvedReference[]
  getUnresolvedReferencesCount(): number
  getUnresolvedReferencesByFiles(filePaths: string[]): UnresolvedReference[]
  deleteSpecificResolvedReferences(keys: Array<{fromNodeId, referenceName, referenceKind}>): void
  deleteUnresolvedReferencesForFile(filePath: string): void

  // Statistics
  getStats(): GraphStats
  getNodeAndEdgeCount(): { nodes: number; edges: number }

  // Metadata
  setMetadata(key: string, value: string): void
  getMetadata(key: string): string | null

  // Management
  clear(): void   // DELETE ALL data
  insertNodes(nodes: Node[]): void
  insertUnresolvedReferences(refs: UnresolvedReference[]): void
}
```

## Migration pattern

```typescript
// schema_versions table tracks applied versions
interface Migration {
  version: number;
  description: string;
  up: (db: DatabaseConnection) => void;
}

// On DatabaseConnection.initialize():
// 1. Run schema.sql (idempotent CREATE TABLE IF NOT EXISTS)
// 2. Check schema_versions for highest applied version
// 3. Apply any unapplied migrations in order
// 4. Insert new version record
```
