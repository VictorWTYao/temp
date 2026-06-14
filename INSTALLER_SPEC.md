# CodeGraph Installer Specification

## 1. Overview

The installer configures opencode to use CodeGraph's custom tools. It places tool files in `.opencode/tools/` and sets up the project for code graph indexing.

## 2. Installation Steps

```
1. Ensure the project root
2. Create `.opencode/tools/` directory if it doesn't exist
3. Copy the following tool files into `.opencode/tools/`:
   - explore.ts
   - node.ts
   - search.ts
   - status.ts
   - callers.ts
   - callees.ts
   - impact.ts
4. Create a `.codegraph/` directory for the SQLite database
5. Run `codegraph init` (or initialize via API)
6. Optionally run `codegraph index` for initial full-index
```

## 3. Install CLI

```typescript
// bin/codegraph.ts — install subcommand

program
  .command('install')
  .description('Install CodeGraph for the current project')
  .option('--index', 'Run initial indexing after install')
  .action(async (options) => {
    const root = process.cwd()

    // 1. Create .opencode/tools/ directory
    const toolsDir = path.join(root, '.opencode', 'tools')
    fs.mkdirSync(toolsDir, { recursive: true })

    // 2. Copy tool definition files from installed package
    const toolFiles = ['explore.ts', 'node.ts', 'search.ts', 'status.ts', 'callers.ts', 'callees.ts', 'impact.ts']
    for (const file of toolFiles) {
      const src = path.join(__dirname, '..', 'installer', 'tools', file)
      const dest = path.join(toolsDir, file)
      if (fs.existsSync(dest)) {
        // Check if existing matches — skip if unchanged
        if (fs.readFileSync(src, 'utf-8') === fs.readFileSync(dest, 'utf-8')) continue
      }
      fs.copyFileSync(src, dest)
    }

    // 3. Initialize codegraph project
    const cg = await CodeGraph.init(root)

    // 4. Index if requested
    if (options.index) {
      await cg.indexAll({ onProgress: showProgress })
    }

    console.log('CodeGraph installed.')
    console.log(`Tools: .opencode/tools/ (${toolFiles.length} files)`)
    console.log(`Database: .codegraph/`)
    if (!options.index) {
      console.log('Run `codegraph index` to build the code graph.')
    }
  })
```

## 4. Uninstall CLI

```typescript
program
  .command('uninstall')
  .description('Remove CodeGraph from the current project')
  .action(async () => {
    const root = process.cwd()

    // 1. Remove .opencode/tools/ tool files
    const toolFiles = ['explore.ts', 'node.ts', 'search.ts', 'status.ts', 'callers.ts', 'callees.ts', 'impact.ts']
    for (const file of toolFiles) {
      const dest = path.join(root, '.opencode', 'tools', file)
      if (fs.existsSync(dest)) fs.unlinkSync(dest)
    }

    // 2. Remove .codegraph/ database directory
    const cgDir = path.join(root, '.codegraph')
    if (fs.existsSync(cgDir)) {
      fs.rmSync(cgDir, { recursive: true, force: true })
    }

    console.log('CodeGraph uninstalled.')
  })
```

## 5. Tool file layout

The tool definition files shipped with the package live at:

```
src/installer/tools/
├── explore.ts
├── node.ts
├── search.ts
├── status.ts
├── callers.ts
├── callees.ts
└── impact.ts
```

The `install` command copies them to `.opencode/tools/`. The tool files import CodeGraph from `../../lib/codegraph` or similar relative path within the project. Alternatively, tools can invoke the CLI via shell.

### invokeCodeGraph helper

For a simpler approach (tools invoke the CLI rather than requiring the lib path):

```typescript
// Shared utility in each tool file
async function invokeCodeGraph(args: string[], worktree: string): Promise<string> {
  const { execSync } = require('child_process')
  try {
    const output = execSync(`npx -y @colbymchenry/codegraph ${args.join(' ')} --path "${worktree}"`, {
      encoding: 'utf-8',
      timeout: 30000,
    })
    return output.trim()
  } catch (err) {
    return `Error: ${err.stderr || err.message}`
  }
}
```

Using the CLI approach means tools don't need to know the import path of the CodeGraph library — they just shell out to the `codegraph` binary.

## 6. Self-heal on re-install

Idempotent: running `install` again should:
1. Update any tool files that changed (overwrite)
2. NOT re-index if already indexed (detect by checking `.codegraph/` directory)
3. Print a summary of what was updated vs unchanged

## 7. OpenCode config

After install, the user has:
```
<project>/
├── .opencode/
│   └── tools/
│       ├── explore.ts
│       ├── node.ts
│       ├── search.ts
│       ├── status.ts
│       ├── callers.ts
│       ├── callees.ts
│       └── impact.ts
└── .codegraph/
    └── codegraph.db
```

The tools are auto-discovered by opencode — no config file changes needed.
