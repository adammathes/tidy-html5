# Implementation Plan

## Overview

Incremental Rust rewrite of HTML Tidy, module by module, using the C
binary as an oracle for correctness. Each phase is independent and
validated before the next begins.

## Current Status

**Active Phase**: None (project scaffolding)
**Last Updated**: 2026-02-25

## Phase Dependency Graph

```
Phase 0: Build Infrastructure
    │
    ▼
Phase 1: entities.c ──────────────────────────────┐
    │                                               │
    ▼                                               │
Phase 2: tmbstr.c                                  │
    │                                               │
    ▼                                               │
Phase 3: utf8.c                                    │
    │                                               │  All of these are
    ▼                                               │  "leaf" modules with
Phase 4: attrdict.c + tags.c ◄── DECISION POINT   │  no tree dependency
    │         (tags.c has function pointers         │
    │          into parser — may need stubs)        │
    │                                               │
    ▼                                               │
Phase 5: config.c + message.c + language.c         │
    │                                               │
    ▼                                               │
Phase 6: buffio.c + fileio.c + streamio.c ─────────┘
    │
    ▼
    ════════════════════════════════════════
    TREE WALL — everything below reads/writes the Node tree
    Must decide on Rust tree representation first
    ════════════════════════════════════════
    │
    ▼
Phase 7: access.c + attrs.c (tree readers)
    │
    ▼
Phase 8: pprint.c (tree reader → output)
    │
    ▼
Phase 9: clean.c (tree mutator)
    │
    ▼
Phase 10: lexer.c + parser.c (tree creator) ◄── THE BOSS FIGHT
```

---

## Phase 0: Build Infrastructure

**Status**: Not started
**Goal**: Prove that Rust code can be compiled and linked into the tidy
build, and that a trivial FFI call works.

### Tasks

- [ ] Set up `reference/` directory with tidy-html5 C snapshot
- [ ] Build reference C binary with CMake, verify it works
- [ ] Run existing 402 regression tests against reference binary
- [ ] Initialize `rust/` crate with `Cargo.toml`
- [ ] Initialize `ffi/` crate with `Cargo.toml`
- [ ] Create a trivial Rust function (`rust_tidy_probe() -> i32`)
- [ ] Export it via FFI crate with `#[no_mangle] extern "C"`
- [ ] Write a C test program that calls `rust_tidy_probe()`, links, runs
- [ ] Verify this works on Linux (primary dev platform)
- [ ] Document the build process in README.md

### Definition of Done

A C program calls a Rust function through FFI, the build completes on
Linux, and the result is correct.

### Notes for Agents

This phase is mostly build system work. Read `reference/CMakeLists.txt`
to understand how the C build works. The Rust crate should produce a
static library (`.a`) that can be linked alongside C object files.

Consider using `cc` crate in `build.rs` for compiling C code alongside
Rust, or keeping CMake for C and linking the outputs. The simpler approach
wins — don't over-engineer the build.

---

## Phase 1: entities.c

**Status**: Not started
**Goal**: Replace the HTML entity lookup table and its 2 functions with Rust.

### Context

`entities.c` contains:
- A 2,032-entry static lookup table: `{name, version_flags, codepoint}`
- `EntityInfo(name, isXml, &code, &versions)` — look up entity by name
  (including numeric `&#N;` and `&#xN;` forms)
- `EntityName(code, versions)` — reverse lookup, codepoint to name
- Binary search by first character, then linear scan within that letter

### Call Sites (3 total)

```
lexer.c:1251  — EntityInfo() during entity parsing in the lexer
pprint.c:1270 — EntityName() during output serialization
pprint.c:1324 — EntityName() during output serialization
```

### Dependencies

- `tmbstrcmp()` from tmbstr.c — BUT we can use Rust's native string
  comparison instead, since we're replacing the module entirely
- No allocator interaction (all data is static/const)
- No document or node state accessed

### Tasks

- [ ] Extract full entity table from C source (all 2,032 entries)
- [ ] Download html5lib-tests `entities.json` for cross-validation
- [ ] Write Rust `entities.rs` with the lookup table
  - Consider `phf` (perfect hash function) crate for O(1) lookup
  - OR a sorted array with binary search (matching C behavior)
- [ ] Implement `entity_info(name, is_xml) -> Option<(u32, u32)>`
  - Must handle `&name;` named entities
  - Must handle `&#decimal;` numeric entities
  - Must handle `&#xhex;` hex entities
  - Case sensitivity: named entities are case-sensitive
- [ ] Implement `entity_name(code, versions) -> Option<&str>`
  - Returns first matching name for given codepoint and version mask
- [ ] Write FFI wrappers in `ffi/src/lib.rs`:
  ```rust
  #[no_mangle]
  pub unsafe extern "C" fn prvTidyEntityInfo(
      name: *const c_char,
      isXml: Bool,
      code: *mut c_uint,
      versions: *mut c_uint,
  ) -> Bool
  ```
- [ ] Write comprehensive tests:
  - All 2,032 named entities (oracle-validated)
  - Numeric entities: `&#0;` through `&#1114111;` (sampling)
  - Hex entities: `&#x0;` through `&#x10FFFF;` (sampling)
  - Invalid entities: empty name, unknown name, malformed numeric
  - html5lib entity tests
