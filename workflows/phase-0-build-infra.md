# Workflow: Phase 0 — Build Infrastructure

**Objective**: Set up the project so Rust code compiles, links into
the C build, and a trivial FFI call works end-to-end.

**Prerequisites**: None (this is the first phase).

**Estimated scope**: Build system only — no tidy logic.

---

## Before You Start

1. Read [AGENTS.md](../AGENTS.md) for project principles
2. Read [ROADMAP.md](../ROADMAP.md) and mark Phase 0 as `[~]` in progress
3. Read [LEARNINGS.md](../LEARNINGS.md) for any notes from prior agents

---

## Step 1: Set Up the Reference C Build

The tidy-html5 C source should be in `reference/`. If it's not already
there, copy it from the repository root (the current repo IS tidy-html5).

```bash
# Verify the C source is available
ls reference/src/  # Should see *.c and *.h files
# OR if reference/ doesn't exist yet, create it from current source
# mkdir -p reference && cp -r src include CMakeLists.txt version.txt README reference/
```

Build the C reference binary:

```bash
cd reference
mkdir -p build && cd build
cmake ..
make -j$(nproc)
# Verify: ./tidy --version should output something
./tidy --version
```

### Validation

- [ ] `reference/build/tidy` exists and runs
- [ ] `reference/build/tidy --version` prints version info

---

## Step 2: Verify Existing Regression Tests

```bash
cd reference/regression_testing
# The test runner is Ruby-based
ruby test.rb
```

If Ruby isn't available or tests don't run, document this in LEARNINGS.md
and move on — the tests can be adapted later.

### Validation

- [ ] Test runner executes (or documented why it can't)
- [ ] Note the pass/fail count in LEARNINGS.md

---

## Step 3: Initialize the Rust Crate

Create `rust/Cargo.toml`:

```toml
[package]
name = "tidy-oxide"
version = "0.1.0"
edition = "2021"

[lib]
name = "tidy_oxide"
crate-type = ["lib", "staticlib"]

[dependencies]
libc = "0.2"
```

Create `rust/src/lib.rs`:

```rust
//! tidy-oxide: Incremental Rust rewrite of HTML Tidy.

/// Probe function to verify Rust-C linking works.
pub fn probe() -> i32 {
    42
}
```

### Validation

- [ ] `cd rust && cargo build` succeeds
- [ ] `cd rust && cargo test` succeeds

---

## Step 4: Initialize the FFI Crate

Create `ffi/Cargo.toml`:

```toml
[package]
name = "tidy-oxide-ffi"
version = "0.1.0"
edition = "2021"

[lib]
name = "tidy_oxide_ffi"
crate-type = ["staticlib"]

[dependencies]
tidy-oxide = { path = "../rust" }
libc = "0.2"
```

Create `ffi/src/lib.rs`:

```rust
use std::os::raw::c_int;

/// Probe function callable from C.
/// Returns 42 to verify Rust-C linking works.
#[no_mangle]
pub extern "C" fn rust_tidy_probe() -> c_int {
    tidy_oxide::probe() as c_int
}
```

### Validation

- [ ] `cd ffi && cargo build` succeeds
- [ ] `ffi/target/debug/libtidy_oxide_ffi.a` exists

---

## Step 5: Write a C Test That Calls Rust

Create `tests/ffi_probe_test.c`:

```c
#include <stdio.h>

/* Rust FFI function declaration */
extern int rust_tidy_probe(void);

int main(void) {
    int result = rust_tidy_probe();
    if (result == 42) {
        printf("PASS: rust_tidy_probe() returned %d\n", result);
        return 0;
    } else {
        printf("FAIL: rust_tidy_probe() returned %d, expected 42\n", result);
        return 1;
    }
}
```

Compile and link:

```bash
# Build the Rust static library first
cd ffi && cargo build --release && cd ..

# Compile the C test and link against Rust
gcc tests/ffi_probe_test.c \
    -L ffi/target/release \
    -l tidy_oxide_ffi \
    -lpthread -ldl -lm \
    -o tests/ffi_probe_test

# Run it
./tests/ffi_probe_test
```

**Note**: The exact linker flags may vary. Common issues:
- Missing `-lpthread -ldl -lm` (Rust runtime dependencies on Linux)
- Static library name might be `libtidy_oxide_ffi.a`
- On macOS, replace `-ldl` with `-framework Security`

Record any linker flag discoveries in LEARNINGS.md — future phases
will need the same flags.

### Validation

- [ ] `tests/ffi_probe_test` compiles without errors
- [ ] Running it prints `PASS: rust_tidy_probe() returned 42`
- [ ] Exit code is 0

---

## Step 6: Set Up Oracle Test Infrastructure

Create `tests/oracle/run_oracle.sh` — a skeleton script that will be
extended in later phases:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PROJECT_ROOT="$(cd "$SCRIPT_DIR/../.." && pwd)"
C_TIDY="$PROJECT_ROOT/reference/build/tidy"

if [ ! -x "$C_TIDY" ]; then
    echo "ERROR: Reference C binary not found at $C_TIDY"
    echo "Build it first: cd reference && mkdir -p build && cd build && cmake .. && make"
    exit 1
fi

echo "Oracle test infrastructure ready."
echo "C tidy binary: $C_TIDY"
echo "Version: $($C_TIDY --version 2>&1 | head -1)"
echo ""
echo "No module-specific oracle tests yet. Run phase workflows to add them."
```

```bash
chmod +x tests/oracle/run_oracle.sh
```

### Validation

- [ ] `tests/oracle/run_oracle.sh` runs and prints the C tidy version

---

## Step 7: Document and Finish

1. Update [ROADMAP.md](../ROADMAP.md):
   - Mark Phase 0 checklist items as `[x]`
   - Update the Progress Overview table
   - If Milestone M1 criteria aren't fully met yet, leave it `[ ]`
2. Add an entry to [LEARNINGS.md](../LEARNINGS.md) documenting:
   - What linker flags were needed
   - Whether the regression test runner worked
   - Any build system surprises
   - Time/effort this phase took
3. Commit with a descriptive message

---

## Definition of Done

All of these must be true:

- [ ] C reference binary builds and runs
- [ ] `cargo build` works for both `rust/` and `ffi/` crates
- [ ] A C program calls a Rust function through FFI and gets the right result
- [ ] Oracle test script skeleton runs
- [ ] ROADMAP.md updated
- [ ] LEARNINGS.md entry added
- [ ] All changes committed
