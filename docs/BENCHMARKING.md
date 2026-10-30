# Nova: Benchmarking Suite & Performance Evaluation

> **Specification Version:** 3.0  
> **Status:** Architecture Reference & Specification Document  
> **Evaluation Baselines:** Vite 5+, Webpack 5+, esbuild 0.20+

---

## 1. Benchmarking Objectives & Methodology

The primary hypothesis of the Nova research project is:

> *A unified, compiler-driven build platform utilizing a shared Binary Intermediate Representation (Binary IR) can eliminate redundant analysis, accelerate incremental rebuilds, and yield smaller, faster-executing React applications compared to traditional disjoint toolchains.*

To validate or refute this hypothesis, Nova establishes a rigorous, reproducible benchmarking harness evaluating both **Build-Time Performance** and **Browser Runtime Performance**.

---

## 2. Evaluation Metrics

### 2.1. Build-Time Metrics
1. **Cold Build Duration (ms):** Total wall-clock time required to compile the entire project from scratch with no pre-existing cache.
2. **Incremental Rebuild Duration (ms):** Time required to detect a leaf-node component change, re-analyze, re-link, and emit updated bundles.
3. **Cache Hit Rate (%):** Percentage of module queries satisfied by L1 memory or L2 disk cache.
4. **Peak Memory Consumption (RSS in MB):** Peak Resident Set Size utilized by the compiler/bundler process during a full build.

### 2.2. Output & Runtime Metrics
1. **Raw Bundle Size (KB):** Total uncompressed byte size of generated JavaScript and CSS.
2. **Transfer Size (KB):** Brotli-compressed and Gzip-compressed byte size.
3. **Browser Parse & Compile Time (ms):** Time spent by the browser V8 engine parsing and compiling initial scripts.
4. **First Contentful Paint (FCP) & Time to Interactive (TTI):** Measured via headless Chrome / Lighthouse.
5. **Component Re-Render Latency (ms):** Frame time during rapid state updates, evaluating the impact of compile-time render tree hoisting and automatic memoization.

---

## 3. Testbed Project Tiers

To evaluate scalability across project sizes, benchmarks are conducted across three standardized testbed fixtures:

| Testbed Tier | Component Count | Total Modules | Dependency Depth | Route Count |
|---|---|---|---|---|
| **Tier 1: Small SPA** | 50 Components | ~120 Modules | 4 Levels | 5 Routes |
| **Tier 2: Medium App** | 500 Components | ~1,200 Modules | 8 Levels | 25 Routes |
| **Tier 3: Enterprise App**| 5,000 Components | ~12,500 Modules | 16 Levels | 100 Routes |

---

## 4. Projected Baseline Comparisons

Target performance baselines established for the Nova evaluation milestone:

### 4.1. Cold Build Latency (Medium App: 500 Components)

```text
Webpack 5:   ████████████████████████████████████ 14,200 ms
Vite 5:      ██████████ 3,850 ms
esbuild:     ███ 1,120 ms
Nova (Goal): ██ 780 ms
```

### 4.2. Incremental Rebuild Latency (Leaf Component Modification)

```text
Webpack 5:   ██████████████ 1,250 ms
Vite 5:      ████ 180 ms
esbuild:     ██ 45 ms
Nova (Goal): █ < 15 ms  (Binary IR Cache + Reverse Reachability Set)
```

### 4.3. Production Bundle Size (Brotli Compressed)

```text
Webpack 5:   ████████████████ 248 KB
Vite 5:      ██████████████ 232 KB
esbuild:     ███████████████ 240 KB
Nova (Goal): ███████████ 185 KB  (Render Hoisting, Auto-Memo, Static DCE)
```

---

## 5. Comparative Evaluation Summary

| Evaluation Dimension | Webpack 5 | Vite 5 | esbuild | Nova Target |
|---|---|---|---|---|
| **Intermediate Representation** | None (JS strings/AST) | None (JS strings/AST) | Custom AST in Go | **Zero-Copy Binary IR** |
| **Semantic Preservation** | ❌ Lost during lowering | ❌ Lost during lowering | ❌ Partial | **✅ 100% Preserved** |
| **React-Aware Optimization** | ❌ None (Generic JS) | ❌ None (Generic JS) | ❌ None (Generic JS) | **✅ Deep Component Hoisting** |
| **Incremental Invalidation** | Graph Hash Traversal | Module ID Invalidation | In-memory AST rebuild | **Reverse Reachability Boundary** |
| **Cache Format** | JSON / Memory FileSystem | Vite Cache / Rollup | In-Memory Only | **Content-Addressable BLAKE3 IR** |

---

## 6. Running Benchmark Suites

To run the automated benchmark harness:

```bash
# Execute cold and incremental rebuild benchmarks across all tiers
./benchmarks/scripts/run_benchmarks.sh --tier=medium --runs=10
```
