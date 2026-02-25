# HTML Tidy: Incremental Rust Rewrite Feasibility Analysis

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Codebase Overview](#codebase-overview)
3. [Current Testing Infrastructure](#current-testing-infrastructure)
4. [Safety and Security Audit](#safety-and-security-audit)
5. [External Test Data and Other Implementations](#external-test-data-and-other-implementations)
6. [Option A: Incremental Rust Rewrite via FFI](#option-a-incremental-rust-rewrite-via-ffi)
7. [Option B: Clean-Room Rust Rewrite Using html5ever](#option-b-clean-room-rust-rewrite-using-html5ever)
8. [Option C: Harden the Existing C Codebase](#option-c-harden-the-existing-c-codebase)
9. [Option D: Hybrid — Rewrite Safety-Critical Paths Only](#option-d-hybrid--rewrite-safety-critical-paths-only)
10. [Option E: AI-Driven Ship of Theseus](#option-e-ai-driven-ship-of-theseus)
11. [Testing Strategy for Any Rewrite](#testing-strategy-for-any-rewrite)
12. [Recommendation Matrix](#recommendation-matrix)
13. [Conclusion](#conclusion)

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

## External Test Data and Other Implementations

A key insight for any rewrite: you are not the first project to parse, clean, or validate HTML. There is a wealth of external test suites and alternative implementations whose test data can be harvested to dramatically improve confidence before and during a rewrite.

### Test Suites to Harvest

#### 1. html5lib-tests (The Gold Standard)

**Repo**: [github.com/html5lib/html5lib-tests](https://github.com/html5lib/html5lib-tests)

This is the de facto standard conformance test suite for HTML5 parsing, used by html5ever (Rust/Servo), Python's html5lib, Go's golang.org/x/net/html, and many others.

| Category | Count | Format | Relevance |
|----------|-------|--------|-----------|
| Tokenizer tests | ~2,700+ | JSON | Direct: validate lexer token output |
| Tree construction | ~500+ files | Custom `.dat` format | Direct: validate parser DOM output |
| Entity tests | All named entities | JSON | Direct: validate `entities.c` replacement |
| Encoding detection | ~50 | Binary + JSON | Direct: validate `charsets.c` replacement |

**How to use it**: For each piece you replace, run the corresponding html5lib tests against both the C original and the new Rust code. Divergences must be intentional and documented. Note: tidy intentionally diverges from WHATWG in places (it's a cleanup tool, not a browser parser), so some divergences are expected and acceptable.

#### 2. JTidy / JTidy NG (Java Port)

**Repos**: [github.com/jtidy/jtidy](https://github.com/jtidy/jtidy), jtidy-ng fork

JTidy is a Java port of the original HTML Tidy. It diverged from the C version years ago but its test suite was built to match C Tidy's behavior at a point in time.

| What's Useful | Details |
|---------------|---------|
| Test cases | ~200 input/expected pairs matching tidy behavior |
| Configuration tests | Java equivalents of tidy config options |
| Edge case documentation | Comments in tests explain WHY tidy behaves certain ways |

**How to use it**: Cross-reference JTidy's test inputs with C tidy's output to find cases where behavior has drifted. These drifts document implicit behavioral decisions that are otherwise undocumented.

#### 3. W3C Nu HTML Checker (Validator)

**Repo**: [github.com/validator/validator](https://github.com/validator/validator)

The W3C's official HTML validator has extensive test suites for HTML5 conformance.

| What's Useful | Details |
|---------------|---------|
| Valid HTML5 documents | Large corpus of spec-compliant HTML |
| Invalid documents with expected errors | Maps to tidy's warning/error reporting |
| SVG/MathML in HTML test cases | Edge cases tidy must handle |

**How to use it**: Feed valid and invalid documents through both old and new tidy. The validator's expected-error metadata helps verify tidy's diagnostic output is correct.

#### 4. html5ever Test Suite

**Repo**: [github.com/servo/html5ever](https://github.com/servo/html5ever)

html5ever has its own test infrastructure beyond html5lib-tests:

| What's Useful | Details |
|---------------|---------|
| Tokenizer benchmarks | Performance data for Rust HTML tokenization |
| Fuzz corpus | Interesting inputs from historical fuzzing |
| Integration tests | How to test a Rust HTML parser properly |

**How to use it**: If you build on html5ever, these tests come "for free." If you write your own parser, use their fuzz corpus as additional input.

#### 5. DOMPurify / Sanitizer Test Suites

**Repo**: [github.com/cure53/DOMPurify](https://github.com/cure53/DOMPurify)

DOMPurify is an HTML sanitizer with hundreds of XSS-focused test cases in `test/fixtures/expect.mjs`.

| What's Useful | Details |
|---------------|---------|
| ~500+ XSS vectors | Malicious HTML that must be handled safely |
| Mutation XSS tests | HTML that changes meaning after round-tripping |
| Parser confusion tests | Inputs designed to confuse HTML parsers |
| SVG/MathML namespace confusion | Foreign content edge cases |

**How to use it**: These are stress tests for the lexer and parser. Feed them through tidy and verify no crashes or unexpected behavior. Essential for security validation.

#### 6. Web Platform Tests (WPT)

**Repo**: [github.com/web-platform-tests/wpt](https://github.com/web-platform-tests/wpt)

The official test suite for web platform features (~56,000 tests, 1.8M subtests).

| What's Useful | Details |
|---------------|---------|
| `html/syntax/parsing/` | Many auto-generated from html5lib-tests |
| `conformance-checkers/` | HTML documents designed for validator testing — valid/invalid pairs |
| `conformance-checkers/html-aria/` | ARIA attribute validation |
| Encoding tests | Character encoding detection and handling |

The `conformance-checkers/` directory is especially relevant — it contains HTML documents explicitly classified as valid (`*-isvalid.html`) or invalid (`*-novalid.html`), which maps directly to tidy's error-reporting functionality.

#### 7. MutaGen (12 Million Adversarial HTML Fragments)

**Repo**: [github.com/ias-tubs/HTML_parsing_differentials](https://github.com/ias-tubs/HTML_parsing_differentials)

Research project from TU Braunschweig that generated 12 million unique HTML fragments designed to trigger parsing differentials between browsers and sanitizers.

| What's Useful | Details |
|---------------|---------|
| 12M generated fragments | Stress testing at massive scale |
| Namespace confusion vectors | SVG/MathML integration edge cases |
| Mis-nested tag patterns | Adoption agency algorithm torture tests |
| Parser differential triggers | Inputs where different parsers disagree |

**How to use it**: Run a large sample through tidy to verify no crashes, infinite loops, or garbled output. The source code to regenerate fragments is available.

#### 8. OWASP Java HTML Sanitizer + AntiSamy

**Repo**: [github.com/OWASP/java-html-sanitizer](https://github.com/OWASP/java-html-sanitizer)

| What's Useful | Details |
|---------------|---------|
| AntiSamy test vectors | ~200+ XSS attack vectors (encoded JS, malicious stylesheets, base64 payloads) |
| Documented CVEs with test cases | CVE-2016-10006, CVE-2022-28367, CVE-2023-43643, etc. |
| Tag smuggling patterns | Real-world bypass attempts |

#### 7. Generating Tests from the C Implementation Itself

The most powerful source of test data is **the existing C tidy binary used as an oracle**:

```
For each module being replaced:
1. Identify all inputs that exercise that module
2. Run C tidy on each input, capture:
   - Output HTML (byte-exact)
   - Error/warning messages (byte-exact)
   - Exit code
   - Intermediate state (if instrumentable)
3. Store as golden test fixtures
4. Run Rust replacement on same inputs
5. Diff automatically
```

This can be done mechanically at massive scale by AI. Thousands of test cases can be generated by:
- Taking all 402 existing regression inputs
- Taking all html5lib-tests inputs
- Taking all DOMPurify test inputs
- Generating random HTML (property-based testing)
- Mutating existing inputs (fuzzer-style)

### Other Implementations Worth Studying

| Implementation | Language | Notes |
|---------------|----------|-------|
| [html5ever](https://github.com/servo/html5ever) | Rust | WHATWG-compliant parser. Study for Rust HTML parser patterns and the `TreeSink` trait. |
| [lol-html](https://github.com/cloudflare/lol-html) | Rust | Cloudflare's streaming HTML rewriter. Different architecture (no full DOM), but shows Rust HTML processing patterns. |
| [JTidy](https://github.com/jtidy/jtidy) | Java | Direct port of tidy (v1.0.5). ~200+ test cases in input/config/expected format — directly extractable. Divergences from C tidy are themselves interesting test data. |
| [Tidier](https://github.com/martinjw/Tidier) | C# | .NET port of TidyNet, updated to .NET 8 / HTML5. Another source of behavioral divergence data. |
| [HTML Purifier](https://github.com/ezyang/htmlpurifier) | PHP | Standards-compliant HTML filter. Its `MakeWellFormed` tests are directly relevant to tidy's repair behavior. |
| [Ammonia](https://github.com/rust-ammonia/ammonia) | Rust | HTML sanitizer built on html5ever. Shows Rust HTML cleanup patterns. |
| [scraper](https://github.com/causal-agent/scraper) | Rust | HTML querying on html5ever. Shows practical Rust DOM traversal patterns. |
| [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/) | Python | HTML cleanup tool. Different approach but ubiquitous. |

**Key insight**: No pure-Rust HTML tidy/cleanup tool exists in the Rust ecosystem. There are parsers (html5ever), sanitizers (Ammonia), and scrapers (scraper), but nothing that does what tidy does: repair + validate + pretty-print. A Rust tidy would fill a real gap.

### Test Data Priority for Extraction

| Priority | Source | Est. Tests | What It Validates |
|----------|--------|-----------|-------------------|
| 1 | tidy-html5 own tests | ~402 | Exact tidy behavior (ground truth) |
| 2 | html5lib-tests | ~9,000+ | HTML5 parser correctness (tokenizer + tree) |
| 3 | JTidy test resources | ~200+ | Tidy behavior from Java port (divergences = data) |
| 4 | MutaGen fragments | 12M generated | Parser robustness / crash resistance |
| 5 | DOMPurify vectors | ~500+ | Edge-case HTML handling / XSS vectors |
| 6 | WPT conformance-checkers | hundreds | Validator behavior (valid/invalid classification) |
| 7 | OWASP AntiSamy tests | ~200+ | Malicious input handling |

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

## Option E: AI-Driven Ship of Theseus

### Strategy

Use AI assistants (Claude, Gemini) to incrementally replace C modules with Rust, using the existing C binary as a black-box oracle for correctness. No human writes code. Each piece follows a cycle: **generate tests from the C oracle -> rewrite in Rust -> validate against oracle -> swap in**.

This is a **personal fork** approach — not PRs to the upstream htacg/tidy-html5 project. You maintain your own fork with a Rust implementation that is drop-in compatible.

### Why This Is Different from Options A-D

The key constraint changes everything: **you have AI labor that is cheap and abundant, but no human developer time**. This inverts the normal cost equation:

| Resource | Traditional | Your Situation |
|----------|------------|----------------|
| Developer time | Expensive, scarce | Zero (AI does it) |
| Test generation | Tedious, expensive | Cheap (AI + C oracle) |
| Code review | Essential | You review AI output |
| Build/CI setup | One-time cost | AI can do this |
| Debugging | Expensive | AI can iterate |

The main bottleneck shifts from "writing code" to "validating correctness" — and that's exactly what the oracle-based approach solves.

### The Oracle Pattern

```
┌──────────────────────────────────────────────────────────┐
│                    THE ORACLE CYCLE                       │
│                                                          │
│  1. Pick a module (e.g., entities.c)                     │
│  2. AI generates exhaustive tests by:                    │
│     a. Reading the C source to understand the contract   │
│     b. Running the C binary on crafted inputs            │
│     c. Harvesting html5lib-tests, DOMPurify, etc.        │
│     d. Generating edge cases via property-based testing  │
│  3. Tests capture: input → expected output (from C)      │
│  4. AI rewrites the module in Rust                       │
│  5. Run all tests against Rust implementation            │
│  6. AI fixes failures, iterates until green              │
│  7. Wire Rust module into the build (FFI shim)           │
│  8. Run full 402-case regression suite                   │
│  9. If green: commit. If red: AI investigates.           │
│  10. Pick next module.                                   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Piece-by-Piece Replacement Order

Ordered by **isolation** (how cleanly can this be swapped) and **testability** (how easily can the C oracle validate correctness):

```
Phase 1 — Pure Functions (ideal for AI, trivially testable)
  ┌─────────────────────────────────────────────────────┐
  │ entities.c (2,195 LOC)                               │
  │   Input: entity name string → Output: code + version │
  │   Input: code point → Output: entity name             │
  │   Testable: 100%. All 2,032 entities are a lookup     │
  │   table. AI generates a test per entity.              │
  │   Oracle: Call C EntityInfo() for every entity.       │
  │   External data: html5lib-tests/tokenizer/entities.test│
  │   has EVERY named character reference.                 │
  ├─────────────────────────────────────────────────────┤
  │ tmbstr.c (277 LOC)                                    │
  │   17 string functions (strcmp, strlen, etc.)           │
  │   Testable: 100%. Pure functions with simple I/O.     │
  │   Oracle: Call each C function with diverse inputs.   │
  │   Bonus: Most just wrap libc; Rust stdlib replaces.   │
  ├─────────────────────────────────────────────────────┤
  │ utf8.c (~300 LOC)                                     │
  │   UTF-8 encode/decode.                                │
  │   Testable: 100%. Rust's String handles this natively.│
  │   Oracle: Generate all edge-case codepoints.          │
  └─────────────────────────────────────────────────────┘

Phase 2 — Data Tables (large but mechanical)
  ┌─────────────────────────────────────────────────────┐
  │ attrdict.c (3,669 LOC)                               │
  │   Static table of HTML attribute definitions.         │
  │   Testable: 100%. It's just data.                    │
  │   Oracle: Dump the full table from C, compare.       │
  ├─────────────────────────────────────────────────────┤
  │ tags.c (2,080 LOC) + tags.h                          │
  │   Tag dictionary + tag properties.                    │
  │   Testable: 100%. Static data + lookup functions.    │
  │   Oracle: Query every tag, compare properties.       │
  └─────────────────────────────────────────────────────┘

Phase 3 — I/O Abstractions (clear interfaces)
  ┌─────────────────────────────────────────────────────┐
  │ buffio.c / fileio.c / streamio.c                     │
  │   Input/output stream abstractions.                   │
  │   Testable: High. Read bytes, write bytes.           │
  │   Oracle: Feed same bytes, verify same output.       │
  │   FFI: Rust Read/Write traits behind C callbacks.    │
  └─────────────────────────────────────────────────────┘

Phase 4 — Configuration & Messages (self-contained)
  ┌─────────────────────────────────────────────────────┐
  │ config.c (2,438 LOC)                                 │
  │   Testable: High. Parse config string, query values. │
  │   Oracle: Set each of ~100 options, query back.      │
  ├─────────────────────────────────────────────────────┤
  │ message.c / language.c / language_en.h                │
  │   Testable: High. Format message, compare string.    │
  │   Oracle: Trigger each message code, compare output. │
  └─────────────────────────────────────────────────────┘

Phase 5 — Analysis Passes (isolated, read-only)
  ┌─────────────────────────────────────────────────────┐
  │ access.c (3,540 LOC)                                 │
  │   WCAG accessibility checker.                         │
  │   Testable: High. 119 existing test cases.           │
  │   Oracle: Run C tidy with accessibility options.     │
  │   This module only READS the tree and emits messages.│
  │   Does NOT modify the DOM. Ideal for isolated rewrite│
  ├─────────────────────────────────────────────────────┤
  │ attrs.c (2,777 LOC)                                  │
  │   Attribute validation.                               │
  │   Testable: High. Given tag + attr, is it valid?     │
  │   Oracle: Craft HTML with each attr, check warnings. │
  └─────────────────────────────────────────────────────┘

Phase 6 — Pretty Printer (output-only, highly testable)
  ┌─────────────────────────────────────────────────────┐
  │ pprint.c (2,714 LOC)                                 │
  │   Testable: 100% via end-to-end output comparison.   │
  │   Oracle: Same input + config → must produce same    │
  │   output bytes. All 402 regression tests validate.   │
  │   This is the EASIEST module to validate because     │
  │   the existing test suite IS a pretty-printer test.  │
  └─────────────────────────────────────────────────────┘

Phase 7 — Cleanup Passes (tree transforms, one at a time)
  ┌─────────────────────────────────────────────────────┐
  │ clean.c (2,862 LOC)                                  │
  │   ~25 independent cleanup functions.                  │
  │   Each can be replaced individually.                  │
  │   Oracle: Enable one cleanup at a time, compare.     │
  │   Testable: High per-function, medium in combination.│
  └─────────────────────────────────────────────────────┘

Phase 8 — The Big One: Lexer + Parser (must do together)
  ┌─────────────────────────────────────────────────────┐
  │ lexer.c (4,548 LOC) + parser.c (6,450 LOC)          │
  │   These are tightly coupled. The lexer produces      │
  │   Nodes directly (not an abstract token stream),     │
  │   and the parser calls back into the lexer.          │
  │                                                      │
  │   Testable: Via the full pipeline. If every other    │
  │   module is already Rust, swapping the parser means  │
  │   the ONLY C code left is lexer+parser. The full     │
  │   regression suite validates end-to-end.             │
  │                                                      │
  │   External data: html5lib tree construction tests    │
  │   provide ~500+ expected parse trees. DOMPurify      │
  │   provides ~3,000+ adversarial inputs.               │
  │                                                      │
  │   This is the hardest phase but by now you have:     │
  │   - Thousands of oracle-generated test cases         │
  │   - All surrounding modules already in Rust          │
  │   - Deep understanding from rewriting everything else│
  └─────────────────────────────────────────────────────┘
```

### The Fork Question: Respecting Upstream

Since you don't want to antagonize the existing htacg maintainers:

**Do:**
- Fork the repository
- Clearly document that this is an experimental Rust reimplementation
- Keep the original license and attribution
- Link back to the upstream project
- If you find real bugs during oracle testing, submit those as issues/PRs to upstream (goodwill)
- Keep the C API compatible so your fork is a drop-in replacement

**Don't:**
- Submit massive "rewrite in Rust" PRs to the upstream project
- Claim your fork is "better" or "the official" version
- Break the existing C API without a compatibility shim

**Naming**: Consider naming the fork something distinct (e.g., `tidy-rs`, `html-tidy-oxide`) to make it clear it's a separate project that happens to be API-compatible.

### What AI Does Well (and Poorly) Here

**AI excels at:**
- Reading C code and understanding the contract
- Generating exhaustive test cases from source code analysis
- Translating algorithmic logic from C to idiomatic Rust
- Running tests, reading failures, iterating on fixes
- Mechanical work like translating 2,032 entity table entries
- Setting up build systems and CI

**AI struggles with:**
- The Node/tree structure design decision (see earlier analysis). This is an architectural choice that cascades everywhere. The human should decide the approach (arena? Rc<RefCell>? indextree?) and the AI implements it.
- Subtle behavioral differences that are "correct" by one interpretation but not another. The human needs to review divergences and decide which behavior is intended.
- Complex FFI boundary design. The interfaces between Rust and remaining C modules require careful thought about ownership, lifetimes, and error handling.

**Recommended human involvement:**
1. Decide the DOM tree representation strategy (biggest architectural call)
2. Review any test case where C and Rust diverge (is the divergence a bug fix or a regression?)
3. Decide when to break C API compatibility (if ever)
4. Final review of each phase before merging

### Advantages

- **Near-zero human code-writing effort**: AI generates tests, writes Rust, iterates on failures
- **Oracle ensures correctness**: The C binary is the ground truth. If the Rust output matches, it's correct by definition.
- **Incremental risk**: Each module swap is independently validated. If a module is too hard, skip it and come back later.
- **External test data supercharges confidence**: Combining oracle-generated tests with html5lib-tests, DOMPurify vectors, and JTidy test cases creates a test suite far more comprehensive than any human would write.
- **Respects upstream**: It's a fork, not a hostile takeover
- **Progressive value**: Even a partial rewrite (Phases 1-6) delivers value — the safety-critical lexer can stay in C while everything around it becomes Rust.

### Disadvantages

- **AI code quality varies**: Each module needs human review for idiomatic Rust, correct unsafe usage, and architectural consistency.
- **The Node structure problem remains**: This is the central design decision and it's not something you can oracle-test your way around. It needs a human architectural decision upfront.
- **Phase 8 (lexer+parser) is still hard**: Even with abundant test data, rewriting 11,000 lines of parser logic is complex. The AI may produce code that passes all tests but is unmaintainable.
- **FFI boundary during transition**: While C and Rust coexist, the FFI boundary is a source of bugs. AI-generated FFI code needs especially careful review.
- **Maintenance burden**: After the rewrite, you still need to maintain the Rust code. If it's only AI-maintained, code quality may drift.
- **Build system complexity**: CMake + Cargo during the transition phase is a real friction source.

### Estimated Timeline (AI-Assisted)

| Phase | Effort (AI sessions) | Human Review | Calendar Time |
|-------|---------------------|-------------|---------------|
| Phase 1: Pure functions | 2-3 sessions | Light | 1 week |
| Phase 2: Data tables | 1-2 sessions | Light | 1 week |
| Phase 3: I/O | 2-3 sessions | Medium | 1-2 weeks |
| Phase 4: Config/Messages | 3-5 sessions | Medium | 2-3 weeks |
| Phase 5: Analysis passes | 3-5 sessions | Medium | 2-3 weeks |
| Phase 6: Pretty printer | 3-5 sessions | Medium | 2-3 weeks |
| Phase 7: Cleanup | 5-8 sessions | Medium-High | 3-4 weeks |
| Phase 8: Lexer+Parser | 10-20 sessions | High | 4-8 weeks |
| **Total** | **~30-50 sessions** | | **~3-6 months** |

"AI session" = one focused conversation where AI generates tests, writes code, and iterates. Human reviews results and gives go/no-go.

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

| Factor | A: FFI Rewrite | B: Clean-Room | C: Harden C | D: Hybrid | E: AI Ship of Theseus |
|--------|:--:|:--:|:--:|:--:|:--:|
| **Safety improvement** | High | High | Medium | Medium-High | High |
| **Behavioral compatibility** | High | Low | Perfect | High | High (oracle-validated) |
| **C API preservation** | Yes | No | Yes | Yes | Yes (via shim) |
| **Human effort** | 12-18 months | 6-12 months | 2-3 months | 3-5 months | ~3-6 months review only |
| **AI suitability** | Medium | Medium | Low | Medium | **High** |
| **Risk of new bugs** | Medium | High | Low | Medium | Medium (oracle mitigates) |
| **FFI complexity** | High | None | None | Medium | High (during transition) |
| **Long-term maintainability** | High | High | Low | Medium | High (if reviewed) |
| **Community disruption** | Low | High | None | Low | None (fork) |
| **Prerequisite test work** | ~2 months | ~2 months | ~1 month | ~2 months | Built-in (test-first) |
| **External test leverage** | Can use | Central | Can use | Can use | **Central** |

### Decision Framework

**Choose Option E (AI Ship of Theseus) if:**
- You have no human developers who want to write code
- You have AI assistants (Claude, Gemini) available for multi-session work
- You're willing to review AI output but not write code yourself
- You want to maintain behavioral compatibility via oracle testing
- You want to do this as a personal fork without disturbing upstream
- You want the test-generation work to have value even if the rewrite stalls

**Choose Option C (Harden C) if:**
- You just want the existing C code to be safer, not rewritten
- You want the highest ROI per hour invested
- You're satisfied with C long-term

**Choose Option B (Clean-Room) if:**
- You're building a new tool, not cloning the old one
- You're willing to break backward compatibility
- You want to use html5ever and the WHATWG spec as the foundation

**Choose Option A or D if:**
- You have human Rust+C developers available
- You want to contribute the rewrite back to the upstream project

---

## Conclusion

### Is an incremental Rust rewrite feasible?

**Yes, technically feasible.** The codebase is well-structured with clear module boundaries. The public API uses opaque types, making it compatible with a hidden Rust implementation. The pluggable allocator and document-scoped design map naturally to Rust patterns.

### Is it worthwhile?

**It depends on the goal.**

If the goal is **immediate security**: Option C (hardening with fuzzing and sanitizers) delivers more safety per hour invested than any rewrite. A week of fuzzing will find more real bugs than months of Rust porting.

If the goal is **long-term maintainability and contributor recruitment**: A Rust rewrite (Option A or D) is more compelling. The C contributor pool for open-source projects is shrinking, and Rust's type system prevents categories of bugs from being introduced by future contributors.

If the goal is **a modern, spec-compliant HTML tool**: Option B (building on html5ever) is most appropriate, but it's really a new project, not a rewrite of this one.

If the goal is **an interesting project where AI does the work**: Option E is uniquely suited. The oracle pattern turns a risky rewrite into a series of testable, validatable steps where AI generates both the tests and the code, and the human reviews results.

### The strongest case against rewriting

The tidy-html5 codebase is 25 years old and has been used by millions. Its bugs are known, its edge cases are documented in 402 regression tests, and its behavior is what users expect. A rewrite — especially an incremental one with FFI boundaries — creates new categories of bugs that the existing code doesn't have. The research paper ["Rewrite it in Rust" Considered Harmful?](https://goto.ucsd.edu/~rjhala/hotos-ffi.pdf) documents real safety violations at FFI boundaries in production Rust rewrites of C code.

### The strongest case for rewriting

The codebase has real, unaddressed memory safety issues (buffer overflows in `strrep()`, unbounded allocations, global mutable state). These aren't theoretical — they're in the code today. Fixing them one at a time in C is a game of whack-a-mole; Rust prevents the entire category. Additionally, the project's long-term health depends on attracting contributors, and Rust is increasingly where systems programmers want to work.

### The honest case for "just do it as a project"

Not everything needs a rigorous cost-benefit justification. If you find the idea of an AI-assisted, oracle-validated, ship-of-Theseus Rust rewrite **interesting** — that's a legitimate reason to do it. The test-generation work has standalone value (contributing test cases back to upstream is pure goodwill). The process of having AI read, understand, test, and rewrite a 25-year-old C codebase is novel enough to be worth documenting. And even a partial rewrite (Phases 1-6, leaving the parser in C) produces a substantially safer codebase.

### Recommended approach for your constraints

Given: no human code-writing, AI labor available, fork not upstream, respect existing maintainers:

**Option E (AI Ship of Theseus)**, starting with Phase 1 (entities.c).

Why `entities.c` first:
1. It's a pure lookup table — 2,032 entities, each with name → (code, version)
2. The html5lib-tests repo has a complete entity test file you can harvest
3. Zero coupling to the DOM tree or parser state
4. The oracle test is trivial: call `EntityInfo("&amp;", false, &code, &versions)` for every entity, compare
5. If this works, you've proven the pattern. If it doesn't, you've lost almost nothing.

**After entities.c, do tmbstr.c** (string utilities). Same reasoning: pure functions, trivially testable, the AI can generate a test for every function with diverse inputs, and the oracle validates.

**The first two phases prove the pattern and build the tooling (build system, FFI shims, test harness). Everything after that is applying a proven process.**

### One key decision you must make first

Before any AI starts writing Rust: **how will the DOM tree be represented?**

This is the one architectural decision that cascades through the entire project. The options are:

1. **Arena + indices** (recommended): Nodes live in a `Vec<Node>` arena. "Pointers" are `NodeId(u32)` indices. Fast, safe, cache-friendly. Used by `indextree`, `ego-tree`, and similar Rust crates. Downside: every function needs access to the arena.

2. **`Rc<RefCell<Node>>`**: Standard Rust shared ownership. More ergonomic but slower, and panics at runtime if you violate borrow rules. Can't easily have mutable parent + child access.

3. **Raw pointers behind unsafe**: Fastest, most C-like, but defeats much of the purpose of using Rust.

This decision doesn't need to be made for Phases 1-4 (they don't touch the tree). But it must be decided before Phase 5 (analysis passes that read the tree). The AI can prototype both approaches on a small subtask and you can evaluate which feels right.

---

## References

### Rust Rewrite Case Studies
- [Lessons learned from a successful Rust rewrite](https://gaultier.github.io/blog/lessons_learned_from_a_successful_rust_rewrite.html)
- ["Rewrite it in Rust" Considered Harmful? — Security Challenges at the C-Rust FFI](https://goto.ucsd.edu/~rjhala/hotos-ffi.pdf)
- [Rewriting in Rust: When It Makes Sense (Discord, Cloudflare, Amazon)](https://www.nandann.com/blog/rewriting-in-rust-when-it-makes-sense)
- [RustMap: Towards Project-Scale C-to-Rust Migration](https://arxiv.org/html/2503.17741v1)
- [Compiling C to Safe Rust, Formalized](https://arxiv.org/pdf/2412.15042)

### HTML Parsers and Tools
- [html5ever — Servo's HTML5 parser in Rust](https://github.com/servo/html5ever)
- [lol-html — Cloudflare's streaming HTML rewriter](https://github.com/cloudflare/lol-html)
- [Ammonia — Rust HTML sanitizer](https://github.com/rust-ammonia/ammonia)
- [JTidy — Java port of HTML Tidy](https://github.com/jtidy/jtidy)
- [Tidier — .NET port of TidyNet](https://github.com/martinjw/Tidier)
- [HTML Purifier — PHP HTML filter](https://github.com/ezyang/htmlpurifier)

### Test Suites
- [html5lib-tests — De facto standard HTML parser test suite](https://github.com/html5lib/html5lib-tests)
- [Web Platform Tests (WPT)](https://github.com/web-platform-tests/wpt)
- [WPT conformance-checkers](https://github.com/web-platform-tests/wpt/tree/master/conformance-checkers)
- [Nu HTML Checker (W3C Validator)](https://github.com/validator/validator)
- [DOMPurify — XSS sanitizer with extensive test vectors](https://github.com/cure53/DOMPurify)
- [MutaGen — 12M adversarial HTML fragments](https://github.com/ias-tubs/HTML_parsing_differentials)
- [OWASP Java HTML Sanitizer](https://github.com/OWASP/java-html-sanitizer)
- [AntiSamy — Java HTML sanitizer with CVE test vectors](https://github.com/nahsra/antisamy)
