# Nova: Build Pipeline Working Flow

> **Specification Version:** 3.0  
> **Status:** Architecture Reference & Specification Document  
> **Scope:** Cold Build, Incremental Rebuild, and Development Server Lifecycles

---

## 1. Overview

Nova's execution flow is strictly deterministic and divided into discrete, measurable phases. By decoupling source analysis, semantic lowering, optimization, and code generation into distinct stages connected by **Binary IR**, Nova ensures that any stage can be bypassed or accelerated via cached artifacts.

---

## 2. Cold Build Execution Flow

During a cold build (when no valid caches exist), Nova executes an 8-stage pipeline from source ingestion to production emission.

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer / CI
    participant CLI as nova-cli
    participant Cache as nova-cache
    participant Parser as nova-parser
    participant Compiler as nova-compiler
    participant Graph as nova-graph
    participant Opt as nova-optimizer
    participant Bundler as nova-bundler
    participant Disk as File System / Dist

    Dev->>CLI: nova build --mode=production
    CLI->>Cache: Initialize Cache Store & Compute Env Fingerprint
    CLI->>Parser: Discover & Scan Workspace Files (SIMD)
    
    par Parallel Parsing
        Parser->>Parser: Lex & Parse TSX/JS into ASTs
        Parser->>Cache: Persist Valid AST Blobs
    end

    Parser->>Compiler: Handover Parsed ASTs
    
    par Parallel Semantic Analysis
        Compiler->>Compiler: Resolve Scopes & Symbols
        Compiler->>Compiler: Synthesize Binary IR (.novair)
        Compiler->>Cache: Persist Binary IR Blobs
    end

    Compiler->>Graph: Ingest Module Edges & Asset References
    Graph->>Graph: Construct DAG & Resolve Circular References
    Graph->>Cache: Persist Dependency Graph Topology

    Graph->>Opt: Transmit IR + Topology to Optimizer
    Opt->>Opt: Run Compiler Passes (DCE, Constant Folding)
    Opt->>Opt: Run Safety Checks & Application Passes (Memo, Hoisting)
    
    Opt->>Bundler: Stream Optimized Binary IR Chunks
    Bundler->>Bundler: Partition Chunks (Entry, Async, Vendors)
    Bundler->>Bundler: Generate JS Code, Source Maps & Scoped CSS
    Bundler->>Disk: Write Minified Production Assets into /dist
    Bundler->>CLI: Return Telemetry & Benchmark Metrics
    CLI->>Dev: Build Completed (Duration, Cache Stats, Bundle Summary)
