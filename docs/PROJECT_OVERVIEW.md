# Nova: Project Overview & Vision

> **Project:** Nova — A React-Compatible High-Performance Build Platform  
> **Status:** Active Research & Prototype Development  
> **Institution:** SRM Institute of Science and Technology, Tiruchirappalli  
> **Department:** Information Technology  
> **Motto:** *Keep React. Rebuild the engine beneath it.*

---

## 1. Executive Summary

Modern web frontend development relies heavily on the **React** component model. However, the toolchains responsible for compiling, analyzing, optimizing, and bundling React applications (such as Vite, Webpack, Rollup, and esbuild) operate as disconnected, multi-stage pipelines. In these conventional systems:

1. **Source Code is Repeatedly Parsed:** Source files are parsed into Abstract Syntax Trees (ASTs) by compilers (e.g., Babel, SWC, TypeScript), transformed into intermediate JavaScript, and written out.
2. **Semantic Knowledge is Lost:** Bundlers (e.g., Webpack, Rollup, Vite) re-read and re-parse the intermediate JavaScript files, attempting to reconstruct module relationships, side-effects, and tree-shaking boundaries from scratch.
3. **Optimizations are Superficial:** Bundlers treat JavaScript as generic control flow, performing code-level dead code elimination without understanding React-specific semantics (such as pure render trees, stable props, hook dependencies, or static subtrees).
4. **Developer Workflows are Fragmented:** Developers must configure and coordinate disparate tools (transpilers, linters, module bundlers, CSS extractors, minifiers) to produce production-grade output.

**Nova** is designed to solve these systemic inefficiencies through a **compiler-driven, unified architecture**. Nova replaces the fragmented toolchain with an integrated Rust engine centered around a shared **Binary Intermediate Representation (Binary IR)**. By analyzing application code once, preserving semantic metadata throughout the pipeline, and driving optimizations through a single source of truth, Nova aims to dramatically improve build performance while lowering browser memory and CPU consumption at runtime.

---

## 2. Problem Statement

Modern web development infrastructure faces fundamental structural bottlenecks:

```text
[Current Build Pipeline]
React/TS ──> Lexer/Parser ──> AST ──> [Compiler Lowering] ──> JS Output (Temp)
                                                                    │
      ┌─────────────────────────────────────────────────────────────┘
      ▼
Bundler Ingestion ──> Re-Lex/Re-Parse ──> Bundler AST ──> Graph Analysis ──> Bundle JS
```

### 2.1. Redundant Analysis & Lost Semantics
When a compiler lowers JSX/TSX into JavaScript function calls (`React.createElement` or `_jsx`), it discards the high-level semantic context of the component:
- Which components are purely static and never re-render.
- Which props are compile-time constants.
- The precise dependency arrays of React hooks (`useMemo`, `useCallback`, `useEffect`).
- The explicit boundaries of component state ownership.

When the bundler reads the resulting JavaScript, it must guess whether function invocations have side effects, leading to conservative bundling, missed dead-code elimination, and bloated runtime bundles.

### 2.2. Inefficient Incremental Rebuilds
In modern rapid-iteration development, changing a single file often triggers expensive AST invalidation and module graph traversal. Without a unified Binary IR, caches either store raw text (requiring re-parsing) or bloated ASTs that do not share cross-module symbol metadata.

### 2.3. Runtime Overhead in the Browser
Because existing bundlers only optimize code syntax rather than the **application model**, browsers execute excessive JavaScript to mount unchanged virtual DOM nodes, re-evaluate inline objects and closures, and reconcile components that could have been pre-computed or memoized at compile time.

---

## 3. Project Objectives

Nova is engineered around five fundamental research and development objectives:

1. **Preserve React Compatibility Without New Syntax:**
   Enable existing React applications written in JavaScript, TypeScript, JSX, and TSX to run on Nova without modifying a single line of component code. Nova does not introduce proprietary template syntax or custom DSLs.

2. **Design and Implement Binary IR:**
   Create a compact, versioned, zero-copy binary intermediate representation that encapsulates module symbols, JSX render subtrees, hook bindings, asset references, and source spans in a single canonical format.

3. **Construct a Bidirectional Dependency Graph:**
   Build a high-performance graph engine tracking forward imports and reverse dependents to enable instant topological impact analysis and surgical cache invalidation.

4. **Develop the Binary-IR-Guided Incremental Optimization Algorithm:**
   Formulate and implement a 7-stage algorithm that leverages content fingerprinting, stage-specific caching, and safety-verified compiler/application optimization passes.

5. **Empirically Validate Performance Against Industry Baselines:**
   Benchmark Nova against industry standards (Vite, Webpack, esbuild) across cold build latency, incremental rebuild latency, memory footprint, production bundle size, and browser runtime performance.

---

## 4. Literature & Industry Review

| Platform / Tool | Architectural Focus | Limitations in Modern React Pipelines | Nova Research Differentiation |
|---|---|---|---|
| **Vite** | Unbundled dev server using native ESM; Rollup-based production builds. | Dev and prod use different bundlers (esbuild vs Rollup); semantic analysis is discarded between tools. | Single unified Rust engine for both dev and prod; preserves Binary IR across all stages. |
| **Webpack** | Highly extensible module graph with rich plugin ecosystem. | High configuration overhead; slow cold starts; multiple AST transforms per file; high memory consumption. | Built-in zero-copy Binary IR; SIMD parallel lexing; native Rust concurrency (Rayon). |
| **esbuild** | Extremely fast Go-based transpiler and bundler. | Primarily performs syntax-level minification and bundling; does not perform deep React-specific application optimizations. | Deep React semantic analysis, static render tree hoisting, and automated safety-checked memoization. |
| **LLVM** | Industry-standard compiler infrastructure with strongly typed IR. | Designed for systems languages (C/C++, Rust, Swift); not built for dynamic ECMAScript or virtual DOM frameworks. | Adapts LLVM's multi-pass IR philosophy to React web architecture. |

---

## 5. Scope and Boundaries

### In Scope
- Parsing and compilation of ES2015+ JavaScript, TypeScript 5+, JSX, and TSX.
- Synthesis, encoding, decoding, and validation of Binary IR.
- Directed acyclic module graphs with circular dependency support.
- Compiler-level optimizations (DCE, constant folding, symbol minimization).
- Application-level optimizations (render tree hoisting, component auto-memoization, static boundary isolation).
- Production bundling with chunk splitting, CSS extraction, asset emission, and source maps.
- Content-addressable persistent caching.

### Out of Scope (Current Milestone)
- Replacement of the React runtime library itself (Nova works with `react` and `react-dom`).
- Browser-native WebGPU rendering engine (deferred to future milestones).
- Non-standard JSX extensions or experimental JS syntax proposals prior to Stage 3.

---

## 6. Research Team & Academic Governance

This project is conducted as an active engineering and research initiative at **SRM Institute of Science and Technology, Tiruchirappalli**, Department of Information Technology.

### Core Investigators
- **Kirutiga G** — Register No: `RA2311008050003`
- **Dinesh Kanna R** — Register No: `RA23110080500010`
- **Nithish M** — Register No: `RA23110080500033`

### Project Supervisor
- **Dr. Jubilant Kizhakkethottam**  
  Head of the Department, Department of Information Technology  
  SRM Institute of Science and Technology, Tiruchirappalli