- [ ] Replace `entities.c` in the build with Rust FFI library
- [ ] Run full 402-case regression suite
- [ ] Document any behavioral divergences

### Definition of Done

All 402 regression tests pass with the Rust entity module linked in
place of `entities.c`. All entity-specific unit tests pass.

---

## Phase 2: tmbstr.c

**Status**: Not started
**Goal**: Replace 17 string utility functions with Rust implementations.

### Context

`tmbstr.c` has 17 functions. Called from **19 of 26 source files**.

### Critical Complexity: The Allocator

Two functions allocate memory through `TidyAllocator*`:
- `tmbstrdup(allocator, str)` — duplicate string
- `tmbstrndup(allocator, str, len)` — duplicate N bytes

The Rust implementations MUST call through the same allocator vtable.
This requires `unsafe` code that dereferences C function pointers:

```rust
pub unsafe fn tmbstrdup(alloc: *mut TidyAllocator, s: *const c_char) -> *mut c_char {
    let len = libc::strlen(s);
    let ptr = ((*(*alloc).vtbl).alloc)(alloc, len + 1);
    // ... copy bytes ...
    ptr as *mut c_char
}
```

### Critical Complexity: ToLower/ToUpper

`tmbstrcasecmp` and `tmbstrtolower/upper` use `TY_(ToLower)` which is
a macro in `lexer.h` that indexes a static byte-type lookup table. The
Rust code must reproduce this exact mapping (ASCII-only, not Unicode
case folding).

### Functions to Implement

| Function | Allocates? | Notes |
|----------|-----------|-------|
| `tmbstrdup` | YES | via TidyAllocator |
| `tmbstrndup` | YES | via TidyAllocator |
| `tmbstrncpy` | no | copy with size limit |
| `tmbstrcpy` | no | copy, returns count |
| `tmbstrcat` | no | concatenate, returns count |
| `tmbstrcmp` | no | exact compare |
| `tmbstrlen` | no | byte length |
| `tmbstrcasecmp` | no | uses ToLower lookup |
| `tmbstrncmp` | no | compare N bytes |
| `tmbstrncasecmp` | no | compare N bytes, case-insensitive |
| `tmbsubstrn` | no | find substring (exact) |
| `tmbsubstr` | no | find substring (case-insensitive) |
| `tmbstrtolower` | no | in-place ASCII lower |
| `tmbstrtoupper` | no | in-place ASCII upper |
| `tmbvsnprintf` | no | wraps vsnprintf |
| `tmbsnprintf` | no | wraps snprintf |
| `strrep` | no | replace substring in buffer (HAS BUFFER OVERFLOW BUG) |

### Tasks

- [ ] Define `TidyAllocator` and `TidyAllocatorVtbl` repr(C) structs in Rust
- [ ] Reproduce the `ToLower`/`ToUpper` lookup table from `lexer.h`
- [ ] Implement all 17 functions
- [ ] For `strrep`: make it safe (bounds-check) but produce identical
  output for inputs that don't trigger the C buffer overflow
- [ ] For `tmbvsnprintf`/`tmbsnprintf`: just call libc `vsnprintf`/`snprintf`
  (no benefit to reimplementing printf in Rust)
- [ ] Write FFI wrappers with `prvTidy` prefix
- [ ] Generate oracle tests for every function with diverse inputs
- [ ] Replace `tmbstr.c` in build, run full regression suite

### Definition of Done

All 402 regression tests pass. All 17 functions tested with oracle
validation. The `strrep` buffer overflow is fixed (no longer crashes
on oversized inputs).

---

## Phase 3: utf8.c

**Status**: Not started
**Goal**: Replace UTF-8/UTF-16 encode/decode functions.

### Critical Complexity: I/O Callbacks

`DecodeUTF8BytesToChar` accepts either:
- A `ctmbstr` buffer of successor bytes, OR
- A `TidyInputSource*` callback that reads bytes one at a time

The Rust code must handle both paths, including calling through C
function pointers for the callback path.

Similarly, `EncodeCharToUTF8Bytes` writes either to a buffer or through
a `TidyOutputSink*` callback.

### Functions to Implement

| Function | I/O Callbacks? | Notes |
|----------|---------------|-------|
| `DecodeUTF8BytesToChar` | YES | Read from buffer or TidyInputSource |
| `EncodeCharToUTF8Bytes` | YES | Write to buffer or TidyOutputSink |
| `GetUTF8` | no | Convenience: decode from string |
| `PutUTF8` | no | Convenience: encode to buffer |
| `IsValidUTF16FromUCS4` | no | Pure arithmetic |
| `IsHighSurrogate` | no | Range check |
| `IsLowSurrogate` | no | Range check |
| `CombineSurrogatePair` | no | Arithmetic |
| `SplitSurrogatePair` | no | Arithmetic |
| `IsValidCombinedChar` | no | Range check |
| `IsCombinedChar` | no | Range check |

### Tasks

