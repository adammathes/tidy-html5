# AGENTS.md — Instructions for AI Agents

## FIRST: Read These Files in Order

1. **This file** (AGENTS.md) — project principles and architecture
2. **[LEARNINGS.md](LEARNINGS.md)** — shared knowledge from all prior agents (READ THE WHOLE THING)
3. **[ROADMAP.md](ROADMAP.md)** — current progress, find what needs doing
4. **[workflows/](workflows/)** — step-by-step instructions for your specific phase

## LAST: Before You Finish

1. Update **[ROADMAP.md](ROADMAP.md)** — mark your checklist items done
2. Add an entry to **[LEARNINGS.md](LEARNINGS.md)** — what you did, what you learned, what went wrong
3. Commit your changes

These are not optional. The learnings journal is how agents communicate
across sessions. If you skip it, the next agent starts blind.

---

## Project Overview

This is **tidy-oxide**: an incremental Rust rewrite of [HTML Tidy](https://github.com/htacg/tidy-html5) (tidy-html5), a ~58,000-line C library/tool for parsing, validating, cleaning, and pretty-printing HTML.

The approach: use the existing C implementation as a **black-box oracle** for correctness, and replace modules one at a time with Rust. The C binary is always the ground truth — if the Rust code produces different output for the same input, the Rust code is wrong unless explicitly documented otherwise.

## How to Execute a Phase

1. Check [ROADMAP.md](ROADMAP.md) to find the next phase to work on
2. Open the corresponding workflow file in [workflows/](workflows/):
   - `workflows/phase-0-build-infra.md`
   - `workflows/phase-1-entities.md`
   - `workflows/phase-2-tmbstr.md`
   - `workflows/phase-3-utf8.md`
   - `workflows/phase-4-attrdict-tags.md`
   - `workflows/phase-5-config-messages.md`
   - `workflows/phase-6-io.md`
3. Follow the workflow step by step
4. Update ROADMAP.md and LEARNINGS.md when done

## Repository Structure

```
tidy-oxide/
├── AGENTS.md              # THIS FILE — read first
├── LEARNINGS.md           # Shared knowledge journal (read + write!)
├── ROADMAP.md             # Progress tracker with checklists
├── PLAN.md                # Detailed technical plan
├── README.md              # Project overview and motivation
│
├── workflows/             # Step-by-step phase execution guides
│   ├── phase-0-build-infra.md
│   ├── phase-1-entities.md
│   ├── phase-2-tmbstr.md
│   ├── phase-3-utf8.md
│   ├── phase-4-attrdict-tags.md
│   ├── phase-5-config-messages.md
│   └── phase-6-io.md
│
├── reference/             # READONLY snapshot of tidy-html5 C source
│   ├── src/               # C source files (DO NOT MODIFY)
│   ├── include/           # C public headers (DO NOT MODIFY)
│   ├── regression_testing/# Existing 402 test cases
│   ├── CMakeLists.txt     # C build system
│   └── version.txt        # Version: 5.9.20, 2022.01.25
│
├── rust/                  # Rust crate (the new implementation)
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs         # Crate root
│   │   ├── entities.rs    # Phase 1: Entity lookup
│   │   ├── tmbstr.rs      # Phase 2: String utilities
│   │   ├── utf8.rs        # Phase 3: UTF-8 encoding
│   │   └── ...            # Future modules
│   ├── build.rs           # Build script (links to C code during transition)
│   └── tests/
│       ├── entities_test.rs
│       ├── tmbstr_test.rs
│       └── ...
│
├── ffi/                   # FFI bridge (C <-> Rust)
│   ├── Cargo.toml         # Separate crate for FFI exports
│   ├── src/
│   │   └── lib.rs         # #[no_mangle] extern "C" wrappers
│   └── cbindgen.toml      # Auto-generate C headers from Rust
│
├── tests/                 # Integration / oracle tests
│   ├── oracle/            # Test harness that runs C binary and Rust, diffs output
│   │   ├── run_oracle.sh  # Script: run C tidy + Rust tidy, compare
│   │   └── fixtures/      # Generated test fixtures (input + expected output)
│   ├── html5lib/          # Vendored or fetched html5lib-tests data
│   └── regression/        # Extended regression tests beyond the 402 originals
│
├── tools/                 # Helper scripts
│   ├── extract_entities.py  # Extract entity table from C for test generation
│   ├── generate_oracle_fixtures.sh
│   └── diff_outputs.sh
│
└── docs/                  # Analysis documents (from planning phase)
    ├── RUST_REWRITE_ANALYSIS.md
    └── OPTION_E_EVALUATION.md
```

## Key Principles

### 1. The C Binary is the Oracle

For EVERY module being replaced:
1. Write tests that capture the C implementation's exact behavior
2. Write the Rust replacement
3. Run the same tests against the Rust code
4. They must match byte-for-byte (or divergences must be explicitly documented)

The C source is in `reference/` and must NEVER be modified. Build it once,
use it as a reference binary.

### 2. One Module at a Time

Never rewrite two modules simultaneously. The sequence is:
1. Read the C source for module X
2. Generate exhaustive tests using the C oracle
3. Write Rust implementation of module X
4. Validate against oracle
5. Wire up FFI bridge so C code calls Rust module X
6. Run full regression suite (all 402 cases)
7. Commit only when green

See PLAN.md for the module order.

### 3. FFI Naming Convention

All internal tidy functions use the `TY_()` macro which expands to
`prvTidy` prefix:

```c
#define TY_(str) TYDYAPPEND(prvTidy,str)
// TY_(EntityInfo) → prvTidyEntityInfo
```

Rust FFI exports must match these mangled names exactly:

```rust
#[no_mangle]
pub unsafe extern "C" fn prvTidyEntityInfo(...) -> Bool { ... }
```

### 4. The Allocator Problem

Tidy uses a custom allocator vtable (`TidyAllocator`). Functions that
allocate memory receive an `allocator` parameter and call through it:

```c
tmbstr TY_(tmbstrdup)(TidyAllocator *allocator, ctmbstr str) {
    tmbstr cp = (tmbstr) TidyAlloc(allocator, 1+len);
    ...
}
```

**Rule**: Any Rust function that replaces a C function which allocates
via `TidyAllocator` MUST also allocate via that same allocator. Do NOT
use Rust's standard allocator for memory that crosses the FFI boundary.

For modules that don't allocate (entities.c), this isn't an issue.

### 5. Type Mappings

| C Type | Rust Equivalent | Notes |
|--------|----------------|-------|
| `ctmbstr` | `*const c_char` | Const string pointer (typedef for `const char*`) |
| `tmbstr` | `*mut c_char` | Mutable string pointer (typedef for `char*`) |
| `uint` | `c_uint` | `unsigned int` |
| `Bool` / `yes` / `no` | `c_int` (0 or 1) | Tidy's boolean is typedef'd int |
| `byte` | `u8` | `unsigned char` |
| `tmbchar` | `c_char` | `char` |
| `tchar` | `u32` | `unsigned int`, used for Unicode codepoints |
| `TidyAllocator*` | `*mut TidyAllocator` | Opaque allocator vtable pointer |

### 6. Testing Strategy

For each module, tests should come from multiple sources:

1. **Oracle tests**: Run C function with input, capture output, compare
   with Rust function output
2. **html5lib-tests**: Where applicable (entities, tokenizer, tree
   construction)
3. **Edge cases**: Empty input, NULL pointers, maximum values, invalid
   UTF-8, etc.
4. **Regression tests**: All 402 existing tidy test cases must pass
   after each module swap

Tests live in `rust/tests/` for unit tests and `tests/` for integration
tests.

### 7. Build System

During the transition period, the project builds in two stages:

1. **C reference binary**: Built from `reference/` using CMake (once,
   cached). Used only for oracle testing.
2. **Rust crate**: Built with Cargo. Links against remaining C object
   files during transition phases.

The `ffi/` crate produces a C-compatible static library that can be
linked in place of the corresponding C `.o` files.

Eventually (after all phases complete), the C build is no longer needed
and the project is pure Rust with a C API shim.

## Working on a Phase

When you are assigned a phase (e.g., "Phase 1: entities.c"):

### 0. Before anything else

1. Read **[LEARNINGS.md](LEARNINGS.md)** — the whole file
2. Open the workflow file for your phase in **[workflows/](workflows/)**
3. Mark the phase as `[~]` in progress in **[ROADMAP.md](ROADMAP.md)**
4. Follow the workflow step by step

The workflow files have detailed, phase-specific instructions. The
general pattern is below for reference.

### Step 1: Read the C Source

Read the relevant C file(s) in `reference/src/`. Understand:
- What functions are exported (look for `TY_()` declarations in the `.h`)
- What functions they call (dependencies)
- What data structures they use
- What allocations they make
- What the caller expects (read the call sites)

### Step 2: Generate Tests

Create test fixtures by:
1. Reading the C source to enumerate all code paths
2. Writing a small C harness that calls each function with diverse inputs
   and prints the results (or use the oracle script)
3. Harvesting external test data (html5lib-tests for entities, etc.)
4. Writing Rust test functions that encode the expected behavior

Place unit tests in `rust/tests/<module>_test.rs`.

### Step 3: Write the Rust Module

Create `rust/src/<module>.rs`. Guidelines:
- Match the C behavior exactly, not what you think the behavior *should* be
- Use safe Rust wherever possible
- Where `unsafe` is required (FFI), keep it minimal and document the
  safety invariants
- Do NOT add extra features, optimizations, or "improvements"
- Do NOT change the public API behavior
- Do NOT use external crates unless absolutely necessary and discussed first

### Step 4: Wire Up FFI

In `ffi/src/lib.rs`, add `#[no_mangle] pub unsafe extern "C"` wrappers
that match the C function signatures. These are the functions the C code
will call.

### Step 5: Validate

1. Run the module's unit tests: `cargo test`
2. Build the hybrid binary (Rust module + remaining C code)
3. Run all 402 regression tests against the hybrid binary
4. Fix any failures

### Step 6: Update Progress

1. Mark checklist items `[x]` in **[ROADMAP.md](ROADMAP.md)**
2. Add a journal entry to **[LEARNINGS.md](LEARNINGS.md)** using the template
3. Commit all changes

## Common Pitfalls

### Don't Assume C Behavior from the Spec
Tidy does NOT follow the WHATWG HTML spec strictly. It has 25 years of
custom error recovery, browser compatibility heuristics, and intentional
spec deviations. Always test against the actual C binary, not the spec.

### Don't "Fix" Bugs During Translation
If the C code has a bug (e.g., the `strrep()` buffer overflow), the
Rust replacement should produce the same OUTPUT as the C code for
non-pathological inputs. You can make the Rust code safe internally
(bounds-checked), but the observable behavior must match.

Exception: if a C bug causes undefined behavior that makes the output
non-deterministic, document it and pick a reasonable behavior for Rust.

### Don't Fight the Allocator
If a C function allocates with `TidyAllocator`, the Rust replacement
must too. Don't try to use Rust's allocator for cross-boundary memory.
This is the #1 source of subtle bugs in C-to-Rust rewrites.

### Don't Rewrite Two Modules at Once
Each module must be individually validated before moving on. If you
rewrite entities.c and tmbstr.c simultaneously and tests fail, you
won't know which one is wrong.

## Reference Material

- `docs/RUST_REWRITE_ANALYSIS.md` — Full codebase analysis
- `docs/OPTION_E_EVALUATION.md` — Honest feasibility assessment
- `reference/src/*.c` — The C source code (read-only reference)
- `reference/include/tidy.h` — The public API
- `reference/src/forward.h` — Internal type definitions and TY_() macro
- `reference/regression_testing/` — 402 existing test cases
