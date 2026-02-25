# Workflow: Phase 2 — tmbstr.c

**Objective**: Replace 17 string utility functions with Rust. This is
the first module that involves the TidyAllocator — a major complexity jump.

**Prerequisites**: Phase 0 complete. Phase 1 recommended (validates the
workflow) but not strictly required.

**Key files**:
- `reference/src/tmbstr.c` — 17 string functions (~277 LOC)
- `reference/src/tmbstr.h` — declarations
- `reference/src/forward.h` — TidyAllocator definitions
- `reference/src/lexer.h` — ToLower/ToUpper lookup table

**Why this is harder than entities.c**: Two functions (`tmbstrdup`,
`tmbstrndup`) allocate memory through the C `TidyAllocator` vtable.
The Rust code must call through the same allocator. Also, tmbstr
functions are called from **19 of 26 source files** — high blast radius.

---

## Before You Start

1. Read [AGENTS.md](../AGENTS.md) — especially the Allocator section
2. Read [ROADMAP.md](../ROADMAP.md) — Phase 0 must be `[x]`
3. Read [LEARNINGS.md](../LEARNINGS.md) — especially Phase 0 and Phase 1 notes
4. Mark Phase 2 as `[~]` in ROADMAP.md

---

## Step 1: Understand the C Code

Read carefully:

```
reference/src/tmbstr.c     # All 17 function implementations
reference/src/tmbstr.h     # Declarations
reference/src/forward.h    # TidyAllocator, TidyAllocatorVtbl, TidyAlloc macro
reference/src/lexer.h      # TY_(IsDigit), TY_(IsLetter), TY_(ToLower) — byte lookup tables
```

### Key things to understand

**The allocator vtable**:
```c
struct _TidyAllocatorVtbl {
    void* (*alloc)(TidyAllocator *self, size_t nBytes);
    void* (*realloc)(TidyAllocator *self, void *block, size_t nBytes);
    void  (*free)(TidyAllocator *self, void *block);
    void  (*panic)(TidyAllocator *self, ctmbstr msg);
};
```

`tmbstrdup` calls `TidyAlloc(allocator, size)` which expands to
`(*allocator->vtbl->alloc)(allocator, size)`.

**The ToLower table**: A 256-byte lookup array. `ToLower(c)` is NOT
`tolower()` from libc — it's ASCII-only and uses tidy's own table.
The Rust code must reproduce this exact table, not use Rust's
`.to_ascii_lowercase()`.

**The strrep buffer overflow**: `strrep()` replaces a substring in a
fixed-size buffer without checking if the replacement makes the string
longer than the buffer. The Rust version should bounds-check but
produce identical output for inputs that fit.

Record observations in LEARNINGS.md.

---

## Step 2: Generate Tests

### 2a: Catalog all 17 functions and their signatures

| # | Function | Allocates? | Special |
|---|----------|-----------|---------|
| 1 | `tmbstrdup` | YES | Allocator vtable |
| 2 | `tmbstrndup` | YES | Allocator vtable |
| 3 | `tmbstrncpy` | no | |
| 4 | `tmbstrcpy` | no | Returns chars copied |
| 5 | `tmbstrcat` | no | Returns chars appended |
| 6 | `tmbstrcmp` | no | |
| 7 | `tmbstrlen` | no | |
| 8 | `tmbstrcasecmp` | no | Uses ToLower table |
| 9 | `tmbstrncmp` | no | |
| 10 | `tmbstrncasecmp` | no | Uses ToLower table |
| 11 | `tmbsubstrn` | no | Exact substring search |
| 12 | `tmbsubstr` | no | Case-insensitive substr |
| 13 | `tmbstrtolower` | no | In-place, ASCII-only |
| 14 | `tmbstrtoupper` | no | In-place, ASCII-only |
| 15 | `tmbvsnprintf` | no | Wraps vsnprintf |
| 16 | `tmbsnprintf` | no | Wraps snprintf |
| 17 | `strrep` | no | HAS BUFFER OVERFLOW BUG |

### 2b: Generate oracle test data

For each non-allocating function, write a C test harness that calls
the function with diverse inputs and prints results. Focus on:

- Empty strings, NULL pointers (check if C crashes or handles them)
- ASCII, Latin-1, high-byte characters (for case functions)
- Boundary conditions (n=0 for strncmp, etc.)
- The ToLower table mapping for all 256 byte values

For allocating functions (`tmbstrdup`, `tmbstrndup`), test with
the default allocator (the C code provides one).

For `strrep`: test with replacements that both fit and overflow the
buffer. Document the overflow behavior.

### Validation

