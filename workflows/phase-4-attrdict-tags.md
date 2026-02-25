# Workflow: Phase 4 — attrdict.c + tags.c

**Objective**: Replace HTML attribute and tag dictionaries with Rust.

**Prerequisites**: Phase 0 and Phase 2 (tmbstr) complete.

**Key files**:
- `reference/src/attrdict.c` — attribute definitions and lookup
- `reference/src/attrs.h` — attribute type declarations
- `reference/src/tags.c` — tag definitions, lookup, parser fn pointers
- `reference/src/tags.h` — tag type declarations

**Critical decision**: tags.c contains function pointers to parser.c
functions. This must be resolved before starting.

---

## Before You Start

1. Read [AGENTS.md](../AGENTS.md)
2. Read [ROADMAP.md](../ROADMAP.md) — Phase 0 and Phase 2 must be `[x]`
3. Read [LEARNINGS.md](../LEARNINGS.md) — all prior phase notes
4. Mark Phase 4 as `[~]` in ROADMAP.md

---

## Decision: Parser Function Pointers

**THIS MUST BE DECIDED AND RECORDED IN LEARNINGS.md BEFORE CODING.**

The tag dictionary contains entries like:
```c
{ TidyTag_A, "a", VERS_ALL, ..., TY_(ParseInline), CheckA }
```

`TY_(ParseInline)` is a function pointer into parser.c, which is still
C at this point. Options:

**Option 1** (Recommended): Store `extern "C" fn` pointers in Rust
```rust
pub type ParserFn = unsafe extern "C" fn(*mut TidyDoc, *mut Node, GetTokenMode) -> ();
```

**Option 2**: Keep tag table in C, only rewrite lookup functions

**Option 3**: Mapping from tag ID to function pointer, C side

Pick one, document rationale in LEARNINGS.md, then proceed.

---

## Step 1: Understand the C Code

Read:
```
reference/src/attrdict.c   # Attribute dictionary
reference/src/tags.c       # Tag dictionary + lookup
reference/src/attrs.h      # Attribute structs
reference/src/tags.h       # Tag structs, Dict entry type
```

Key things:
- How many attributes are defined?
- How many tags are defined?
- What does the lookup function do (hash table? linear scan?)
- What fields does each tag entry have?
- Which functions from attrdict.c and tags.c are called by other modules?

Map all call sites with grep for `TY_(` function names.

---

## Step 2–6: Follow Standard Phase Pattern

Follow the same pattern as prior phases:
1. Generate oracle tests for all lookup functions
2. Implement Rust module(s)
3. Write FFI wrappers
4. Wire into build
5. Run 402 regression tests

### Extra concern for this phase

The tag dictionary is LARGE and touches many parts of the codebase.
A lookup returning the wrong tag ID or parser function will cause
cascading failures. Test lookups exhaustively.

---

## Definition of Done

- [ ] Attribute dictionary implemented in Rust
- [ ] Tag dictionary implemented in Rust
- [ ] Parser function pointer decision documented in LEARNINGS.md
- [ ] All lookups oracle-tested
- [ ] All 402 regression tests pass
- [ ] ROADMAP.md and LEARNINGS.md updated
