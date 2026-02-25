# Workflow: Phase 6 — buffio.c + fileio.c + streamio.c

**Objective**: Replace I/O stream abstractions with Rust.

**Prerequisites**: Phase 2 (tmbstr) and Phase 3 (utf8) complete.

**Key files**:
- `reference/src/buffio.c` — buffer-based I/O
- `reference/src/fileio.c` — file-based I/O
- `reference/src/streamio.c` — stream abstraction with encoding

**This is the last phase before the Tree Wall.** After this, all
non-tree-dependent modules are in Rust.

---

## Before You Start

1. Read [AGENTS.md](../AGENTS.md)
2. Read [ROADMAP.md](../ROADMAP.md) — Phases 0, 2, 3 must be `[x]`
3. Read [LEARNINGS.md](../LEARNINGS.md)
4. Mark Phase 6 as `[~]` in ROADMAP.md

---

## Step 1: Understand the C Code

These three files implement the `TidyInputSource` and `TidyOutputSink`
interfaces that the lexer and pretty printer use.

Read:
```
reference/src/buffio.c     # TidyBuffer-based I/O
reference/src/fileio.c     # FILE*-based I/O
reference/src/streamio.c   # Stream abstraction, encoding conversion
reference/include/tidybuffio.h  # TidyBuffer type
```

Key questions:
- What encoding conversions does streamio.c handle?
- How does it interact with the utf8 functions from Phase 3?
- What memory management does buffio use?

---

## Steps 2–6: Standard Phase Pattern

Follow the same approach as prior phases. Oracle-test all I/O
operations.

### Extra concern

streamio.c handles character encoding (UTF-8, UTF-16, Latin-1, etc.).
Encoding detection and conversion must match C behavior exactly, or
every subsequent module will produce wrong output.

---

## After This Phase

If Phase 6 passes, you've hit **Milestone M3** — all non-tree modules
are in Rust. Update ROADMAP.md to reflect this.

The next phases require a **tree representation decision**. Do NOT
start Phase 7 without making this decision and recording it in
LEARNINGS.md.

---

## Definition of Done

- [ ] buffio.c replaced with Rust
- [ ] fileio.c replaced with Rust
- [ ] streamio.c replaced with Rust (including encoding conversion)
- [ ] All 402 regression tests pass
- [ ] Milestone M3 assessed
- [ ] ROADMAP.md and LEARNINGS.md updated
