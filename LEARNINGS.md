# Learnings & Progress Journal

Shared knowledge base for all agents working on tidy-oxide. Every agent
MUST read this before starting work and MUST add an entry after
completing work.

**How to use this file**:
- Read the whole file before starting your phase
- Add your entry at the TOP of the journal (newest first)
- Use the template below
- Be honest about what went wrong — that's the most valuable part
- Include specific details: error messages, linker flags, code patterns

---

## Entry Template

Copy this when adding a new entry:

```markdown
### [Phase X / Task] — YYYY-MM-DD

**Agent**: [agent name/model]
**Duration**: [rough time estimate]
**Status**: [completed / partial / blocked]

#### What was done
- ...

#### What worked well
- ...

#### What was difficult or surprising
- ...

#### Mistakes made (so others can avoid them)
- ...

#### Key technical details
- ...

#### Tips for the next agent
- ...

#### Open questions
- ...
```

---

## Journal Entries

*(Newest first — add your entry here)*

---

### Planning Phase — 2026-02-25

**Agent**: Claude (Opus)
**Duration**: ~1 session
**Status**: Completed

#### What was done
- Full codebase analysis of tidy-html5 (~58,000 LOC)
- Created 5 analysis/planning documents
- Identified module dependency graph and phase ordering
- Identified entities.c as the true leaf module (not tmbstr.c)

#### What worked well
- Using grep to trace `TY_()` call sites gave accurate dependency data
- Reading `forward.h` early was essential — it has the TY_ macro, allocator
  types, and all the key typedefs

#### What was difficult or surprising
- **tmbstr.c is NOT a leaf**: Initial assumption was wrong. It's called from
  19 of 26 source files. entities.c (3 call sites) is the real first target.
- **TY_ macro**: `#define TY_(str) TYDYAPPEND(prvTidy,str)` — all internal
  symbols get `prvTidy` prefix. This is critical for FFI naming.
- **TidyAllocator is pervasive**: 51 calls across 15 files. Any module that
  allocates memory must use the C allocator vtable, not Rust's allocator.
- **No unit tests exist**: Only 402 integration regression tests (Ruby runner).
  No fuzzing, no sanitizers. Testing infrastructure must be built from scratch.
- **Build system**: CMake-based. 26 .c files compiled into libtidy.

#### Key technical details
- Entity lookup: binary search by first char, then linear scan
- ToLower table in lexer.h is NOT standard ASCII tolower — custom 256-byte table
- `strrep()` in tmbstr.c has a buffer overflow (writes past buffer end)
- Global mutable state in `alloc.c` (default allocator uses globals)
- The tag dictionary has parser function pointers embedded in each entry

#### Tips for the next agent
- Always read `forward.h` first — it's the Rosetta Stone for the codebase
- Use `nm` on C object files to get exact symbol names for FFI
- The regression test runner is in `regression_testing/test.rb` (Ruby + Thor)
- html5lib-tests repo has entity validation data at `tokenizer/entities.json`

#### Open questions
- Will the Ruby test runner work in the build environment?
- What's the exact set of linker flags needed to link Rust .a with C?
- Does the C reference binary need to be built with specific flags for
  oracle testing (e.g., debug symbols, specific optimization level)?

---

## Index of Key Discoveries

Quick-reference for common questions. Update as new entries are added.

### Build System
- C build: CMake (`reference/CMakeLists.txt`)
- 26 source files compiled
- Linker flags for Rust: TBD (Phase 0 will discover)

### FFI / Symbol Names
- Internal symbols use `prvTidy` prefix via `TY_()` macro
- Verify with: `nm <object-file> | grep prvTidy`
- Bool is `c_int` (0 = no, 1 = yes)

### The Allocator
- `TidyAllocator` has a vtable: `alloc`, `realloc`, `free`, `panic`
- Defined in `forward.h`
- Used by: tmbstr, config, tags, streamio, lexer, parser, clean, ...
- NOT used by: entities, utf8 pure functions

### The ToLower Table
- In `lexer.h`, NOT standard ASCII
- 256-byte lookup, ASCII-only case folding
- Must be reproduced exactly in Rust for tmbstr functions

### Known C Bugs
- `strrep()` buffer overflow in tmbstr.c — no bounds checking
- Document others here as discovered

### External Test Data
- html5lib-tests: entity data, tokenizer tests, tree construction
- Markus Kuhn UTF-8 stress test
- DOMPurify XSS vectors
- tidy's own 402 regression tests