```

### Detailed Phase Breakdown

1. **Environment Initialization:**
   - The CLI parses command-line arguments and loads `nova.config.ts` or `nova.config.json`.
   - Generates a **Master Environment Fingerprint** $\mathcal{H}_{\text{env}} = \text{BLAKE3}(\text{NodeVersion} \parallel \text{NovaVersion} \parallel \text{ConfigContent})$.

2. **File Discovery & SIMD Lexing:**
   - The scanner discovers all `.tsx`, `.ts`, `.jsx`, `.js`, and `.css` files.
   - For each file, compute $\mathcal{H}_{\text{source}} = \text{BLAKE3}(\text{RawFileBytes})$.
   - If present in the AST cache, retrieve the cached AST; otherwise, tokenize via SIMD lexer.

3. **Parallel Parsing:**
   - Rayon worker threads convert token streams into ASTs with exact source span indices.
   - Syntax errors abort processing with formatted diagnostic diagnostics.

4. **Semantic Analysis & Binary IR Generation:**
   - Build lexical scope trees, track variable bindings, and resolve imports against the workspace module resolver.
   - Identify React component definitions, analyzing props, hooks, and static JSX subtrees.
   - Encode data into the binary `.novair` format.

5. **Dependency Graph Construction:**
   - For each module, record:
     - Direct imports (`import { a } from './b'`)
     - Dynamic imports (`import('./c')`)
     - Type imports (`import type { T } from './d'`)
     - Asset references (`import img from './logo.svg'`)
   - Compute strongly connected components (SCCs) to detect and manage circular dependencies.

6. **Two-Tier Optimization:**
   - **Tier 1 (Compiler):** Local module constant folding, DCE, and symbol mangling.
   - **Tier 2 (Application):** Cross-module render hoisting, component memoization, and route splitting.

7. **Chunking & Linker:**
   - Traverse the graph starting from entry points.
   - Split chunks based on dynamic import boundaries and shared vendor dependencies.
   - Synthesize the runtime loader code.

8. **Asset & Code Emission:**
   - Generate production JavaScript, CSS stylesheets, and source maps (`.map`).
   - Write files to the output directory with content-addressed file hashes (e.g., `main.7a8b9c.js`).

---

## 3. Incremental Rebuild Flow (Watch Mode)

When a developer modifies a source file during active development, Nova executes a surgical rebuild avoiding full recompilation.

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer
    participant Watcher as File Watcher
    participant Cache as nova-cache
    participant Parser as nova-parser
    participant Compiler as nova-compiler
    participant Graph as nova-graph
    participant Opt as nova-optimizer
    participant Bundler as nova-bundler
    participant HMR as Browser HMR Client

    Dev->>Watcher: Modifies src/components/Counter.tsx
    Watcher->>Cache: Check New Content Hash vs Cache
    Note over Cache: Content Changed (Cache Miss)
    
    Watcher->>Parser: Parse Counter.tsx into AST
    Parser->>Compiler: Semantic Analysis of Counter.tsx
    Compiler->>Compiler: Synthesize New Binary IR
    
    Compiler->>Graph: Reconcile Dependency Edges
    Graph->>Graph: Perform Reverse Topological Walk
    Note over Graph: Compute Affected Set: [Counter.tsx, App.tsx]
    
    Graph->>Opt: Re-optimize Affected Binary IR Modules
    Opt->>Bundler: Invalidate Only Affected Chunks
    Bundler->>Bundler: Re-link app.chunk.js
    Bundler->>HMR: Dispatch Binary HMR Update Payload
    HMR->>Dev: Fast Hot Module Replacement in Browser
```

### Invalidation & Impact Analysis Mechanics

1. **File Watcher Event:** The file system watcher triggers on `Counter.tsx` modification.
2. **Fingerprint Comparison:** Nova hashes the new file content:
   $$\mathcal{H}_{\text{new}} \neq \mathcal{H}_{\text{cached}}$$
3. **Targeted Parse & Lowering:** Only `Counter.tsx` is re-parsed and lowered into Binary IR.
4. **Graph Reconciliation:**
   - If the imports of `Counter.tsx` did not change, the graph topology remains constant.
   - If an import was added or removed, update the graph edge index.
5. **Reverse Invalidation Propagation:**
   - Nova looks up the **Reverse Dependent Index**:
     $$\mathcal{R}(\text{Counter.tsx}) = \{ \text{App.tsx} \}$$
   - Only `Counter.tsx` and its direct consumer `App.tsx` are scheduled for re-optimization.
   - Unaffected modules (e.g., `Navbar.tsx`, `Footer.tsx`, `utils.ts`, `node_modules`) are **never re-analyzed**.
6. **Chunk Re-linking:**
   - Only the chunk containing the affected components is re-emitted.
   - Emits an incremental Hot Module Replacement (HMR) update directly to the connected browser client in < 15ms.

---

## 4. Telemetry and Performance Checkpoints

Every build step records telemetry metrics:

| Checkpoint | Metrics Captured |
|---|---|
| `scan_phase` | Discovered file count, total source bytes, I/O scan time |
| `parse_phase` | AST construction duration, tokens per second, cache hit ratio |
| `semantic_phase` | Symbols resolved, Binary IR serialization time, IR compression ratio |
| `graph_phase` | Node count, edge count, SCC count, topological sort duration |
| `opt_phase` | Passes executed, AST nodes folded, components memoized, render subtrees hoisted |
| `bundle_phase` | Chunks generated, JS bytes emitted, CSS bytes extracted, source map synthesis time |
| `total_pipeline` | Wall-clock elapsed time, peak memory usage (RSS) |
