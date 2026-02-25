# Workflow: Phase 1 — entities.c

**Objective**: Replace the C entity lookup module with Rust. This is
the first real module swap — proof that the Ship of Theseus approach works.

**Prerequisites**: Phase 0 complete (build infrastructure works).

**Key files**:
- `reference/src/entities.c` — 2,032-entry lookup table + 2 functions
- `reference/src/entities.h` — function declarations
- Call sites: `lexer.c:1251`, `pprint.c:1270`, `pprint.c:1324`

---

## Before You Start

1. Read [AGENTS.md](../AGENTS.md) for project principles
2. Read [ROADMAP.md](../ROADMAP.md) — Phase 0 must be `[x]`
3. Read [LEARNINGS.md](../LEARNINGS.md) for build system notes from Phase 0
4. Mark Phase 1 as `[~]` in ROADMAP.md

---

## Step 1: Understand the C Code

Read these files carefully:

```
reference/src/entities.c    # The implementation
reference/src/entities.h    # The header (2 function declarations)
reference/src/forward.h     # TY_() macro definition, type definitions
reference/src/lexer.c       # Line ~1251: call to TY_(EntityInfo)
reference/src/pprint.c      # Lines ~1270, ~1324: calls to TY_(EntityName)
```

Key observations to verify:
- The entity table is a sorted static array of `{ name, versions, code }`
- `EntityInfo` uses binary search by first character, then linear scan
- `EntityInfo` handles `&#decimal;` and `&#xhex;` numeric entities
- `EntityName` does linear scan for reverse lookup (code → name)
- No allocator interaction — all data is `static const`
- The only external dependency is `tmbstrcmp` from tmbstr.c, but since
  we're replacing the whole module, we can use Rust string comparison

Record what you learn in LEARNINGS.md.

---

## Step 2: Extract Test Data

### 2a: Extract the entity table from C

Write a script or manually extract all 2,032 entity entries from
`reference/src/entities.c`. Each entry has:
- `name` (string)
- `versions` (bitmask — which HTML versions support this entity)
- `code` (Unicode codepoint)

Save as a machine-readable format (JSON, CSV, or Rust literal).

### 2b: Download html5lib entity data

```bash
mkdir -p tests/html5lib
# Download the official entity test data
curl -L -o tests/html5lib/entities.json \
    https://raw.githubusercontent.com/nicknisi/html5lib-tests/refs/heads/master/tokenizer/entities.json
```

### 2c: Generate oracle test fixtures

Write a small C program (or extend the probe test) that calls
`TY_(EntityInfo)` and `TY_(EntityName)` with diverse inputs and
prints the results. Run it against the C binary to capture expected
behavior.

Inputs to test:
- Every one of the 2,032 named entities
- Numeric entities: `&#0;` through `&#1114111;` (sample key values)
- Hex entities: `&#x0;` through `&#x10FFFF;` (sample key values)
- Edge cases: empty string, unknown entity, case variations
- XML mode vs non-XML mode differences

Save oracle output as test fixtures.

### Validation

- [ ] Entity table extracted (all 2,032 entries)
- [ ] html5lib entities.json downloaded
- [ ] Oracle test fixtures generated

---

## Step 3: Implement the Rust Module

Create `rust/src/entities.rs`:

### Entity table

Consider two approaches:
1. **Sorted array + binary search** (matches C behavior closely)
2. **`phf` crate** (compile-time perfect hash — faster lookups)

Recommendation: Start with sorted array to match C behavior. Switch to
phf later if performance matters.

```rust
pub struct EntityInfo {
    pub name: &'static str,
    pub versions: u32,
    pub code: u32,
}

static ENTITIES: &[EntityInfo] = &[
    // ... all 2,032 entries ...
];
```

### Functions to implement

```rust
/// Look up entity by name. Returns (codepoint, versions) if found.
/// Handles named entities, &#decimal; and &#xhex; forms.
pub fn entity_info(name: &str, is_xml: bool) -> Option<(u32, u32)>

/// Reverse lookup: given codepoint and version mask, return entity name.
pub fn entity_name(code: u32, versions: u32) -> Option<&'static str>
```

### Important behavioral details to match

- Named entity lookup is case-sensitive
- Numeric `&#0;` through `&#31;` (control chars) — check what C returns
- Invalid numeric entities — check C behavior
- The `versions` bitmask filtering — XML entities are a subset
- Binary search boundary conditions

### Validation

- [ ] `cargo test` passes for all entity test cases
- [ ] Oracle comparison matches on all 2,032 named entities
- [ ] Numeric/hex entity edge cases match C behavior

---

## Step 4: Write FFI Wrappers

In `ffi/src/lib.rs`, add:

```rust
use std::os::raw::{c_char, c_int, c_uint};

type Bool = c_int;

#[no_mangle]
pub unsafe extern "C" fn prvTidyEntityInfo(
    name: *const c_char,
    is_xml: Bool,
    code: *mut c_uint,
    versions: *mut c_uint,
) -> Bool {
    // Convert C string to Rust &str
    // Call tidy_oxide::entities::entity_info()
    // Write results through output pointers
    // Return yes (1) or no (0)
    todo!()
}

#[no_mangle]
pub unsafe extern "C" fn prvTidyEntityName(
    code: c_uint,
    versions: c_uint,
) -> *const c_char {
    // Call tidy_oxide::entities::entity_name()
    // Return pointer to static string (from the entity table)
    // Return null if not found
    todo!()
}
```

**Critical**: The return type of `prvTidyEntityName` must be a pointer
to a `static` string, not heap-allocated memory. Since entity names are
compile-time constants, this is straightforward.

### Validation

- [ ] FFI crate builds
- [ ] Symbol names match: `nm ffi/target/release/libtidy_oxide_ffi.a | grep prvTidy`

---

## Step 5: Wire Into the Build

This is the tricky part — replacing `entities.o` with the Rust library
in the C build.

### Approach

1. Build the Rust FFI static library: `cd ffi && cargo build --release`
2. Modify the C build to:
   - Remove `entities.c` from the source list
   - Link against `libtidy_oxide_ffi.a`
3. Build the hybrid binary
4. Test it

The exact build integration depends on what Phase 0 established.
Reference the build notes in LEARNINGS.md.

### Validation

- [ ] Hybrid binary builds (C code + Rust entities)
- [ ] `hybrid-tidy --version` works

---

## Step 6: Run Regression Tests

Run all 402 existing regression tests against the hybrid binary:

```bash
# However the test runner was set up in Phase 0
# Point it at the hybrid binary instead of reference binary
```

### Validation

- [ ] All 402 regression tests pass
- [ ] No output differences from reference C binary

---

## Step 7: Document and Finish

1. Update [ROADMAP.md](../ROADMAP.md):
   - Mark all Phase 1 checklist items `[x]`
   - Update Progress Overview table
   - Check if M1 milestone is met
2. Add entry to [LEARNINGS.md](../LEARNINGS.md):
   - How long the phase took
   - Surprises or difficulties encountered
   - Entity lookup implementation details worth knowing
   - Any behavioral divergences found (and how they were resolved)
   - Tips for the next agent doing Phase 2
3. Commit everything with a descriptive message

---

## Definition of Done

- [ ] `rust/src/entities.rs` implements both lookup functions
- [ ] All 2,032 named entities oracle-tested
- [ ] Numeric/hex entities oracle-tested
- [ ] FFI wrappers export correct symbol names
- [ ] Hybrid binary builds
- [ ] All 402 regression tests pass with Rust entities linked in
- [ ] ROADMAP.md and LEARNINGS.md updated
- [ ] All changes committed
