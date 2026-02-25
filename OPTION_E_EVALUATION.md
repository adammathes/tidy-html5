# Option E: AI-Driven Ship of Theseus — Honest Evaluation

## Goal

> "I want to see how far AI-driven rewriting can go."

This isn't primarily about producing a better tidy. It's an experiment in
AI-assisted C-to-Rust translation on a real, non-trivial codebase. The tidy
binary is the subject, not the product.

---

## The Hard Truths

### Truth 1: "Leaf modules" aren't actually leaves

The analysis document presents `entities.c`, `tmbstr.c`, and `utf8.c` as
isolated, trivially swappable modules. Let's look at reality:

**`tmbstr.c`** — 17 string functions. Sounds simple. But:

- Called from **19 of 26 source files** in the project
- `tmbstrdup()` and `tmbstrndup()` take a `TidyAllocator*` parameter —
  they allocate memory through tidy's custom allocator vtable
- `tmbstrcasecmp()` calls `TY_(ToLower)()` which is defined as a macro
  in `lexer.h` that indexes into a static character type table
- `strrep()` uses raw `strcpy`/`strcat` into a stack buffer and is called
  from `language.c`

You can't just swap in Rust `String` operations. You need to:
1. Match the exact `TidyAllocator` interface (Rust functions must allocate
   via the C allocator, not Rust's global allocator)
2. Match the `TY_(ToLower)` behavior exactly (it's ASCII-only, not Unicode)
3. Expose 17 C-callable functions with identical signatures
4. Handle the case where callers pass `NULL` (the C code checks for this)

This is **doable** but it's not "trivially testable pure functions." It's
17 FFI shims that must participate in C's memory management.

**`entities.c`** — This one actually IS close to trivially swappable:

- Only 2 exported functions: `EntityInfo()` and `EntityName()`
- Only 3 call sites: `lexer.c:1251`, `pprint.c:1270`, `pprint.c:1324`
- Uses `tmbstrcmp()` internally (only dependency besides the static table)
- No allocator interaction (pure lookups on a const table)
- The entity table is 2,032 entries of `{name, version_flags, codepoint}`

**Verdict**: entities.c is the real leaf. tmbstr.c looks like a leaf but has
tendrils everywhere.

**`utf8.c`** — Medium complexity:

- Encode/decode functions are pure logic (good)
- But `DecodeUTF8BytesToChar()` takes a `TidyInputSource*` parameter —
  it reads bytes from an I/O abstraction, not just from a buffer
- Called from 8 files including `lexer.c` and `streamio.c`

### Truth 2: The TY_ macro makes FFI non-trivial

Every internal function is name-mangled:

```c
#define TY_(str) TYDYAPPEND(prvTidy,str)
// So TY_(EntityInfo) becomes prvTidyEntityInfo
```

This is a C visibility trick to prevent symbol clashes. For FFI, you need
Rust functions that export symbols with these exact mangled names, or you
need to change the C call sites. Either way, it's not just "write the Rust
function and link it."

### Truth 3: The allocator is pervasive

51 calls to `TidyAlloc`/`TidyRealloc`/`TidyFree` across 15 files. Every
Rust module that allocates memory must either:

a) Use Rust's own allocator (creates a dual-allocator situation where some
   memory is Rust-owned and some is C-owned — a recipe for use-after-free
   at the boundary), or

b) Route all allocations through the C `TidyAllocator` vtable (requires
   implementing a custom Rust `GlobalAllocator` that calls into C — ugly
   but correct), or

c) Only work with stack/static data (limits what you can do)

For the entity table, (c) works fine — it's all `const`. For string
functions, you need (b). For anything that creates `Node` structures,
you need (b) and it gets much harder.

### Truth 4: The build system change is real friction

The CMake file compiles 26 `.c` files into a static/shared library. Adding
Rust means:

1. Adding `corrosion` or `cmake-cargo` to CMakeLists.txt
2. Creating a Rust crate that builds a static `.a` library
3. Linking the Rust `.a` into the C library build
4. Handling platform differences (Windows MSVC, MinGW, macOS, Linux)
5. CI must install Rust toolchain on all platforms

This is a one-time cost but it's also the first thing that must work before
any Rust code can be tested in context. It's boring, fiddly, platform-specific
work that AI can partially do but will likely need human debugging on at
least one platform.

