# Nova: Binary Intermediate Representation (Binary IR)

> **Specification Version:** 3.0  
> **Status:** Specification & Format Standard  
> **File Extension:** `.novair`  
> **Magic Bytes:** `0x4E 0x4F 0x56 0x41` (`"NOVA"`)

---

## 1. Rationale: Why Binary IR?

Traditional web bundlers and transpilers rely on one of two paradigms:
1. **JavaScript as Intermediate Representation:** Tools like Babel and Vite transform JSX/TS into JavaScript strings. Later stages (Rollup, esbuild, Terser) parse that emitted JavaScript back into new ASTs. This introduces immense serialization/deserialization overhead and throws away semantic understanding.
2. **Abstract Syntax Trees (ASTs) in Memory:** Tools like Webpack preserve ASTs in memory. However, raw ASTs are extremely memory-inefficient (often consuming 10x-20x the size of the original source text) and lack cross-module semantic context.

**Nova's Solution: Custom Binary IR**
Nova synthesizes a canonical binary bytecode format that:
- **Preserves High-Level React Semantics:** Records which AST nodes represent React components, their hook dependencies, static render subtrees, and prop immutability.
- **Zero-Copy Deserialization:** Structured with 64-bit word alignment and offset tables, allowing the bundler and optimizer to read records directly from memory-mapped files without heap allocations.
- **Compact Footprint:** Uses string interning and variable-length integer encoding to achieve a memory footprint that is typically 40% smaller than raw source text and 85% smaller than standard JSON/AST representations.
- **Unified Pipeline Contract:** Serves as the single shared interface across compiler analysis, optimization passes, and chunk linking.

---

## 2. File Format & Binary Layout

A `.novair` binary file is divided into a **File Header**, followed by a **Section Header Table**, and six dedicated **Data Sections**:

```text
+-------------------------------------------------------+
| File Header (32 bytes)                                |
| - Magic: 0x4E 0x4F 0x56 0x41 ("NOVA")                 |
| - Version: Major (u16), Minor (u16)                   |
| - Checksum: BLAKE3 64-bit truncated hash              |
| - Flags: Target Arch, Optimization Level              |
| - Section Table Offset (u64) & Count (u16)            |
+-------------------------------------------------------+
| Section Header Table                                  |
| [Section ID | Offset | Length | Uncompressed Length]  |
+-------------------------------------------------------+
| Section 1: String Pool & Interned Symbol Table        |
+-------------------------------------------------------+
| Section 2: Module Scopes & Variable Bindings          |
+-------------------------------------------------------+
| Section 3: React Component Metadata & Hook Graph      |
+-------------------------------------------------------+
| Section 4: Canonical JSX Render Tree                  |
+-------------------------------------------------------+
| Section 5: Import, Export & Asset Dependency Records  |
+-------------------------------------------------------+
| Section 6: Optimization Annotations & Source Spans    |
+-------------------------------------------------------+
```

---

## 3. Section Specifications

### 3.1. File Header (32 Bytes)

```rust
#[repr(C, packed)]
pub struct NovaIrHeader {
    pub magic: [u8; 4],            // 0x4E, 0x4F, 0x56, 0x41 ("NOVA")
    pub version_major: u16,        // Current: 3
    pub version_minor: u16,        // Current: 0
    pub source_hash: [u8; 8],      // Truncated BLAKE3 hash of source file
    pub flags: u32,                // Bitflags (is_tsx, has_jsx, is_strict, etc.)
    pub section_count: u16,        // Total number of sections (typically 6)
    pub reserved: u16,             // Alignment padding
    pub section_table_offset: u64, // Byte offset to Section Header Table
}
```

### 3.2. Section 1: String Pool & Symbol Table
Contains all deduplicated strings (identifiers, string literals, property keys, import specifiers).
- Stores strings sequentially prefixed by length.
- Symbols are referenced throughout the IR as 32-bit `SymbolId` indices into this pool.

### 3.3. Section 2: Module Scopes & Variable Bindings
Encodes the lexical scope hierarchy:
- Scope depth and parent pointer.
- Declared identifiers and their mutability (`const`, `let`, `var`).
- References to external variables (free variables / closures).

