# HTML Tidy: Incremental Rust Rewrite Feasibility Analysis

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Codebase Overview](#codebase-overview)
3. [Current Testing Infrastructure](#current-testing-infrastructure)
4. [Safety and Security Audit](#safety-and-security-audit)
5. [Option A: Incremental Rust Rewrite via FFI](#option-a-incremental-rust-rewrite-via-ffi)
6. [Option B: Clean-Room Rust Rewrite Using html5ever](#option-b-clean-room-rust-rewrite-using-html5ever)
7. [Option C: Harden the Existing C Codebase](#option-c-harden-the-existing-c-codebase)
8. [Option D: Hybrid — Rewrite Safety-Critical Paths Only](#option-d-hybrid--rewrite-safety-critical-paths-only)
9. [Testing Strategy for Any Rewrite](#testing-strategy-for-any-rewrite)
10. [Recommendation Matrix](#recommendation-matrix)
11. [Conclusion](#conclusion)

---

## Executive Summary

HTML Tidy (tidy-html5) is a ~58,000-line C library for parsing, validating, cleaning, and pretty-printing HTML/XHTML/XML. It has zero external dependencies, a clean public API with opaque types, a pluggable allocator system, and a well-defined compiler-like architecture (Lexer -> Parser -> AST -> Pretty Printer).

**Key findings:**

- The codebase is well-structured and modular, but has real memory safety issues (buffer overflows in string handling, no input size limits, global mutable state)
- The test suite is **strong for regression detection** (402 black-box test cases across 6 sets, multi-platform CI) but has **critical gaps** for a rewrite: no unit tests, no fuzzing, no memory safety tooling (ASan/UBSan), no API-level tests, and no performance benchmarks
- An incremental Rust rewrite is technically feasible but the FFI boundary cost is high — real-world experience from Firefox/Stylo, Meta, and others shows FFI-heavy Rust "feels like a completely different language" and can introduce new classes of undefined behavior
- The existing Rust ecosystem already has mature HTML parsers (html5ever, lol-html) that solve many of the same problems differently
- **The strongest argument against a rewrite is that hardening the existing C code (fuzzing, sanitizers, fixing known issues) delivers most of the safety benefits at a fraction of the cost**
- **The strongest argument for a rewrite is long-term maintainability** — finding C developers for open-source projects is increasingly difficult, and Rust's type system prevents entire categories of bugs from recurring

---

## Codebase Overview

### Architecture

```
Input HTML/XML
      |
      v
  [StreamIn]     -- I/O abstraction (file, buffer, string, stdin)
      |
      v
   [Lexer]       -- State machine tokenizer (4,548 lines)
      |           -- Produces tokens: text, start tags, end tags, comments, etc.
      v           -- Stores all text in a shared `lexbuf` with offset spans
   [Parser]      -- Non-recursive, stack-based parser (6,450 lines)
      |           -- Builds DOM tree from token stream
      |           -- Automatic error recovery and tag inference
      v
  [Node Tree]    -- Linked tree of Node structs (DOM representation)
      |
      +---> [Clean]      -- Markup cleanup and repair (2,862 lines)
      +---> [Attrs]      -- Attribute validation (2,777 lines)
      +---> [Access]     -- WCAG accessibility checking (3,540 lines)
      |
      v
  [PrettyPrint]  -- Output formatting (2,714 lines)
      |
      v
  [StreamOut]    -- Output abstraction
```

### File Inventory (src/ and include/)

| Category | Files | Lines | Key Files |
|----------|-------|-------|-----------|
| **Lexer/Parser** | 4 (.c + .h) | ~11,835 | `lexer.c` (4,548), `parser.c` (6,450) |
| **DOM/Tree** | included in lexer.h | ~750 | Node, AttVal structs in `lexer.h` |
| **Tags/Attributes** | 6 | ~8,348 | `tags.c`, `attrs.c`, `attrdict.c` |
| **Cleanup/Repair** | 2 | ~2,943 | `clean.c` |
| **Accessibility** | 2 | ~3,630 | `access.c` |
| **Pretty Print** | 2 | ~2,857 | `pprint.c` |
| **Configuration** | 2 | ~2,438 | `config.c` |
| **I/O** | 6 | ~2,323 | `streamio.c`, `fileio.c`, `buffio.c` |
| **Messages/i18n** | 5 | ~7,980 | `message.c`, `language.c`, `language_en.h` |
| **String/Encoding** | 4 | ~1,858 | `tmbstr.c`, `utf8.c`, `charsets.c` |
| **Entities** | 2 | ~2,213 | `entities.c` |
| **Core API** | 3 | ~5,646 | `tidylib.c`, `tidy-int.h`, `tidy.h` |
| **Platform** | 2 | ~1,018 | `tidyplatform.h`, `alloc.c` |
| **Console** | 1 | ~700 | `tidy.c` (console app) |
| **Total** | ~56 files | **~58,000** | |

### Core Data Structures

```c
// The top-level document — everything hangs off this
struct _TidyDocImpl {
    Node root;                    // Document tree root
    Lexer* lexer;                 // Tokenizer state
    TidyConfigImpl config;        // ~100 configuration options
    TidyTagImpl tags;             // Tag dictionary
    TidyAttribImpl attribs;       // Attribute definitions
    TidyAccessImpl access;        // Accessibility checker state
    TidyPrintImpl pprint;         // Pretty-printer state
    TidyParserStack stack;        // Non-recursive parser stack
    StreamIn* docIn;              // Input stream
    StreamOut* docOut;            // Output stream
    StreamOut* errout;            // Error output stream
    TidyAllocator* allocator;     // Pluggable memory allocator
    uint errors, warnings;        // Counters
    // ... callbacks, mutedMessages, etc.
};

// DOM node — linked tree structure
struct _Node {
    Node *parent, *prev, *next, *content, *last;  // Tree pointers
    AttVal* attributes;           // Linked list of attributes
    const Dict* tag;              // Tag definition lookup
    tmbstr element;               // Tag name string
    uint start, end;              // Offset span into lexer buffer
    NodeType type;                // TextNode, StartTag, EndTag, etc.
    uint line, column;            // Source position
};

// Attribute — linked list per node
struct _AttVal {
    AttVal* next;
    const Attribute* dict;        // Attribute definition
    tmbstr attribute;             // Name
    tmbstr value;                 // Value
    int delim;                    // Quote character
};
```

### Public API Surface

The library exposes **~173 functions** through `include/tidy.h` using opaque handle types:

```c
TidyDoc doc = tidyCreate();
tidyOptSetBool(doc, TidyShowWarnings, yes);
tidyParseString(doc, htmlString);
tidyCleanAndRepair(doc);
tidyRunDiagnostics(doc);
tidySaveString(doc, buffer, &buflen);
tidyRelease(doc);
```

The API is well-designed: opaque types, no exposed internals, const-correct, and uses a document-scoped pattern that makes it theoretically thread-safe (one TidyDoc per thread).

### Key Architectural Properties

| Property | Status | Rewrite Implication |
|----------|--------|-------------------|
| External dependencies | **None** (pure C + libc) | Simplifies build integration |
| Global mutable state | **Yes** — allocator function pointers, debug logging | Must be eliminated |
| Thread safety | **No** — designed for single-threaded use | Rust's Send/Sync can enforce |
| Custom allocator | **Yes** — vtable-based, pluggable | Maps to Rust's Allocator trait |
| Error handling | Return codes (0/1/2) + callbacks | Maps to Result<T, E> |
| String handling | Raw `char*` with thin wrappers | Maps to String/&str |
| Memory model | Manual malloc/free via allocator | Ownership model is natural fit |
| Platform code | Extensive `#ifdef` (~700 lines) | Rust's `cfg()` is cleaner |

---

## Current Testing Infrastructure

### What Exists

The test suite lives in `regression_testing/` and uses a custom Ruby-based test runner (`test.rb`) built with the Thor CLI framework.

**Test sets:**

| Set | Cases | Purpose |
|-----|-------|---------|
| `legacy-cases` | ~220 | Historical HTML edge cases (SourceForge era) |
| `access-cases` | ~119 | WCAG accessibility compliance |
| `github-cases` | ~30 | Regression tests for GitHub issues |
| `xml-cases` | ~27 | XML/XHTML output |
| `dev-cases` | ~5 | Work-in-progress tests |
| `special-cases` | ~1 | Large file stress test (skipped by default) |
| **Total** | **~402** | |

**How each test works:**

1. Input: `case-<name>@<expected-exit-code>.html` (e.g., `case-119a@1.html`)
2. Config: `case-<name>.conf` or `config_default.conf`
3. Run: `tidy -lang en_us -f <output.txt> -config <config> --tidy-mark no -o <output.html> <input.html>`
4. Compare: `diff` the output HTML against `<name>-expects/<name>.html`
5. Compare: `diff` the console output against `<name>-expects/<name>.txt`
6. Verify: Check exit code matches the `@N` in the filename

**CI/CD:** GitHub Actions runs tests on Ubuntu, macOS (x86_64 + ARM64), Windows (MSVC, MinGW, Cygwin) on every push to `master`/`next` and all PRs.

### What's Missing (Critical Gaps for a Rewrite)

| Gap | Severity | Impact on Rewrite |
|-----|----------|-------------------|
| **No unit tests** | High | Cannot test individual components (lexer, parser) in isolation. You can only test the full pipeline end-to-end. |
| **No fuzzing** | Critical | For a parser that handles untrusted input, this is a serious omission. No AFL, libFuzzer, or OSS-Fuzz integration. |
| **No memory safety tooling** | Critical | No AddressSanitizer, UndefinedBehaviorSanitizer, or Valgrind integration in CI. Existing bugs may be latent. |
| **No API-level tests** | High | The 173-function C API has no systematic test coverage. Only a Swift wrapper exists (macOS only). |
| **No performance benchmarks** | Medium | No way to detect performance regressions. A Rust rewrite could be slower and you wouldn't know. |
| **No property-based testing** | Medium | Test cases are manually curated. No generative testing for parser invariants. |
| **No coverage measurement** | Medium | Unknown which code paths are exercised. |

### Verdict on Test Suite Adequacy

**For maintaining the current C codebase**: The test suite is adequate. 402 regression tests with multi-platform CI catches most behavioral regressions.

**For an incremental rewrite**: **Insufficient.** The test suite only validates the final output. It cannot:
- Verify that internal data structures are equivalent between C and Rust implementations
- Catch performance regressions
- Exercise edge cases a fuzzer would find
- Test individual components being replaced

**Before any rewrite begins, the test infrastructure must be significantly strengthened.** This is prerequisite work regardless of which rewrite strategy is chosen.

---

## Safety and Security Audit

### Known Vulnerabilities in Current Code

**1. Buffer overflow in `strrep()` — `src/tmbstr.c:250-277`**

```c
void TY_(strrep)(tmbstr buffer, ctmbstr str, ctmbstr rep)
{
    char buf[1024];  // Fixed-size stack buffer — OVERFLOW RISK
    // ...
    strcpy(buf, rep);           // No bounds check
    strcat(buf, p+strlen(str)); // No bounds check
    // ...
    strcpy(buffer, buf);        // Assumes buffer >= 1024
}
```

This function uses a fixed 1024-byte stack buffer with unbounded `strcpy`/`strcat` operations. If replacement strings exceed 1024 bytes, this is exploitable.

**2. Unbounded string operations throughout**

- `tmbstrcpy` wraps `strcpy` — no bounds checking
- `tmbstrcat` wraps `strcat` — no bounds checking
- Fallback to `vsprintf` (without `n`) on older platforms

**3. No input size limits**

The lexer buffer (`lexbuf`) grows unboundedly via `realloc`. A multi-gigabyte malicious input could exhaust memory without any protection.

**4. Global mutable state is not thread-safe**

```c
// src/alloc.c — shared across ALL TidyDoc instances
static TidyMalloc  g_malloc  = NULL;
static TidyRealloc g_realloc = NULL;
static TidyFree    g_free    = NULL;
static TidyPanic   g_panic   = NULL;
```

Race conditions possible if `tidySetMallocCall()` is called from multiple threads.

**5. No cycle detection in tree traversal**

Tree navigation functions assume acyclic structure. A bug that creates a cycle (e.g., `node->parent->content == node`) would cause infinite loops.

**6. OOM handling calls `exit(2)`**

The default panic handler terminates the process. Libraries should not call `exit()` — this is hostile to embedders.

### What Rust Would Fix

| Issue | Rust Solution |
|-------|---------------|
| Buffer overflows | Bounds-checked `Vec<u8>`, `String` |
| Use-after-free | Ownership system prevents dangling pointers |
| Null pointer dereference | `Option<T>` instead of nullable pointers |
| Thread safety | `Send`/`Sync` traits enforced at compile time |
| Global mutable state | No `static mut` without `unsafe`; use `Arc<Mutex<T>>` |
| Integer overflow | Checked arithmetic in debug, configurable in release |
| OOM handling | `Result<T, AllocError>` with `try_reserve()` |

### What Rust Would NOT Fix

| Issue | Why Not |
|-------|---------|
| Logic bugs in HTML parsing | Rust prevents memory corruption, not algorithmic errors |
| Spec compliance gaps | Same parser logic, same spec interpretation |
| Performance issues | Rust is not automatically faster; `Rc<RefCell<Node>>` trees have overhead |
| API design problems | The public API must remain compatible regardless |
| Configuration complexity | 100+ options remain complex in any language |

---

## Option A: Incremental Rust Rewrite via FFI

### Strategy

Replace C modules one at a time with Rust implementations, maintaining the existing C API via FFI throughout. The C and Rust code coexist in the same binary.

### Migration Order

A bottom-up approach, replacing leaf dependencies first:

```
Phase 1 — Utilities (Low risk, low coupling)
  ├── tmbstr.c    → rust string utilities
  ├── utf8.c      → use Rust's native UTF-8 String
  ├── charsets.c   → encoding detection
  ├── entities.c   → entity lookup table
  └── alloc.c      → Rust allocator trait impl

Phase 2 — I/O Layer (Medium risk, clear boundary)
  ├── streamio.c   → Rust Read/Write traits
  ├── fileio.c     → std::fs wrappers
  ├── buffio.c     → Vec<u8> buffers
  └── mappedio.c   → memory-mapped I/O

Phase 3 — Configuration & Messages (Medium risk)
  ├── config.c     → typed configuration with serde
  ├── message.c    → structured diagnostics
  ├── messageobj.c
  └── language.c   → i18n with Rust crate

Phase 4 — Tag/Attribute Dictionaries (Medium risk)
  ├── tags.c       → HashMap-based lookups
  ├── attrs.c      → validated attribute types
  └── attrdict.c   → attribute dictionary

Phase 5 — Core Engine (HIGH risk, tight coupling)
  ├── lexer.c      → enum-based state machine
  ├── parser.c     → iterative parser with owned stack
  ├── clean.c      → tree transformation passes
  └── istack.c     → inline element stack

Phase 6 — Output & Analysis (Medium risk)
  ├── pprint.c     → pretty printer
  ├── access.c     → accessibility checker
  └── tidylib.c    → API implementation

Phase 7 — Remove C completely
  └── FFI shim becomes the only C code (for C API compatibility)
```

### FFI Boundary Design

At each phase boundary, the Rust code must expose a C-compatible interface:

```rust
// Example: Rust entity lookup replacing entities.c
#[no_mangle]
pub extern "C" fn TY_EntityInfo(name: *const c_char, is_xml: c_int) -> *const Entity {
    let name = unsafe { CStr::from_ptr(name) };
    // ... Rust implementation
}
```

And consume C types:

```rust
// Rust code calling into remaining C code
extern "C" {
    fn TY_ParseDocument(doc: *mut TidyDocImpl) -> c_int;
}
```

### Advantages

- **Continuous deployability**: The library works at every step
- **Risk containment**: Each phase can be individually validated
- **Feature parity**: New features can be added mid-rewrite (unlike parallel rewrite)
- **Preserves the public C API**: Existing consumers don't need to change
- **Proven pattern**: Meta, Firefox/Stylo, and others have done this successfully

### Disadvantages

- **FFI overhead is real and painful**: Every C-Rust boundary requires `unsafe`, raw pointers, manual lifetime management. The safety gains from Rust are partially negated at boundaries. Research shows FFI boundaries are where most bugs in "rewrite-it-in-Rust" efforts actually occur ([Rewrite it in Rust Considered Harmful?](https://goto.ucsd.edu/~rjhala/hotos-ffi.pdf)).
- **"Using Rust with a lot of C interop feels like a completely different language"** — the ergonomic benefits of Rust largely disappear when every other function call crosses an FFI boundary.
- **Phase 5 is a cliff**: The lexer and parser are tightly coupled and account for ~11,000 lines. You can't incrementally replace half the lexer. This phase is effectively a mini big-bang rewrite.
- **`Node` structure is everywhere**: The DOM tree type is used by virtually every module. Changing it requires updating all modules simultaneously or maintaining two parallel representations (which defeats the purpose).
- **Build system complexity**: CMake must orchestrate both a C compiler and `cargo`, with correct linking. This is solvable (corrosion, cmake-cargo) but adds friction.
- **Estimated effort**: 12-18 person-months for a developer experienced in both C and Rust. Based on comparable projects (Servo CSS in Firefox took a team years; the 80K-line HACL* verified crypto library took 3 person-months with automated tooling on a cleaner codebase).

### Critical Risk: The Node Structure Problem

The `Node` struct is the heart of tidy-html5. It's a linked tree with raw pointers:

```c
struct _Node {
    Node *parent, *prev, *next, *content, *last;
    AttVal* attributes;
    // ...
};
```

In Rust, representing this as a safe structure is notoriously difficult. Options:

1. **`Rc<RefCell<Node>>`**: Safe but has runtime overhead (reference counting, borrow checking). Breaks if you need mutable access to parent and child simultaneously.
2. **Arena allocation**: Use an arena (e.g., `typed-arena`, `bumpalo`) where nodes are indices into a `Vec`. Fast and safe, but requires threading the arena through all code.
3. **`unsafe` raw pointers**: Essentially replicating the C approach in Rust, gaining little safety.
4. **`petgraph` or similar graph library**: Adds a dependency and may not match tidy's traversal patterns well.

This is not a theoretical concern — it's the central design decision of the entire rewrite and will cascade through every module. Getting it wrong means either pervasive `unsafe` (defeating the purpose) or painful refactoring of every tree operation.

---

## Option B: Clean-Room Rust Rewrite Using html5ever

### Strategy

Instead of translating the C code, build a new "tidy" tool on top of the existing Rust HTML parsing ecosystem.

### Architecture

```
html5ever (parsing) + markup5ever (tree building)
    |
    v
New "tidy" layer:
    ├── Tree walker / validator
    ├── Cleanup passes (equivalent to clean.c)
    ├── Accessibility checker
    ├── Pretty printer
    └── Configuration system

Optional: C API shim via cbindgen for backward compatibility
```

### Advantages

- **Leverage battle-tested parser**: html5ever has 12M+ downloads, is spec-compliant (WHATWG), and is already memory-safe.
- **No FFI pain**: Pure Rust from day one.
- **Modern HTML spec**: html5ever follows the living WHATWG standard, while tidy-html5 has its own historical parser that diverges from the spec in various ways.
- **Smaller scope**: Only the "tidy" logic (cleanup, validation, pretty-printing) needs to be written. The parser already exists.
- **Community**: html5ever is actively maintained by the Servo project.

### Disadvantages

- **Different behavior**: html5ever and tidy-html5 parse HTML differently. html5ever follows WHATWG strictly; tidy has its own error recovery heuristics developed over 25 years. The output WILL differ, breaking existing users.
- **No backward compatibility**: The 173-function C API cannot be preserved. Every consumer of libtidy must rewrite their integration.
- **Missing "tidy" functionality**: html5ever is only a parser. All the cleanup, repair, accessibility checking, and pretty-printing logic (~15,000+ lines) must be rewritten from scratch.
- **Lost institutional knowledge**: 25 years of edge cases, browser compatibility workarounds, and user-reported fixes embedded in tidy's parser would need to be rediscovered.
- **Second-system effect**: High risk of over-engineering the new design while losing parity with the old one.
- **Estimated effort**: 6-12 person-months, but with a long tail of behavioral compatibility issues.

---

## Option C: Harden the Existing C Codebase

### Strategy

Instead of rewriting, invest in modern C safety tooling and practices to get most of the safety benefits without the rewrite cost.

### Action Plan

```
Phase 1 — Immediate Safety (1-2 weeks)
  ├── Fix strrep() buffer overflow in tmbstr.c
  ├── Replace all unbounded strcpy/strcat with bounded variants
  ├── Add input size limits to the lexer
  ├── Add ASan + UBSan builds to CI
  └── Run existing test suite under sanitizers

Phase 2 — Fuzzing (2-4 weeks)
  ├── Write libFuzzer harness for tidyParseString()
  ├── Write libFuzzer harness for tidyParseBuffer()
  ├── Seed corpus from existing regression tests
  ├── Integrate with OSS-Fuzz for continuous fuzzing
  └── Fix all findings

Phase 3 — Structural Improvements (1-2 months)
  ├── Eliminate global mutable state in alloc.c
  │   (make allocator per-document, not global)
  ├── Add thread-safety documentation
  ├── Add unit tests for lexer, parser, config modules
  ├── Add code coverage measurement to CI
  └── Add performance benchmarks

Phase 4 — Ongoing
  ├── Maintain fuzzing corpus
  ├── Keep sanitizer builds green
  ├── Consider safe-C subsets or static analysis (Coverity, etc.)
  └── Gradual refactoring as needed
```

### Advantages

- **Lowest cost, highest immediate safety ROI**: Fuzzing alone typically finds more real bugs in a week than a rewrite prevents in a year.
- **No behavioral changes**: Users see zero difference.
- **No build system changes**: CMake stays, no Rust toolchain required.
- **No FFI complexity**: No unsafe boundaries to maintain.
- **Battle-tested code preserved**: 25 years of fixes and edge cases remain intact.
- **Immediate**: Can start today, first results in days.
- **Estimated effort**: 2-3 person-months total.

### Disadvantages

- **Does not prevent future memory bugs**: New C code can still introduce use-after-free, buffer overflows, etc. Sanitizers catch bugs at runtime, not at compile time.
- **Developer recruitment**: C is increasingly niche for open-source projects. Fewer contributors will be attracted than to a Rust project.
- **Technical debt remains**: The codebase still has `#ifdef` spaghetti, global state, and manual memory management. These aren't bugs, but they impede maintainability.
- **No compile-time safety guarantees**: Rust's ownership model prevents categories of bugs statically. C tooling is always reactive (finding bugs after they're written).

---

## Option D: Hybrid — Rewrite Safety-Critical Paths Only

### Strategy

Keep the existing C codebase but rewrite only the most security-sensitive components in Rust: the lexer (which handles untrusted input) and string handling.

### What Gets Rewritten

```
Rust (new):
  ├── lexer.c → Safe tokenizer with bounds-checked buffers
  ├── tmbstr.c → Rust String/&str with proper UTF-8
  ├── utf8.c → Rust's native UTF-8 handling
  ├── entities.c → HashMap-based entity resolution
  └── buffio.c → Vec<u8> buffer management

C (kept):
  ├── parser.c (consumes tokens from Rust lexer via FFI)
  ├── clean.c, attrs.c, access.c (tree operations)
  ├── pprint.c (output formatting)
  ├── config.c (configuration)
  ├── tidylib.c (API layer)
  └── Everything else
```

### FFI Boundary

The key interface is between the Rust lexer and C parser:

```rust
// Rust lexer produces tokens consumed by C parser
#[repr(C)]
pub struct Token {
    token_type: TokenType,
    text_start: u32,
    text_end: u32,
    line: u32,
    column: u32,
}

#[no_mangle]
pub extern "C" fn tidy_get_next_token(lexer: *mut RustLexer) -> Token { ... }
```

### Advantages

- **Targets the highest-risk code**: The lexer is the most likely source of memory corruption bugs because it directly processes untrusted input.
- **Limited FFI surface**: Only the lexer-to-parser boundary needs FFI, which is a well-defined token stream.
- **Preserved architecture**: Most of the codebase stays C; the rewrite is contained.
- **Estimated effort**: 3-5 person-months.

### Disadvantages

- **Partial safety**: Tree operations in C can still have memory bugs.
- **Two-language build**: Still need CMake + Cargo integration.
- **FFI complexity**: The token stream boundary seems clean but the lexer currently shares mutable state with the parser (e.g., `lexbuf` is referenced by Node offsets). Untangling this is non-trivial.
- **Maintenance burden**: Contributors must know both C and Rust.

---

## Testing Strategy for Any Rewrite

Regardless of which option is chosen, the test infrastructure must be strengthened first. This is prerequisite work.

### Required Test Investments

#### 1. Fuzzing (Critical — Do This First)

```
Priority: P0
Effort: 1-2 weeks

- Write libFuzzer/AFL harness for tidyParseString()
- Seed corpus from existing 402 regression test inputs
- Run for 72+ hours on multi-core
- Fix all crashes and UBSan findings
- Integrate with OSS-Fuzz for continuous coverage
- This establishes a behavioral baseline BEFORE any rewrite
```

**Why first**: Fuzzing the current C code creates a massive corpus of edge-case inputs. This corpus becomes the golden test suite for validating a Rust rewrite — any input that the C version handles without crashing must also be handled by the Rust version.

#### 2. Unit Tests for Core Components

```
Priority: P0
Effort: 2-4 weeks

- Lexer unit tests: tokenize specific inputs, verify token stream
- Parser unit tests: parse fragments, verify tree structure
- Entity resolution tests: systematic coverage of all 2,231 entities
- Config parsing tests: all ~100 options with valid/invalid values
- String function tests: boundary conditions, empty strings, UTF-8
```

#### 3. API Compatibility Tests

```
Priority: P1
Effort: 2-3 weeks

- Systematic tests for all 173 public API functions
- Null/invalid argument handling
- Lifecycle tests (create, configure, parse, query, release)
- Callback tests (message, filter, option callbacks)
- Memory leak detection under test (Valgrind/ASan)
```

#### 4. Performance Benchmarks

```
Priority: P1
Effort: 1 week

- Benchmark suite with representative HTML documents
- Wall-clock time, peak memory, allocation count
- Run in CI to detect regressions
- Minimum: 10 documents of varying sizes (1KB to 10MB)
```

#### 5. Differential Testing (For Rewrite Phases)

```
Priority: P1 (during rewrite only)
Effort: Ongoing

- Run both C and Rust implementations on same input
- Compare output byte-for-byte
- Flag any behavioral difference for human review
- Use fuzzer-generated corpus for maximum coverage
```

#### 6. Property-Based Testing

```
Priority: P2
Effort: 2-3 weeks

- Generate random HTML documents
- Verify invariants:
  - Parsing never panics/crashes
  - Output is always valid (re-parseable)
  - Idempotency: tidy(tidy(x)) == tidy(x)
  - Round-trip: parse(serialize(parse(x))) == parse(x)
```

---

## Recommendation Matrix

| Factor | A: Incremental FFI | B: Clean-Room | C: Harden C | D: Hybrid |
|--------|:--:|:--:|:--:|:--:|
| **Safety improvement** | High | High | Medium | Medium-High |
| **Behavioral compatibility** | High | Low | Perfect | High |
| **C API preservation** | Yes | No | Yes | Yes |
| **Effort (person-months)** | 12-18 | 6-12 | 2-3 | 3-5 |
| **Risk of new bugs** | Medium | High | Low | Medium |
| **FFI complexity** | High | None | None | Medium |
| **Long-term maintainability** | High | High | Low | Medium |
| **Developer recruitment** | Better | Best | Worst | Mixed |
| **Community disruption** | Low | High | None | Low |
| **Prerequisite test work** | ~2 months | ~2 months | ~1 month | ~2 months |

### Decision Framework

**Choose Option C (Harden C) if:**
- The primary concern is security vulnerabilities
- Maintainer time is scarce
- You want the highest ROI per hour invested
- The contributor pool is comfortable with C

**Choose Option D (Hybrid) if:**
- You want targeted memory safety for the highest-risk code
- You can accept a two-language build system
- You have a contributor who knows both C and Rust
- You want to experiment before committing to a full rewrite

**Choose Option A (Incremental FFI) if:**
- You're committed to a full Rust transition long-term
- C API backward compatibility is essential
- You have at least one dedicated developer for 12+ months
- You accept that the FFI transition period will be painful

**Choose Option B (Clean-Room) if:**
- You're willing to break backward compatibility
- The goal is a modern, spec-compliant tool (not a clone of tidy)
- You're okay with behavioral differences from the current tidy
- You want to attract a new Rust contributor community

---

## Conclusion

### Is an incremental Rust rewrite feasible?

**Yes, technically feasible.** The codebase is well-structured with clear module boundaries. The public API uses opaque types, making it compatible with a hidden Rust implementation. The pluggable allocator and document-scoped design map naturally to Rust patterns.

### Is it worthwhile?

**It depends on the goal.**

If the goal is **immediate security**: Option C (hardening with fuzzing and sanitizers) delivers more safety per hour invested than any rewrite. A week of fuzzing will find more real bugs than months of Rust porting.

If the goal is **long-term maintainability and contributor recruitment**: A Rust rewrite (Option A or D) is more compelling. The C contributor pool for open-source projects is shrinking, and Rust's type system prevents categories of bugs from being introduced by future contributors.

If the goal is **a modern, spec-compliant HTML tool**: Option B (building on html5ever) is most appropriate, but it's really a new project, not a rewrite of this one.

### The strongest case against rewriting

The tidy-html5 codebase is 25 years old and has been used by millions. Its bugs are known, its edge cases are documented in 402 regression tests, and its behavior is what users expect. A rewrite — especially an incremental one with FFI boundaries — creates new categories of bugs that the existing code doesn't have. The research paper ["Rewrite it in Rust" Considered Harmful?](https://goto.ucsd.edu/~rjhala/hotos-ffi.pdf) documents real safety violations at FFI boundaries in production Rust rewrites of C code.

### The strongest case for rewriting

The codebase has real, unaddressed memory safety issues (buffer overflows in `strrep()`, unbounded allocations, global mutable state). These aren't theoretical — they're in the code today. Fixing them one at a time in C is a game of whack-a-mole; Rust prevents the entire category. Additionally, the project's long-term health depends on attracting contributors, and Rust is increasingly where systems programmers want to work.

### Recommended approach

**Start with Option C regardless.** Invest 2-3 months in:
1. Fixing known safety issues
2. Adding fuzzing and sanitizers to CI
3. Writing unit tests for core components
4. Adding performance benchmarks

This work is valuable no matter what. It makes the C code safer today AND creates the test infrastructure needed for any future rewrite. After this foundation is in place, evaluate whether to proceed with Option A or D based on contributor availability, user demand, and the severity of issues found during fuzzing.

---

## References

- [html5ever — Servo's HTML5 parser in Rust](https://github.com/servo/html5ever)
- [Cloudflare LOL HTML — Rust streaming HTML parser/rewriter](https://blog.cloudflare.com/html-parsing-2/)
- [Lessons learned from a successful Rust rewrite](https://gaultier.github.io/blog/lessons_learned_from_a_successful_rust_rewrite.html)
- [How Meta's Engineers Shifted a Billion-User Codebase from C to Rust](https://www.artiba.org/intelligent-engineering-at-scale/how-metas-engineers-shifted-a-billion-user-codebase-from-c-to-rust)
- ["Rewrite it in Rust" Considered Harmful? — Security Challenges at the C-Rust FFI](https://goto.ucsd.edu/~rjhala/hotos-ffi.pdf)
- [Integrating Rust and C++ in Firefox](https://manishearth.github.io/blog/2021/02/22/integrating-rust-and-c-plus-plus-in-firefox/)
- [Control what crosses FFI boundaries — Effective Rust](https://effective-rust.com/ffi.html)
- [RustMap: Towards Project-Scale C-to-Rust Migration](https://arxiv.org/html/2503.17741v1)
- [Compiling C to Safe Rust, Formalized](https://arxiv.org/pdf/2412.15042)
- [Rewriting in Rust: When It Makes Sense (Discord, Cloudflare, Amazon)](https://www.nandann.com/blog/rewriting-in-rust-when-it-makes-sense)