---

## Revised Phase Plan (Honest Version)

### Phase 0: Infrastructure (MUST DO FIRST)

**What**: Set up Rust in the build system, prove a trivial Rust function
can be called from C code, run in CI.

**Concrete deliverable**: A Rust crate with one function:

```rust
#[no_mangle]
pub extern "C" fn rust_tidy_test() -> i32 { 42 }
```

Called from a C test file, built by CMake, runs on Linux/macOS/Windows CI.

**Estimated AI effort**: 1-3 sessions (mostly fighting CMake + platform issues)

**Risk**: Medium. CMake + Cargo integration is well-documented but
platform edge cases are painful. Windows MSVC is the most likely problem.

**Why first**: If this doesn't work, nothing else matters. This is the
"can we actually do this" gate.

### Phase 1: entities.c (The Proof of Concept)

**What**: Rewrite the entity lookup table and its 2 functions in Rust.

**The actual work**:
1. Translate the 2,032-entry static entity table to a Rust `const` array
   or `phf` (perfect hash function) map
2. Implement `EntityInfo(name, isXml, &code, &versions)` — string lookup
   returning code point and version bitmask
3. Implement `EntityName(code, versions)` — reverse lookup, code to name
4. Export both as `#[no_mangle] extern "C"` functions with the mangled
   `prvTidy` prefix
