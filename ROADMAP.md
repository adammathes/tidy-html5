# Roadmap — tidy-oxide

Machine-readable progress tracker. Agents: update this file after
completing any phase or task. Humans: read this for project status.

**Last updated**: 2026-02-25
**Updated by**: claude (planning phase)

---

## Status Legend

| Marker | Meaning |
|--------|---------|
| `[ ]`  | Not started |
| `[~]`  | In progress |
| `[x]`  | Completed |
| `[!]`  | Blocked — see notes |

---

## Progress Overview

| Phase | Module | Status | Agent | Date Started | Date Completed | Notes |
|-------|--------|--------|-------|-------------|----------------|-------|
| 0 | Build infrastructure | `[ ]` | — | — | — | — |
| 1 | entities.c | `[ ]` | — | — | — | — |
| 2 | tmbstr.c | `[ ]` | — | — | — | — |
| 3 | utf8.c | `[ ]` | — | — | — | — |
| 4 | attrdict.c + tags.c | `[ ]` | — | — | — | Decision needed: parser fn ptrs |
| 5 | config.c + messages | `[ ]` | — | — | — | May split into 5a/5b/5c |
| 6 | buffio/fileio/streamio | `[ ]` | — | — | — | — |
| — | **TREE WALL** | — | — | — | — | Decision needed: tree repr |
| 7 | access.c + attrs.c | `[ ]` | — | — | — | — |
| 8 | pprint.c | `[ ]` | — | — | — | — |
| 9 | clean.c | `[ ]` | — | — | — | — |
| 10 | lexer.c + parser.c | `[ ]` | — | — | — | THE BOSS FIGHT |

## Milestones

| Milestone | Phases | Status | Date |
|-----------|--------|--------|------|
| **M1: First Rust** | 0-1 | `[ ]` | — |
| **M2: Leaf modules** | 0-3 | `[ ]` | — |
| **M3: All non-tree** | 0-6 | `[ ]` | — |
| **M4: Full read path** | 0-8 | `[ ]` | — |
| **M5: Full rewrite** | 0-10 | `[ ]` | — |

---

## Phase 0: Build Infrastructure

**Status**: `[ ]` Not started
**Workflow**: [workflows/phase-0-build-infra.md](workflows/phase-0-build-infra.md)
**Depends on**: Nothing

### Checklist

- [ ] Set up `reference/` directory with tidy-html5 C snapshot
- [ ] Build reference C binary with CMake, verify it works
- [ ] Run existing 402 regression tests against reference binary
- [ ] Initialize `rust/` crate with `Cargo.toml`
- [ ] Initialize `ffi/` crate with `Cargo.toml`
- [ ] Create trivial Rust probe function, export via FFI
- [ ] Write C test program that calls the Rust function
- [ ] Verify link + run on Linux
- [ ] Document build process in README
- [ ] **Update this file and LEARNINGS.md**

---

## Phase 1: entities.c

**Status**: `[ ]` Not started
**Workflow**: [workflows/phase-1-entities.md](workflows/phase-1-entities.md)
**Depends on**: Phase 0

### Checklist

- [ ] Read C source: `reference/src/entities.c`, `reference/src/entities.h`
- [ ] Read call sites: lexer.c:1251, pprint.c:1270, pprint.c:1324
- [ ] Extract entity table (2,032 entries) for test data
- [ ] Download html5lib-tests `entities.json`
- [ ] Write Rust `entities.rs` with lookup table
- [ ] Implement `entity_info()` — named, numeric, hex entities
- [ ] Implement `entity_name()` — reverse lookup
- [ ] Write FFI wrappers (`prvTidyEntityInfo`, `prvTidyEntityName`)
- [ ] Write oracle tests (all 2,032 named + numeric sampling)
- [ ] Wire into build, replacing `entities.c`
- [ ] Run 402 regression tests — all pass
- [ ] **Update this file and LEARNINGS.md**

---

## Phase 2: tmbstr.c

**Status**: `[ ]` Not started
**Workflow**: [workflows/phase-2-tmbstr.md](workflows/phase-2-tmbstr.md)
**Depends on**: Phase 0 (Phase 1 recommended but not strictly required)

### Checklist