- [ ] Define `TidyInputSource` and `TidyOutputSink` repr(C) structs
- [ ] Implement decode/encode with both buffer and callback paths
- [ ] Reproduce the exact overlong-sequence detection from C
- [ ] Test with all valid codepoints (0x0000-0x10FFFF)
- [ ] Test with Markus Kuhn's UTF-8 stress test inputs
- [ ] Test invalid/overlong sequences
- [ ] Replace `utf8.c` in build, run regression suite

---

## Phase 4: attrdict.c + tags.c

**Status**: Not started
**Goal**: Replace HTML attribute and tag dictionaries.

### WARNING: Function Pointers in Tag Table

The tag dictionary contains function pointers to parser functions:

```c
{ TidyTag_A, "a", VERS_ALL, ..., TY_(ParseInline), CheckA }
```

During the transition (while parser.c is still C), these function
pointers must reference the C parser functions. This means the Rust
tag table must store `extern "C"` function pointers.

### DECISION NEEDED BEFORE STARTING

How to handle parser function pointers in the tag table when the
parser is still in C. Options:
1. Store raw `extern "C" fn` pointers in Rust structs
2. Keep the tag table in C and only rewrite the lookup functions
3. Use a mapping from tag ID to function pointer, kept in C

Recommend option 1 for now, converting to native Rust functions in
Phase 10.

---

## Phase 5: config.c + message.c + language.c

**Status**: Not started
**Goal**: Replace configuration parsing, message formatting, localization.

Large phase (~12,000 LOC total). May split into sub-phases:
- 5a: language.c + language_en.h (string tables, low coupling)
- 5b: message.c + messageobj.c (message formatting)
- 5c: config.c (option parsing, high coupling to TidyDoc)

---

## Phase 6: buffio.c + fileio.c + streamio.c

**Status**: Not started
**Goal**: Replace I/O stream abstractions.

These implement the `TidyInputSource` and `TidyOutputSink` interfaces
used by the lexer and pretty printer.

---

## ═══ TREE WALL ═══

Everything below this line reads or writes the `Node` tree structure.

### DECISION REQUIRED: Tree Representation

Before starting Phase 7, decide the Rust tree representation:

**Option A: Arena + indices** (recommended)
```rust
struct Arena { nodes: Vec<Node> }
struct NodeId(u32);
struct Node {
    parent: Option<NodeId>,
    first_child: Option<NodeId>,
    last_child: Option<NodeId>,
    prev_sibling: Option<NodeId>,
    next_sibling: Option<NodeId>,
    // ...
}
```

**Option B: Rc<RefCell<Node>>**
```rust
type NodeRef = Rc<RefCell<Node>>;
struct Node {
    parent: Option<Weak<RefCell<Node>>>,
    children: Vec<NodeRef>,
    // ...
}
```

**Option C: Raw pointers** (not recommended — defeats the purpose)

This decision cascades through Phases 7-10 and cannot be changed later
without rewriting all tree-walking code.

---

## Phase 7: access.c + attrs.c (Tree Readers)

**Status**: Not started
**Depends on**: Tree representation decision

Read-only tree traversal for accessibility checking and attribute
validation. These modules emit diagnostic messages but don't modify
the tree.

---

## Phase 8: pprint.c (Pretty Printer)

**Status**: Not started
**Depends on**: Phase 7

Reads the tree, writes formatted HTML output. Most testable module
(all 402 regression tests directly validate its output).

---

## Phase 9: clean.c (Cleanup Passes)

**Status**: Not started
**Depends on**: Phase 8

Tree mutation — creates, removes, and re-parents nodes. ~25
independent cleanup functions that can be swapped one at a time.

---

## Phase 10: lexer.c + parser.c (The Boss Fight)

**Status**: Not started
**Depends on**: All previous phases

11,000 lines of tightly coupled tokenizer and parser. These must be
replaced together because they share mutable state.

By this point, all other modules are Rust, so the parser is the last
C code in the project.

### External Test Data for This Phase

- html5lib tree construction tests (~500+)
- DOMPurify XSS vectors (~500+)
- MutaGen adversarial fragments (sample from 12M)
- All 402 regression tests
- All oracle-generated tests from previous phases

---

## Milestones

| Milestone | Phases | Significance |
|-----------|--------|-------------|
| **M1: First Rust** | 0-1 | Rust code in the build, entities replaced |
| **M2: Leaf modules done** | 0-3 | All allocation-free modules replaced |
| **M3: All non-tree modules** | 0-6 | Everything that doesn't touch Node tree |
| **M4: Full read path** | 0-8 | Can read and output HTML entirely in Rust (except parsing) |
| **M5: Full rewrite** | 0-10 | No C code remains (except C API shim) |

Expected stopping point: **M3** (Phases 0-6). Going further requires
the tree representation decision and significantly more complex FFI.

---

## External Test Sources

Fetch or vendor these for testing:

- [ ] html5lib-tests: `git clone https://github.com/html5lib/html5lib-tests`
  - `tokenizer/entities.test` for Phase 1
  - `tree-construction/*.dat` for Phase 10
- [ ] Markus Kuhn UTF-8 stress test for Phase 3
- [ ] DOMPurify vectors: extract from `test/fixtures/expect.mjs`
