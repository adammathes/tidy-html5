# Workflow: Phase 5 — config.c + message.c + language.c

**Objective**: Replace configuration parsing, error/warning messages,
and localization strings with Rust.

**Prerequisites**: Phase 2 (tmbstr) and Phase 4 (tags) complete.

**Key files**:
- `reference/src/language.c` + `reference/src/language_en.h`
- `reference/src/message.c` + `reference/src/messageobj.c`
- `reference/src/config.c`

**This is the largest phase** (~12,000 LOC). Split into sub-phases.

---

## Before You Start

1. Read [AGENTS.md](../AGENTS.md)
2. Read [ROADMAP.md](../ROADMAP.md) — Phases 0, 2, 4 must be `[x]`
3. Read [LEARNINGS.md](../LEARNINGS.md) — all prior notes
4. Mark Phase 5 as `[~]` in ROADMAP.md

---

## Sub-phase 5a: language.c + language_en.h

**Scope**: Static string tables for English messages. Low coupling.

1. Read `language.c` and `language_en.h`
2. Extract all message strings and their IDs
3. Implement Rust equivalent (could be a simple match or HashMap)
4. Oracle-test all string lookups
5. Wire up FFI, run regression tests

---

## Sub-phase 5b: message.c + messageobj.c

**Scope**: Message formatting, report generation. Depends on 5a.

1. Read `message.c` and `messageobj.c`
2. Understand printf-style formatting used in messages
3. Implement message formatting in Rust
4. Oracle-test message output
5. Wire up FFI, run regression tests

---

## Sub-phase 5c: config.c

**Scope**: Option parsing. High coupling to TidyDoc struct.

This is the hardest sub-phase. config.c reads/writes fields of the
main TidyDoc struct. The Rust code must access these fields through
the C struct layout.

1. Map all TidyDoc fields that config.c touches
2. Define `repr(C)` struct layout for the relevant parts
3. Implement option parsing
4. Oracle-test all configuration options
5. Wire up FFI, run regression tests

**WARNING**: If TidyDoc struct layout is too complex to reproduce in
Rust, consider keeping config.c in C and moving on. Record decision
in LEARNINGS.md.

---

## Definition of Done

- [ ] 5a: Language strings in Rust, oracle-tested
- [ ] 5b: Message formatting in Rust, oracle-tested
- [ ] 5c: Config parsing in Rust, oracle-tested (or documented skip)
- [ ] All 402 regression tests pass
- [ ] ROADMAP.md and LEARNINGS.md updated