- [ ] Read C source: `reference/src/tmbstr.c`, `reference/src/tmbstr.h`
- [ ] Map all 19 calling files
- [ ] Define `TidyAllocator` + `TidyAllocatorVtbl` as `repr(C)` in Rust
- [ ] Reproduce `ToLower`/`ToUpper` lookup table from `lexer.h`
- [ ] Implement all 17 functions
- [ ] Fix `strrep` buffer overflow (safe for valid inputs, bounded)
- [ ] Write FFI wrappers with `prvTidy` prefix
- [ ] Oracle-test every function
- [ ] Wire into build, replacing `tmbstr.c`
- [ ] Run 402 regression tests — all pass
- [ ] **Update this file and LEARNINGS.md**

---

## Phase 3: utf8.c

**Status**: `[ ]` Not started
**Workflow**: [workflows/phase-3-utf8.md](workflows/phase-3-utf8.md)
**Depends on**: Phase 0

### Checklist

- [ ] Read C source: `reference/src/utf8.c`, `reference/src/utf8.h`
- [ ] Define `TidyInputSource`, `TidyOutputSink` as `repr(C)`
- [ ] Implement decode/encode with buffer and callback paths
- [ ] Reproduce overlong-sequence detection exactly
- [ ] Test all valid codepoints (0x0000–0x10FFFF)
- [ ] Test Markus Kuhn UTF-8 stress test inputs
- [ ] Test invalid/overlong sequences
- [ ] Wire into build, replacing `utf8.c`
- [ ] Run 402 regression tests — all pass
- [ ] **Update this file and LEARNINGS.md**

---

## Phase 4: attrdict.c + tags.c

**Status**: `[ ]` Not started
**Workflow**: [workflows/phase-4-attrdict-tags.md](workflows/phase-4-attrdict-tags.md)
**Depends on**: Phase 2 (tmbstr used in lookups)

### Decision Required Before Starting

How to handle parser function pointers in the tag table while parser
is still in C. See PLAN.md for options. Record decision in LEARNINGS.md.

### Checklist

- [ ] Decide on parser function pointer approach
- [ ] Read C source for both files
- [ ] Implement attribute dictionary in Rust
- [ ] Implement tag dictionary in Rust
- [ ] Handle parser function pointer references
- [ ] Write FFI wrappers
- [ ] Oracle test all lookups
- [ ] Wire into build, run 402 regression tests
- [ ] **Update this file and LEARNINGS.md**

---

## Phase 5: config.c + message.c + language.c

**Status**: `[ ]` Not started
**Workflow**: [workflows/phase-5-config-messages.md](workflows/phase-5-config-messages.md)
**Depends on**: Phase 2, Phase 4

Large phase (~12,000 LOC). Sub-phases:
- [ ] **5a**: language.c + language_en.h (string tables)
- [ ] **5b**: message.c + messageobj.c (formatting)
- [ ] **5c**: config.c (option parsing — high coupling to TidyDoc)

---

## Phase 6: buffio.c + fileio.c + streamio.c

**Status**: `[ ]` Not started
**Workflow**: [workflows/phase-6-io.md](workflows/phase-6-io.md)
**Depends on**: Phase 3 (utf8), Phase 2 (tmbstr)

### Checklist

- [ ] Read C source for all three files
- [ ] Implement buffer I/O
- [ ] Implement file I/O
- [ ] Implement stream I/O with encoding support
- [ ] Wire into build, run 402 regression tests
- [ ] **Update this file and LEARNINGS.md**

---

## TREE WALL — Decision Point

**Before Phase 7**: Choose Rust tree representation.

- [ ] Option A: Arena + indices (recommended)
- [ ] Option B: Rc<RefCell<Node>>
- [ ] Option C: Raw pointers (not recommended)

Record decision rationale in LEARNINGS.md.

---

## Phases 7–10 (Post-Tree-Wall)

Detailed checklists will be written when we reach this point. See
PLAN.md for current outlines.

- [ ] Phase 7: access.c + attrs.c (tree readers)
- [ ] Phase 8: pprint.c (tree reader → output)
- [ ] Phase 9: clean.c (tree mutator)
- [ ] Phase 10: lexer.c + parser.c (tree creator — the boss fight)

---

## How Agents Should Update This File

After completing any phase or significant task:

1. Change `[ ]` → `[x]` for completed items
2. Change `[ ]` → `[~]` when starting work
3. Update the **Progress Overview** table row (status, agent name, dates)
4. Update the milestone table if a milestone is reached
5. Change the **Last updated** line at the top
6. Add a corresponding entry in [LEARNINGS.md](LEARNINGS.md)