- [ ] Oracle test data generated for all 17 functions
- [ ] ToLower table extracted (all 256 entries)
- [ ] strrep overflow behavior documented

---

## Step 3: Implement the Rust Module

Create `rust/src/tmbstr.rs`.

### 3a: Define allocator types

```rust
use std::os::raw::{c_char, c_int, c_uint, c_void};

#[repr(C)]
pub struct TidyAllocatorVtbl {
    pub alloc: unsafe extern "C" fn(*mut TidyAllocator, usize) -> *mut c_void,
    pub realloc: unsafe extern "C" fn(*mut TidyAllocator, *mut c_void, usize) -> *mut c_void,
    pub free: unsafe extern "C" fn(*mut TidyAllocator, *mut c_void),
    pub panic: unsafe extern "C" fn(*mut TidyAllocator, *const c_char),
}

#[repr(C)]
pub struct TidyAllocator {
    pub vtbl: *const TidyAllocatorVtbl,
}
```

Put these in a shared module (`rust/src/alloc.rs` or similar) since
later phases will need them too.

### 3b: Reproduce the ToLower table

Extract the exact 256-byte table from `lexer.h` and embed it:

```rust
static TO_LOWER: [u8; 256] = [
    // Copy exact values from lexer.h
];

pub fn to_lower(c: u8) -> u8 {
    TO_LOWER[c as usize]
}
```

### 3c: Implement all 17 functions

Work function by function. For each one:
1. Read the C source
2. Write the Rust equivalent
3. Run the oracle tests
4. Move to the next

For `tmbstrdup` / `tmbstrndup`:
```rust
pub unsafe fn tmbstrdup(alloc: *mut TidyAllocator, s: *const c_char) -> *mut c_char {
    if s.is_null() {
        return std::ptr::null_mut();
    }
    let len = libc::strlen(s);
    let ptr = ((*(*alloc).vtbl).alloc)(alloc, len + 1) as *mut c_char;
    if !ptr.is_null() {
        std::ptr::copy_nonoverlapping(s, ptr, len + 1);
    }
    ptr
}
```

For `tmbvsnprintf` / `tmbsnprintf`: just delegate to libc. Don't
reimplement printf in Rust.

### Validation

- [ ] All 17 functions implemented
- [ ] `cargo test` passes for all oracle test cases
- [ ] ToLower table matches C exactly (test all 256 values)

---

## Step 4: Write FFI Wrappers

Export all 17 functions with `prvTidy` prefix. Example:

```rust
#[no_mangle]
pub unsafe extern "C" fn prvTidytmbstrdup(
    alloc: *mut TidyAllocator,
    str: *const c_char,
) -> *mut c_char {
    tidy_oxide::tmbstr::tmbstrdup(alloc, str)
}
```

Check the exact mangled names by looking at the C object file:
```bash
nm reference/build/CMakeFiles/tidy-obj.dir/src/tmbstr.c.o | grep prvTidy
```

The Rust symbols must match exactly.

### Validation

- [ ] All 17 FFI wrappers exported
- [ ] Symbol names verified against C object file

---

## Step 5: Wire Into Build and Test

1. Remove `tmbstr.c` from the C build
2. Link the Rust FFI library
3. Build the hybrid binary
4. Run all 402 regression tests

**This is the highest-risk step in this phase** because tmbstr is
called from 19 files. Any symbol mismatch, calling convention error,
or behavioral difference will likely cause crashes or test failures.

Debug strategy if tests fail:
- Start with a single regression test case
- Run under valgrind to catch memory issues
- Check that allocator calls go through the vtable correctly
- Verify ToLower table matches for the specific characters in the
  failing test case

### Validation

- [ ] Hybrid binary builds without linker errors
- [ ] All 402 regression tests pass
- [ ] No memory errors (valgrind clean or equivalent)

---

## Step 6: Document and Finish

1. Update [ROADMAP.md](../ROADMAP.md)
2. Add entry to [LEARNINGS.md](../LEARNINGS.md):
   - Allocator vtable experience — what worked, what was tricky
   - ToLower table nuances
   - strrep overflow handling decision
   - Linker or build system changes needed
   - Tips for the Phase 3 agent
3. Commit everything

---

## Definition of Done

- [ ] All 17 string functions implemented in Rust
- [ ] TidyAllocator vtable integration works for allocating functions
- [ ] ToLower table matches C exactly
- [ ] strrep buffer overflow is bounded in Rust
- [ ] All oracle tests pass
- [ ] All 402 regression tests pass with Rust tmbstr linked in
- [ ] ROADMAP.md and LEARNINGS.md updated
- [ ] All changes committed
