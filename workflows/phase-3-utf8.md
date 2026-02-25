# Workflow: Phase 3 — utf8.c

**Objective**: Replace UTF-8/UTF-16 encode/decode functions with Rust.

**Prerequisites**: Phase 0 complete.

**Key files**:
- `reference/src/utf8.c` — encode/decode functions (~523 LOC)
- `reference/src/utf8.h` — declarations
- `reference/include/tidy.h` — TidyInputSource, TidyOutputSink definitions

**Why this is interesting**: The encode/decode functions accept either
raw byte buffers OR I/O callback function pointers. The Rust code must
handle both paths, including calling through C function pointers.

---

## Before You Start

1. Read [AGENTS.md](../AGENTS.md)
2. Read [ROADMAP.md](../ROADMAP.md) — Phase 0 must be `[x]`
3. Read [LEARNINGS.md](../LEARNINGS.md) — build notes, allocator lessons
4. Mark Phase 3 as `[~]` in ROADMAP.md

---

## Step 1: Understand the C Code

Read:

```
reference/src/utf8.c       # All encode/decode implementations
reference/src/utf8.h       # Internal declarations
reference/include/tidy.h   # TidyInputSource, TidyOutputSink (public API types)
```

### Functions to understand

| Function | Callback? | Notes |
|----------|----------|-------|
| `DecodeUTF8BytesToChar` | YES | Reads from buffer or TidyInputSource |
| `EncodeCharToUTF8Bytes` | YES | Writes to buffer or TidyOutputSink |
| `GetUTF8` | no | Convenience: decode from string |
| `PutUTF8` | no | Convenience: encode to buffer |
| `IsValidUTF16FromUCS4` | no | Pure arithmetic |
| `IsHighSurrogate` | no | Range check |
| `IsLowSurrogate` | no | Range check |
| `CombineSurrogatePair` | no | Arithmetic |
| `SplitSurrogatePair` | no | Arithmetic |
| `IsValidCombinedChar` | no | Range check |
| `IsCombinedChar` | no | Range check |

### The callback complexity

```c
struct _TidyInputSource {
    void*       sourceData;
    TidyGetByteFunc getByte;    // int (*)(void*)
    TidyUngetByteFunc ungetByte; // void (*)(void*, byte)
    TidyEOFFunc eof;            // Bool (*)(void*)
};
```

The Rust code will need `unsafe extern "C"` function pointer calls
to read bytes from a TidyInputSource.

### The overlong detection

The C code explicitly detects overlong UTF-8 sequences and rejects
them. Understand the exact error codes and behavior for:
- Overlong 2-byte sequences (codepoints < 0x80)
- Overlong 3-byte sequences (codepoints < 0x800)
- Overlong 4-byte sequences (codepoints < 0x10000)
- Invalid continuation bytes
- Truncated sequences
- Surrogate halves (U+D800-U+DFFF)

Record these in LEARNINGS.md — the exact behavior matters.

---

## Step 2: Generate Tests

### 2a: Pure arithmetic functions

These are trivial — test exhaustively:
- `IsHighSurrogate`: test all values 0x0000–0xFFFF
- `IsLowSurrogate`: same
- `CombineSurrogatePair`: all valid pairs
- `SplitSurrogatePair`: all valid codepoints > 0xFFFF
- `IsValidUTF16FromUCS4`: full range

### 2b: Encode/decode

Test categories:
- All valid codepoints (0x0000–0x10FFFF, excluding surrogates)
- Overlong encodings for each length (2, 3, 4 byte)
- Invalid continuation bytes (0x80–0xBF in wrong positions)
- Truncated sequences (missing continuation bytes)
- Out-of-range (> 0x10FFFF)
- Surrogate codepoints (U+D800–U+DFFF)
- The Markus Kuhn UTF-8 stress test file

### 2c: Callback path

Generate tests that use TidyInputSource callbacks to feed bytes one
at a time. Compare with the buffer path — results must be identical.

### Validation

- [ ] Oracle tests for all pure functions
- [ ] Oracle tests for encode/decode (buffer path)
- [ ] Oracle tests for encode/decode (callback path)
- [ ] Markus Kuhn test file processed

---

## Step 3: Implement the Rust Module

Create `rust/src/utf8.rs`.

### 3a: Pure functions first

Start with the simple range checks and arithmetic. These are safe Rust:

```rust
pub fn is_high_surrogate(n: u32) -> bool {
    (0xD800..=0xDBFF).contains(&n)
}
```

### 3b: Buffer-path encode/decode

Implement decode/encode using Rust slices for the buffer path.
This can be mostly safe Rust.

### 3c: Callback-path encode/decode

This requires `unsafe` to call through C function pointers:

```rust
pub unsafe fn decode_utf8_from_source(
    source: *mut TidyInputSource,
) -> Result<u32, Utf8Error> {
    let get_byte = (*source).getByte;
    let byte = get_byte((*source).sourceData);
    // ... decode using byte-by-byte reads ...
}
```

### Important: match C error behavior exactly

The C code returns specific error codes for different failure modes.
The Rust code must return the same codes, not Rust-idiomatic errors.

### Validation

- [ ] All pure functions match C oracle
- [ ] Buffer encode/decode matches C for all test vectors
- [ ] Callback encode/decode matches C for all test vectors

---

## Step 4: FFI Wrappers

Export with `prvTidy` prefix. All callback-accepting functions will
have `unsafe` in their signatures.

Verify symbol names:
```bash
nm reference/build/CMakeFiles/tidy-obj.dir/src/utf8.c.o | grep prvTidy
```

---

## Step 5: Wire Into Build and Test

Replace `utf8.c` in the build. Run all 402 regression tests.

UTF-8 is used throughout the parsing pipeline, so failures here will
show up as garbled output or crashes in many tests.

---

## Step 6: Document and Finish

Update ROADMAP.md and LEARNINGS.md. Key things to record:
- How the callback function pointer calling worked out
- Any overlong detection differences found
- Whether Rust's standard UTF-8 handling helped or conflicted

---

## Definition of Done

- [ ] All 11 functions implemented in Rust
- [ ] Buffer and callback paths both work
- [ ] Overlong detection matches C exactly
- [ ] All oracle tests pass
- [ ] All 402 regression tests pass
- [ ] ROADMAP.md and LEARNINGS.md updated
- [ ] All changes committed