5. Handle the numeric entity parsing (&#decimal; and &#xhex;) that's
   currently in `EntityInfo()`
6. Remove `entities.c` from CMake, link Rust instead

**Test strategy**:
- Extract all 2,032 entity names from the C source
- For each: call C `EntityInfo("&name;", false, ...)` and capture result
- For each code point: call C `EntityName(code, VERS_ALL)` and capture
- Also harvest html5lib `entities.json` test data
- Run identical calls against Rust implementation, diff

**Estimated AI effort**: 1-2 sessions

**Risk**: Low. This is genuinely isolated. The only dependency is
`tmbstrcmp()` which can be replaced with Rust's native string comparison.
The numeric entity parsing uses `sscanf` which needs care but is
straightforward.

**What this proves**: That the build system works, FFI works, and the
oracle testing pattern works. If this phase succeeds cleanly, the
approach is validated for simple modules.

### Phase 2: tmbstr.c (The First Hard One)

**What**: Replace 17 string utility functions with Rust implementations.

**The actual work**:
1. Each function must be `extern "C"` with the exact same signature
2. Functions that allocate (`tmbstrdup`, `tmbstrndup`) must call through
   the `TidyAllocator` — this means the Rust code receives a C vtable
   pointer and calls `alloc` through it
3. `tmbstrcasecmp` and `tmbstrtolower/upper` must use the exact same
   case-mapping as the C code (ASCII-only, using the `IsUpper`/`IsLower`
   macros from `lexer.h` which use a lookup table)
4. `strrep` (the buffer-overflow-prone one) should be rewritten safely
   but must produce identical output for non-overflowing inputs
5. `tmbvsnprintf`/`tmbsnprintf` wrap C's `vsnprintf` — these probably
   stay as thin wrappers calling libc, there's no benefit to rewriting
   `printf` in Rust

**Test strategy**:
- For each of the 17 functions, generate test cases:
  - Empty strings, NULL pointers, single chars, long strings
  - Non-ASCII bytes (Latin-1 range), UTF-8 sequences
  - Boundary conditions (size=0, size=1, size=MAX)
- Call C version, capture output, compare with Rust version

**Estimated AI effort**: 2-4 sessions

**Risk**: Medium. The allocator interaction is the tricky part. Getting
the `TidyAllocator` vtable to work from Rust requires `unsafe` code that
correctly dereferences C function pointers. One wrong lifetime and you
get a segfault that's hard to debug.

**What this proves**: That Rust can participate in C's memory management
system. This is the gate for all subsequent phases.

### Phase 3: utf8.c (Encoding)

**What**: Replace UTF-8 encode/decode and UTF-16 surrogate handling.

**The actual work**:
1. `DecodeUTF8BytesToChar` — takes a first byte + either a buffer or a
   `TidyInputSource` callback and decodes a UTF-8 sequence. The callback
   variant reads bytes one at a time through a C function pointer.
2. `EncodeCharToUTF8Bytes` — takes a codepoint and either fills a buffer
   or writes through a `TidyOutputSink` callback.
3. `GetUTF8`/`PutUTF8` — convenience wrappers
4. Surrogate pair functions — pure arithmetic, trivial

**The catch**: Functions 1-2 accept I/O callbacks (`TidyInputSource`,
`TidyOutputSink`). The Rust code must call through C function pointers
to read/write bytes. This is another `unsafe` boundary.

**Test strategy**:
- All valid UTF-8 sequences for codepoints 0x0000-0x10FFFF
- All invalid sequences from Markus Kuhn's UTF-8 stress test
- Overlong encodings (must reject)
- Surrogate pair round-trip tests

**Estimated AI effort**: 2-3 sessions

**Risk**: Medium. The I/O callback interaction is the hard part, not the
UTF-8 logic itself (which Rust handles natively).

### Phase 4: attrdict.c + tags.c (Data Tables)

**What**: Replace the static HTML attribute and tag dictionaries.

**Why harder than it looks**: These aren't just lookup tables. They contain
function pointers:

```c
// Each tag entry has a parser function and a content checker function
static const Dict tag_defs[] = {
    { TidyTag_A, "a", VERS_ALL, ..., TY_(ParseInline), CheckA },
    ...
};
```

The tag dictionary maps tag names to **parser functions** — C function
pointers that implement tag-specific parsing behavior. You can't translate
these to Rust without also translating the parser functions they point to,
or keeping them as `extern "C"` references.

**Estimated AI effort**: 3-5 sessions

**Risk**: Medium-High. The function pointer issue means this isn't purely
mechanical data translation.

### Phase 5-6: config.c, message.c, language.c

**What**: Configuration parsing, message formatting, localization.

**Why medium**: config.c has its own option table with ~100 entries,
each with a type, default value, parser, and serializer. message.c formats
diagnostic messages using the language strings. These are self-contained
but interact with the `TidyDoc` structure extensively.

**Estimated AI effort**: 5-8 sessions total

**Risk**: Medium. Lots of string formatting, option parsing, and interaction
with the document structure through pointers.

### Phase 7: access.c, attrs.c (Tree Readers)

**What**: Accessibility checker and attribute validator.

**Why this is the wall**: These modules traverse the `Node` tree. In C:

```c
for (node = body->content; node; node = node->next) {
    if (nodeIsA(node)) {
        for (av = node->attributes; av; av = av->next) { ... }
    }
    if (node->content) CheckAccessibility(doc, node->content);
}
```

In Rust, to read C `Node*` pointers, you need `unsafe` blocks everywhere.
Every `node->next`, `node->content`, `node->attributes` dereference is
unsafe because you're following raw C pointers.

**The options**:
1. Write the Rust code with pervasive `unsafe` (works but defeats purpose)
2. Write a safe Rust wrapper that converts the C tree to a Rust tree on
   entry and converts back on exit (expensive, but makes the actual logic
   safe)
3. Wait until the tree itself is Rust (Phase 8) then rewrite these

Option 3 is the honest recommendation. Don't rewrite tree-reading code
until the tree itself is Rust.

**Estimated AI effort**: 5-8 sessions (if done with pervasive unsafe)

**Risk**: High. Lots of pointer chasing through C structures from Rust.

### Phase 8: pprint.c (Pretty Printer)

**What**: Output formatting — the most testable module.

**Same wall as Phase 7**: The pretty printer walks the `Node` tree to
produce output. Same pervasive `unsafe` issue.

**But**: It's the most testable module because every one of the 402
regression tests is effectively a pretty-printer test. The oracle
validation is trivially automated.

**Estimated AI effort**: 5-10 sessions

**Risk**: Medium-High (same tree walking issue, but excellent test coverage).

### Phase 9: clean.c (Cleanup Passes)

**What**: Tree transformation — insert/remove/modify nodes.

**Why hardest before parser**: Not only reads the tree but **mutates** it.
Creates new nodes, re-parents nodes, removes nodes. In Rust with C
pointers, this means `unsafe` code that modifies linked list pointers
through raw pointers. This is exactly the class of code that causes
use-after-free bugs in C, and doing it in `unsafe` Rust is no safer.

**Estimated AI effort**: 5-10 sessions

**Risk**: High.

### Phase 10: lexer.c + parser.c (The Boss Fight)

**What**: 11,000 lines of tightly coupled tokenizer and parser.

**Why it's a single unit**: The lexer doesn't produce a clean token stream.
It creates `Node` objects directly, manipulates the `Lexer` struct's
internal state (pushback buffer, inline stack, line/column tracking), and
the parser calls *back into* the lexer (`GetToken`, `UngetToken`). They
share mutable state through the `TidyDocImpl` struct.

**If everything else is Rust**: By this point, the ONLY C code is lexer +
parser. All the code they call (entities, strings, tags, config) is Rust.
So the FFI boundary is now reversed — Rust is the majority, C is the
island. This is actually the right time to do it, because you can define
the Rust tree structure and have the C parser populate it through FFI, then
in a second pass replace the C parser.

**Or**: This is where you stop and say "Phase 1-6 in Rust, lexer+parser
stays in C." That's a legitimate outcome. The parser is the most tested
part of the code (every regression test exercises it) and it's where 25
years of edge-case handling lives.

**Estimated AI effort**: 15-40 sessions (wide range because unpredictable)

**Risk**: Very high. This is the make-or-break phase.

---

## Cost Estimate (Revised, Honest)

### Scenario A: Stop After Phase 3 (entities + strings + utf8)

The "proof of concept" — validates the approach, replaces the lowest-risk
code, gets Rust into the build.

| Item | Sessions | API Cost (Opus) | API Cost (Sonnet) | Max Plan |
|------|----------|----------------|-------------------|----------|
| Phase 0: Build setup | 2 | $30 | $6 | incl |
| Phase 1: entities.c | 1 | $15 | $3 | incl |
| Phase 2: tmbstr.c | 3 | $45 | $9 | incl |
| Phase 3: utf8.c | 2 | $30 | $6 | incl |
| **Total** | **8** | **$120** | **$24** | **~1 month** |

**What you get**: 3 modules in Rust (~2,800 LOC replaced), the buffer
overflow in `strrep()` is fixed, a test suite for these modules, and
proof the pattern works.

**What you learn**: Whether AI can handle the FFI boundary, allocator
interaction, and build system integration.

### Scenario B: Through Phase 6 (everything except tree-walking code)

Replaces all "leaf" modules — data tables, I/O, config, messages.
Stops before any code that traverses the `Node` tree.

| Item | Sessions | API Cost (Opus) | API Cost (Sonnet) | Max Plan |
|------|----------|----------------|-------------------|----------|
| Scenario A | 8 | $120 | $24 | incl |
| Phase 4: Data tables | 4 | $80 | $16 | incl |
| Phase 5-6: Config/Msgs | 7 | $140 | $28 | incl |
| **Total** | **19** | **$340** | **$68** | **~2-3 months** |

**What you get**: ~12,000 LOC replaced. Everything that doesn't touch
the DOM tree is Rust. The library still works identically.

**What you learn**: Whether AI can handle complex FFI with function pointers,
config parsing, and string formatting across the boundary.

### Scenario C: Full Rewrite (including parser)

The "see how far it goes" scenario.

| Item | Sessions | API Cost (Opus) | API Cost (Sonnet) | Max Plan |
|------|----------|----------------|-------------------|----------|
| Scenario B | 19 | $340 | $68 | incl |
| Phase 7-9: Tree code | 18 | $400 | $80 | incl |
| Phase 10: Lexer+Parser | 25 | $625 | $125 | incl |
| **Total** | **62** | **$1,365** | **$273** | **~5-8 months** |

**What you get**: A fully Rust tidy (with C API shim). ~58,000 LOC replaced.

**What you learn**: Whether AI can rewrite a complex parser. This is the
genuinely interesting research question.

**Honest probability of full completion**: ~40-60%. Phases 1-6 will almost
certainly work. Phases 7-9 will probably work with ugly `unsafe` code.
Phase 10 is a coin flip — the AI might produce something that passes
tests but is unmaintainable, or it might get stuck in iteration loops on
edge cases.

### Cost Per Session Estimate

"Session" = one focused Claude Code or API conversation, heavy on file
reading, code generation, and build/test iteration.

| Model | Avg Input Tokens | Avg Output Tokens | Cost/Session |
|-------|-----------------|-------------------|-------------|
| Opus (API) | 200K | 40K | ~$6-20 |
| Sonnet (API) | 200K | 40K | ~$1-4 |
| Claude Code Max ($100/mo) | varies | varies | ~$100/month flat |
| Claude Code Max ($200/mo) | varies | varies | ~$200/month flat |
| Gemini 2.5 Pro (API) | 200K | 40K | ~$2-5 |

**If using Claude Code subscription**: The flat-rate plan is almost
certainly the better deal. You'd burn through API credits fast with the
amount of code reading and iteration required.

**Splitting across models**: Use Sonnet/Gemini for mechanical phases
(entity table translation, test generation), Opus for architecture
decisions and complex FFI.

---

## What Could Go Wrong

### 1. The build system doesn't cooperate on Windows

CMake + Cargo on Windows MSVC is the least-tested combination. The upstream
project's CI includes Windows builds. If your Rust integration breaks
Windows, that's a regression.

**Mitigation**: Start with Linux-only. Add platforms incrementally.

### 2. The allocator boundary causes subtle bugs

If Rust code allocates memory with Rust's allocator but the C code later
calls `TidyFree()` on it (which routes through the C allocator), you get
undefined behavior — probably a crash, possibly corruption.

**Mitigation**: Religiously use the C allocator for anything that crosses
the boundary. Have a test that allocates in Rust and frees in C.

### 3. The test oracle doesn't catch behavioral differences

The oracle compares final output. If a Rust module produces the same final
output but with different intermediate state (e.g., different memory layout,
different traversal order), it passes the oracle but might break when
composed with other replaced modules.

**Mitigation**: Test at module boundaries (not just end-to-end) wherever
possible. For entities and strings, this is easy. For tree-walking code,
it's hard.

### 4. AI generates code that passes tests but is unmaintainable

The AI might produce Rust code that works but has:
- Pervasive `unsafe` that provides no real safety benefit over C
- Non-idiomatic patterns (e.g., raw pointer arithmetic instead of iterators)
- No documentation of invariants or safety contracts
- Reimplemented std library functionality

**Mitigation**: Human review after each phase. But you said you don't want
to write code — so review is "does this look reasonable" not "let me fix
this." If the code is ugly but correct, that might be acceptable for the
experiment.

### 5. The project stalls at Phase 7 (tree walking)

This is the most likely failure mode. Phases 1-6 go well, you hit the
tree-walking code, the `unsafe` is pervasive and ugly, and you lose
motivation because the Rust code isn't actually safer than the C.

**Mitigation**: Decide upfront whether you're okay with this outcome. If
Phases 1-6 working is a success, define it as such before starting.

---

## What This Experiment Actually Tests

Forget the tidy binary for a moment. The interesting questions are:

1. **Can AI autonomously do C-to-Rust FFI integration?** Phase 0-1 answers
   this. Build system, symbol naming, calling conventions, linking.

2. **Can AI handle C's memory management from Rust?** Phase 2 answers this.
   Custom allocators, NULL handling, exact behavioral matching.

3. **Can AI generate comprehensive tests from C source code?** Every phase
   tests this. The oracle pattern is the novel part.

4. **Where does the approach break down?** Phase 7+ will find the limits.
   Tree-walking code with raw pointers is where "safe Rust" becomes
   "unsafe Rust with extra syntax."

5. **Is the oracle pattern sufficient for correctness?** If the Rust code
   passes all oracle tests but has a memory safety bug that the oracle can't
   detect (because it only checks output), that's an important finding.

---

## Recommendation

Start with Scenario A (Phases 0-3). Total cost: ~$120 in API credits or
~1 month of Claude Code Max.

If it works cleanly, continue to Scenario B (Phases 4-6). Another ~$220
or ~1-2 months.

At that point, you'll have replaced ~12,000 lines of C with Rust, fixed
real bugs, built a comprehensive test suite, and learned exactly where
the approach hits its limits. You'll also have a concrete artifact to
write about.

Whether to attempt Phase 10 (the parser) should be a separate decision
made after Phases 1-6 are complete. By then you'll have much better
intuition for what AI can and can't do with this codebase.

**The first session should be Phase 0**: get Rust into the CMake build and
prove a trivial FFI call works. Everything else depends on that.