### 3.4. Section 3: React Component Metadata
Records React-specific component semantics:

```rust
pub struct ComponentMetadata {
    pub component_symbol: SymbolId,
    pub flags: ComponentFlags,
    pub hook_count: u16,
    pub hooks: Vec<HookRecord>,
    pub prop_count: u16,
    pub props: Vec<PropRecord>,
    pub render_tree_root: NodeIndex,
}

bitflags::bitflags! {
    pub struct ComponentFlags: u32 {
        const IS_PURE             = 1 << 0; // No side effects in render
        const IS_STATIC           = 1 << 1; // Renders constant output, no props/state
        const CAN_AUTO_MEMO       = 1 << 2; // Safe to wrap in React.memo
        const CONTAINS_CLIENT_HOOK= 1 << 3; // Uses useState, useEffect, etc.
        const IS_ROOT_ENTRY       = 1 << 4; // Top-level application entry
        const HAS_DYNAMIC_CHILDREN= 1 << 5; // Children cannot be statically hoisted
    }
}

pub struct HookRecord {
    pub kind: HookKind,          // State, Effect, Memo, Callback, Ref, Custom
    pub dependency_indices: Vec<SymbolId>, // Variable bindings tracked as dependencies
    pub is_exhaustive: bool,      // Whether dependency array covers all closed variables
}
```

### 3.5. Section 4: Canonical JSX Render Tree
Encodes JSX elements as an optimized tree:
- **Node Kinds:**
  - `Element`: HTML element or React Component reference.
  - `StaticText`: Pre-evaluated string literal.
  - `Expression`: Dynamic JS expression.
  - `Fragment`: `<React.Fragment>` wrapper.
- **Static Subtree Marking:** Any subtree containing only static tags and literal props is tagged with `FLAG_HOISTABLE`, indicating the optimizer should extract it to module scope.

### 3.6. Section 5: Dependencies & Import/Export Records
Maintains exact dependency contracts:
- `StaticImport`: `import { Foo } from './bar'` (Source symbol, target symbol, specifier path).
- `DynamicImport`: `import('./lazy')` (Specifier path, promise resolution target).
- `ReExport`: `export { x } from './y'`.
- `AssetImport`: `import logo from './logo.svg'` (Asset type, format, bundling action).

### 3.7. Section 6: Optimization Annotations & Source Spans
- Stores optimization instructions computed by the compiler engine.
- Contains compressed source mapping spans (Original line, column, length) for every emitted IR instruction.

---

## 4. In-Memory Rust Representation (`nova-ir`)

In the `nova-ir` crate, the module is exposed via zero-allocation structs:

```rust
pub struct BinaryIrModule {
    pub header: NovaIrHeader,
    pub string_pool: Vec<String>,
    pub scopes: Vec<ScopeNode>,
    pub components: Vec<ComponentMetadata>,
    pub render_nodes: Vec<RenderNode>,
    pub dependencies: Vec<DependencyRecord>,
    pub optimizations: Vec<OptimizationAnnotation>,
    pub spans: Vec<SourceSpan>,
}

impl BinaryIrModule {
    /// Encodes this module into a zero-copy byte vector conforming to .novair spec
    pub fn encode(&self) -> Result<Vec<u8>, IrError>;

    /// Decodes a memory-mapped slice of bytes into a BinaryIrModule
    pub fn decode(bytes: &[u8]) -> Result<Self, IrError>;

    /// Verifies structural and semantic invariants of the IR
    pub fn validate(&self) -> Result<(), ValidationError>;
}
```

---

## 5. Benefits Over Existing Approaches

| Metric | Traditional Bundler (Raw AST / JS) | Nova Binary IR |
|---|---|---|
| **Memory Footprint** | ~15x Source File Size | **~0.6x Source File Size** |
| **Parse Latency** | 100% on every tool boundary | **0ms (Memory Mapped)** |
| **Component Semantics** | Lost during lowering | **Fully Preserved** |
| **Cache Serialization** | Slow JSON or custom AST marshal | **Direct Binary Dump** |
| **Verification** | Re-parse and lint | **O(1) Checksum + Schema Validation** |
