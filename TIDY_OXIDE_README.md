# tidy-oxide

An experimental, AI-assisted incremental rewrite of [HTML Tidy](https://github.com/htacg/tidy-html5) in Rust.

## What This Is

HTML Tidy is a 25-year-old C library (~58,000 LOC) that parses, validates, cleans, and pretty-prints HTML. It's widely used but has known memory safety issues and is difficult to attract new contributors to.

This project is an experiment to see how far AI-driven, oracle-validated C-to-Rust translation can go. The existing C binary serves as the ground truth: for each module replaced, the Rust version must produce byte-identical output on all inputs.

**This is NOT a fork of tidy-html5.** It's a new project that:
- Contains a read-only snapshot of the tidy-html5 C source for reference and oracle testing
- Incrementally reimplements tidy's functionality in Rust
- Maintains C API compatibility via FFI shims
- Aims for behavioral compatibility validated by the original's 402 regression tests plus thousands of additional generated tests

## Status

See [PLAN.md](PLAN.md) for current progress.

| Phase | Module | LOC | Status |
|-------|--------|-----|--------|
| 0 | Build infrastructure | — | Not started |
| 1 | entities.c | 2,195 | Not started |
| 2 | tmbstr.c | 277 | Not started |
| 3 | utf8.c | 523 | Not started |
| 4 | attrdict.c + tags.c | 5,749 | Not started |
| 5 | config.c + messages | 12,856 | Not started |
| 6 | buffio/fileio/streamio | 2,323 | Not started |
| 7 | access.c + attrs.c | 6,317 | Not started |
| 8 | pprint.c | 2,714 | Not started |
| 9 | clean.c | 2,862 | Not started |
| 10 | lexer.c + parser.c | 10,998 | Not started |

## Motivation

1. **Exploring AI-assisted code translation**: Can AI agents (Claude, Gemini) rewrite a real C codebase piece-by-piece, using the original as an oracle?
2. **Memory safety**: Tidy has real bugs (buffer overflows in string handling, unbounded allocations, global mutable state). Rust prevents these categories.
3. **Ecosystem gap**: No pure-Rust HTML tidy/cleanup tool exists. Parsers (html5ever), sanitizers (Ammonia), and scrapers exist, but nothing does repair + validate + pretty-print.

## Approach

For each module:
1. AI reads the C source and generates exhaustive tests by running the C binary as an oracle
2. AI writes a Rust implementation
3. Tests validate the Rust code matches C behavior exactly
4. The Rust module is wired in via FFI, replacing the C module
5. All 402 regression tests pass on the hybrid binary

See [AGENTS.md](AGENTS.md) for detailed instructions for AI agents working on this project.

## Building

### Prerequisites

- Rust toolchain (stable)
- C compiler (gcc or clang) for building the reference binary
- CMake 2.8.12+ for the reference C build

### Build the reference C binary (one-time)

```bash
cd reference
mkdir -p build && cd build
cmake ..
make -j$(nproc)
# Binary at reference/build/tidy
```

### Build the Rust crate

```bash
cd rust
cargo build
cargo test
```

### Run oracle tests

```bash
./tests/oracle/run_oracle.sh
```

## Relationship to tidy-html5

This project includes a snapshot of tidy-html5 version 5.9.20 (2022-01-25) in the `reference/` directory. That code is copyright (c) 1998-2016 W3C (MIT, ERCIM, Keio) and HTACG, used under the terms of the W3C license (see `reference/README/LICENSE.txt`).

The Rust code in `rust/` and `ffi/` is new work.

We are grateful to the HTACG maintainers and all contributors to HTML Tidy for their decades of work on this essential tool.

## License

The reference C code retains its original W3C license. New Rust code is dual-licensed MIT/Apache-2.0 (standard Rust ecosystem licensing).
