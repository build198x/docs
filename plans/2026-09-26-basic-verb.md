# `build198x basic` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn Spectrum and C64 BASIC listings into loadable `.tap`/`.prg` files, linted to be exactly what the machine's LIST shows, so every BASIC lesson can run its program in the browser.

**Architecture:** Emu198x's two tokenisers graduate unchanged to Format198x. There each gains three things: a positioned token stream (`lex_line`), a lister (`list`) that follows the machine's LIST rules, and (Spectrum only) a fix so it stops storing the space before a keyword. Build198x adds a `basic` verb (build) and `basic lint` (six rules, `--fix`) on those crates. Code-samples then rewrites its listings to the listed form and gains a Makefile per BASIC lesson, which the website already stages for its run strips.

**Tech Stack:** Rust 2024 (latest stable; `env -u RUSTUP_TOOLCHAIN cargo …`), release-plz, cargo-dist, bash Makefiles, Node 24 for the one-off ROM capture.

**Spec:** `Build198x/docs/specs/2026-09-26-basic-verb.md` (amended 2026-09-26). Binding content rule: Code198x `docs/specifications/unit.md`, the BASIC spacing paragraph.

## Global Constraints

- Crate names: `format198x-sinclair-zx-spectrum-bas`, `format198x-commodore-c64-bas` (umbrella `decisions/crate-naming.md`). They depend on nothing, the licence is `GPL-2.0-or-later`, and edition and lints come from the workspace.
- Machine ids: `sinclair-zx-spectrum`, `commodore-c64`, the ids `mediaspec198x` already uses.
- Output extensions: `.tap` (Spectrum) and `.prg` (C64). A mismatch is an error.
- Errors name the source line (`in.bas:12: …`), exit non-zero, and write nothing.
- The listed form: each source line equals its machine listing exactly (trailing whitespace ignored); screen wrapping is never modelled.
- Spectrum line numbers are 1–9999, right-aligned to four columns in the listed form. C64 line numbers are 1–63999, written as the plain number followed by one space.
- Lint output format: `path:line:column: rule-id: message`, with a 1-based line and column.
- Commits are conventional (`feat:`, `fix:`, `chore:`, `docs:`, `test:`). Build198x and Format198x are PR-only: work on a branch and open a PR.
- Commit in the repo that owns the file, using explicit pathspecs; never `git add -A`.
- Never bulk-delete. The 99 committed Spectrum tapes under `verification/` folders are evidence and are not touched.
- No `.unwrap()` in library code (workspace lint `unwrap_used = "deny"`).

## Review Focus

1. **A listing with DOS line endings or a trailing blank line** must build and lint exactly like the LF version. Pinned in Task 3 and Task 9.
2. **Spaces inside strings, `REM` text and a string containing a keyword** (`PRINT "GO TO 12"`) must survive tokenising, listing, `--fix` and the Task 12 check unchanged. Pinned in Task 5 and Task 12.
3. **A token that gets no ROM spaces next to text** (`PRINT a;INKEY$;RND`, `x<=y`) must not gain or lose a space in `list()` or `--fix`. Pinned in Task 4.
4. **`lint --fix` on a file that is already canonical** must leave the file byte-identical: same timestamp, not rewritten. Pinned in Task 9.
5. **A C64 listing using lowercase** (petcat's text convention) must not be flagged or mangled. Pinned in Task 6.

---

## Part A — Format198x (repo `Format198x/format198x`, branch `feat/basic-tokenisers`, one PR)

Create the worktree first:
`cd /Users/stevehill/Projects/198x/Format198x/format198x && git fetch -q && git worktree add ../format198x-bas -b feat/basic-tokenisers origin/main`.
All Part A paths are relative to that worktree.

### Task 1: Graduate both tokenisers unchanged

**Files:**
- Create: `crates/format198x-sinclair-zx-spectrum-bas/{Cargo.toml,README.md,CHANGELOG.md,src/*.rs}`
- Create: `crates/format198x-commodore-c64-bas/{Cargo.toml,README.md,CHANGELOG.md,src/*.rs}`

**Interfaces:**
- Produces:
  - `format198x_sinclair_zx_spectrum_bas::{tokenise, tokenise_listing, parse, BasicProgram { bytes: Vec<u8> }, ast}`;
  - `format198x_commodore_c64_bas::{tokenise, BasicProgram { bytes: Vec<u8> }}`.
  - The signatures are exactly those in Emu198x today.

- [ ] **Step 1: Copy the sources verbatim**

```bash
E=/Users/stevehill/Projects/198x/Emu198x/emu198x/crates
mkdir -p crates/format198x-sinclair-zx-spectrum-bas/src crates/format198x-commodore-c64-bas/src
cp $E/format-sinclair-zx-spectrum-bas/src/*.rs crates/format198x-sinclair-zx-spectrum-bas/src/
cp $E/format-commodore-c64-bas/src/*.rs crates/format198x-commodore-c64-bas/src/
```

- [ ] **Step 2: Write the manifests.** Follow `crates/format198x-sinclair-zx-spectrum-tap/Cargo.toml`.

```toml
# The ZX Spectrum BASIC tokeniser: numbered text listings to the bytes the ROM
# stores. Graduated from emu198x once Build198x needed to write BASIC tapes
# (umbrella decisions/formats-graduate-to-their-own-projects.md).
[package]
name = "format198x-sinclair-zx-spectrum-bas"
description = "ZX Spectrum BASIC — tokenise numbered text listings into stored program bytes, and list them as the ROM does."
version = "0.1.0"
edition.workspace = true
license.workspace = true
repository.workspace = true
readme = "README.md"
keywords = ["zx-spectrum", "sinclair", "basic", "tokeniser", "retro"]
categories = ["encoding", "parser-implementations"]

[lints]
workspace = true
```

```toml
# Commodore 64 BASIC V2: numbered text listings to a PRG loading at $0801.
# Graduated from emu198x once Build198x needed to write BASIC programs
# (umbrella decisions/formats-graduate-to-their-own-projects.md).
[package]
name = "format198x-commodore-c64-bas"
description = "Commodore 64 BASIC V2 — tokenise numbered text listings into a PRG, and list them as the machine does."
version = "0.1.0"
edition.workspace = true
license.workspace = true
repository.workspace = true
readme = "README.md"
keywords = ["c64", "commodore", "basic", "tokeniser", "retro"]
categories = ["encoding", "parser-implementations"]

[lints]
workspace = true
```

- [ ] **Step 3: Write the READMEs and CHANGELOGs.** Base each README on `crates/format198x-sinclair-zx-spectrum-tap/README.md`: what the crate does, a five-line usage example calling `tokenise_listing` (Spectrum) or `tokenise` (C64), and "Graduated from emu198x's `format-…-bas` crate". Each CHANGELOG starts with the release-plz header used by the TAP crate's CHANGELOG and has no entries.

- [ ] **Step 4: Run the carried tests; they must pass unchanged**

Run: `env -u RUSTUP_TOOLCHAIN cargo test -p format198x-sinclair-zx-spectrum-bas -p format198x-commodore-c64-bas`
Expected: PASS, with 28 Spectrum tests and 10 C64 tests (the counts Emu198x has today).

Run: `env -u RUSTUP_TOOLCHAIN cargo clippy -p format198x-sinclair-zx-spectrum-bas -p format198x-commodore-c64-bas --all-targets -- -D warnings`
Expected: clean. If the workspace's `unwrap_used = "deny"` fires inside `#[cfg(test)]` code, change those calls to `expect("…")` in test code only.

- [ ] **Step 5: Commit**

```bash
git add crates/format198x-sinclair-zx-spectrum-bas crates/format198x-commodore-c64-bas Cargo.lock
git commit -m "feat: graduate the Spectrum and C64 BASIC tokenisers from emu198x

Build198x needs to write BASIC tapes and PRGs, making it a consumer that
is not the producer, so both crates move here unchanged under the
family's crate names."
```

### Task 2: Record the graduation

**Files:**
- Create: `decisions/basic-tokenisers.md`

- [ ] **Step 1: Write the record.** Follow the shape of `decisions/oric-tap-is-a-standalone-format.md`. Status Active, dated 2026-09-26. Include:
  - why the crates graduated (Build198x's `basic` verb);
  - that they are text-listing tokenisers and listers, not parsers (the parser is the planned language project);
  - that Emu198x still has its own copies until it switches, tracked by the issue filed in Task 13.

- [ ] **Step 2: Commit**

```bash
git add decisions/basic-tokenisers.md
git commit -m "docs: record why the BASIC tokenisers graduated to Format198x"
```

### Task 3: Spectrum positioned token stream

**Files:**
- Modify: `crates/format198x-sinclair-zx-spectrum-bas/src/listing.rs`
- Modify: `crates/format198x-sinclair-zx-spectrum-bas/src/lib.rs` (re-exports)

**Interfaces:**
- Produces:

```rust
/// One stored piece of a line body, with where it came from in the source.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Piece {
    pub kind: PieceKind,
    /// Bytes this piece stores (a token byte, text, or text + 0x0E + 5-byte float).
    pub bytes: Vec<u8>,
    /// 0-based byte offset of the piece in the body text passed to `lex_line`.
    pub column: usize,
    /// The source text the piece came from.
    pub text: String,
}
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum PieceKind { Keyword(u8), Name, Number, Str, Rem, Space, Punct }
/// A listing line split into number and pieces.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct LexLine {
    pub number: u16,
    /// 0-based byte offset of the body within the source line.
    pub body_column: usize,
    pub pieces: Vec<Piece>,
}
pub fn lex_line(line: &str) -> Result<LexLine, String>;
```

  `tokenise_listing` becomes: `lex_line` for each non-blank line, then the concatenated `pieces[].bytes` plus `0x0D`, with the same header and ordering as today.

- [ ] **Step 1: Write the failing tests** (in `listing.rs` `mod tests`)

```rust
#[test]
fn lex_line_reports_kinds_and_columns() {
    // No space before THEN or REM: Task 5 stops storing those, and this test must hold either side of it.
    let line = lex_line("  20 IF a=1THEN PRINT \"x\":REM hi").expect("lex");
    assert_eq!(line.number, 20);
    assert_eq!(line.body_column, 5);
    let kinds: Vec<PieceKind> = line.pieces.iter().map(|p| p.kind).collect();
    assert_eq!(kinds, vec![
        PieceKind::Keyword(0xFA), PieceKind::Name, PieceKind::Punct, PieceKind::Number,
        PieceKind::Keyword(0xCB), PieceKind::Keyword(0xF5), PieceKind::Str,
        PieceKind::Punct, PieceKind::Keyword(0xEA), PieceKind::Rem,
    ]);
    let name = &line.pieces[1];
    assert_eq!((name.column, name.text.as_str()), (3, "a"));
}

#[test]
fn pieces_concatenate_to_the_stored_bytes() {
    for src in ["10 PRINT \"GO TO 12\": REM PRINT 99", "20 IF x<>2 THEN PRINT x", "10 LET cat=2"] {
        let lexed: Vec<u8> = lex_line(src).expect("lex").pieces.into_iter().flat_map(|p| p.bytes).collect();
        let stored = tokenise_listing(src).expect("tokenise").bytes;
        assert_eq!(&stored[4..stored.len() - 1], lexed.as_slice(), "{src}");
    }
}

#[test]
fn crlf_and_trailing_blank_lines_tokenise_like_lf() {
    let lf = tokenise_listing("10 PRINT 1\n20 GO TO 10\n").expect("lf").bytes;
    let crlf = tokenise_listing("10 PRINT 1\r\n20 GO TO 10\r\n\r\n").expect("crlf").bytes;
    assert_eq!(lf, crlf);
}
```

- [ ] **Step 2: Run to confirm they fail**

Run: `env -u RUSTUP_TOOLCHAIN cargo test -p format198x-sinclair-zx-spectrum-bas lex_line`
Expected: FAIL to compile, "cannot find function `lex_line`".

- [ ] **Step 3: Implement.** Refactor `tokenise_body` into `lex_body(source) -> Result<Vec<Piece>, String>`. Each branch that today does `out.extend…`/`out.push…` instead pushes a `Piece` with the matching kind, the `pos` at which the branch started as `column`, and `source[start..pos]` as `text`:
  - variable-name branch and alphabetic branch → `Name`;
  - quote branch → `Str`;
  - space → `Space`;
  - `<=`/`>=`/`<>` and keywords → `Keyword(token)`;
  - `REM` tail → a `Rem` piece holding the rest of the line;
  - number branch → `Number`, whose bytes are the spelling, `14` and the five float bytes;
  - everything else → `Punct`.

  The keyword branch's "skip one following space" stays as it is: the skipped space is not a piece. `lex_line` does the line-number parsing currently inline in `tokenise_listing`, using `raw.trim_end_matches('\r')` before the existing `trim`, and returns `body_column` as the offset of the body in the untrimmed line. `tokenise_listing` calls `lex_line` and keeps every existing error message word for word. Export `Piece`, `PieceKind`, `LexLine` and `lex_line` from `lib.rs`.

- [ ] **Step 4: Run the whole crate's tests**

Run: `env -u RUSTUP_TOOLCHAIN cargo test -p format198x-sinclair-zx-spectrum-bas`
Expected: PASS, with all the carried tests unchanged and the three new ones.

- [ ] **Step 5: Commit**

```bash
git add crates/format198x-sinclair-zx-spectrum-bas/src
git commit -m "feat: expose the Spectrum tokeniser's pieces with their source columns

Lint rules and the planned language server need to know where each
stored piece came from; tokenise_listing is now the concatenation."
```

### Task 4: Spectrum lister, checked against the real ROM

**Files:**
- Create: `crates/format198x-sinclair-zx-spectrum-bas/src/list.rs`
- Create: `crates/format198x-sinclair-zx-spectrum-bas/tests/rom_list.rs`
- Create: `crates/format198x-sinclair-zx-spectrum-bas/tests/fixtures/rom-list/{README.md,capture.mjs,cases.bas,cases.list}`
- Modify: `src/lib.rs` (`mod list; pub use list::{list, list_line};`)

**Interfaces:**
- Produces:
  - `pub fn list_line(number: u16, body: &[u8]) -> String`: one line as LIST prints it, without screen wrapping. `body` runs up to but excluding the `0x0D`.
  - `pub fn list(program: &[u8]) -> Result<Vec<String>, String>`: every line of a stored program. The input is the bytes `tokenise_listing` returns: number (big-endian), length (little-endian), body, `0x0D`.

- [ ] **Step 1: Write the fixture cases.** `cases.bas` is a listing that begins with `1 STOP`, so running it never executes anything else. It has one line per case:
  - every token code `0xA5`–`0xFF` in a statement that shows its leading and trailing spacing (a function after `PRINT`, an operator word between values, a statement after `:`);
  - `PRINT a;INKEY$;RND`, `PRINT PI`, `IF a<=b THEN`, `CHR$ (147)`;
  - strings containing keywords;
  - `REM` with spaces;
  - line numbers 1, 10, 100, 1000 and 9999;
  - 40 real lines taken from `Code198x/code-samples/sinclair-zx-spectrum/basic/*/unit-*/*.bas`, renumbered into free numbers.

  Keep the file under 22 lines per screen page: the capture pages with `LIST n`.

- [ ] **Step 2: Write `capture.mjs` and capture from the ROM.** The script:
  - loads `@emu198x/zx-spectrum` from `$EMU198X_ZX_SPECTRUM_PKG`, a directory;
  - creates `Spectrum.createHeadless(rom)` with the genuine 48K ROM from `$SPECTRUM_48K_ROM`, and refuses any file whose SHA1 is not `5ea7c2b824672e914525d1d5c419d71b84a426a2`;
  - runs 3000 ms, then `runBasic(cases.bas)`, which stops at line 1;
  - for each line number, types `LIST n` + ENTER and reads `query('screen.text.lines')` (a JSON array of 32-character rows);
  - takes the rows of line *n* (from its first row up to the row where the next line number starts, or a blank row), joins them, and trims the end;
  - writes one line per case to `cases.list`.

  The key-tapping helper holds each key down for 100 ms, releases it, and waits 200 ms, as proven in the 2026-09-26 spike. The README records the ROM's SHA1, the package version, the date and the command:

```bash
EMU198X_ZX_SPECTRUM_PKG=/Users/stevehill/Projects/198x/Code198x/website-bpi/node_modules/@emu198x/zx-spectrum \
SPECTRUM_48K_ROM=$HOME/.emu198x/roms/sinclair-zx-spectrum-48k/48.rom \
node crates/format198x-sinclair-zx-spectrum-bas/tests/fixtures/rom-list/capture.mjs
```

  Expected: `cases.list` has as many lines as `cases.bas`. `  10 PRINT CHR$ (147)` appears for a source line `10 PRINT CHR$(147)`.

- [ ] **Step 3: Write the failing test** (`tests/rom_list.rs`)

```rust
use format198x_sinclair_zx_spectrum_bas::{list, tokenise_listing};

#[test]
fn lister_matches_the_rom_for_every_captured_case() {
    let source = include_str!("fixtures/rom-list/cases.bas");
    let expected: Vec<&str> = include_str!("fixtures/rom-list/cases.list").lines().collect();
    let program = tokenise_listing(source).expect("cases tokenise");
    let listed = list(&program.bytes).expect("list");
    assert_eq!(listed.len(), expected.len());
    for (got, want) in listed.iter().zip(&expected) {
        assert_eq!(got.trim_end(), *want, "ROM lists `{want}`");
    }
}
```

  Run: `env -u RUSTUP_TOOLCHAIN cargo test -p format198x-sinclair-zx-spectrum-bas --test rom_list`
  Expected: FAIL to compile, "no function `list`".

- [ ] **Step 4: Implement `list.rs`.** The rules come from the ROM's `PO-TOKENS`/`PO-SEARCH`/`PO-CHAR` routines (Logan and O'Hara, *The Complete Spectrum ROM Disassembly*, 0C10–0C54 and 0B6A):

```rust
//! LIST as the 48K ROM prints it: PO-TOKENS/PO-SEARCH (0C10–0C54) for token
//! spacing, PO-CHAR (0B6A) for the leading-space flag, OUT-NUM-2 for the
//! four-column line number. Screen wrapping is not modelled.

/// Token text in ROM table order, codes 0xA5 (RND) to 0xFF (COPY).
const TOKENS: [&str; 91] = [
    "RND", "INKEY$", "PI", "FN", "POINT", "SCREEN$", "ATTR", "AT", "TAB", "VAL$",
    "CODE", "VAL", "LEN", "SIN", "COS", "TAN", "ASN", "ACS", "ATN", "LN", "EXP",
    "INT", "SQR", "SGN", "ABS", "PEEK", "IN", "USR", "STR$", "CHR$", "NOT", "BIN",
    "OR", "AND", "<=", ">=", "<>", "LINE", "THEN", "TO", "STEP", "DEF FN", "CAT",
    "FORMAT", "MOVE", "ERASE", "OPEN #", "CLOSE #", "MERGE", "VERIFY", "BEEP",
    "CIRCLE", "INK", "PAPER", "FLASH", "BRIGHT", "INVERSE", "OVER", "OUT", "LPRINT",
    "LLIST", "STOP", "READ", "DATA", "RESTORE", "NEW", "BORDER", "CONTINUE", "DIM",
    "REM", "FOR", "GO TO", "GO SUB", "INPUT", "LOAD", "LIST", "LET", "PAUSE", "NEXT",
    "POKE", "PRINT", "PLOT", "RUN", "SAVE", "RANDOMIZE", "IF", "CLS", "DRAW",
    "CLEAR", "RETURN", "COPY",
];

/// One line as LIST prints it, without screen wrapping.
pub fn list_line(number: u16, body: &[u8]) -> String {
    let mut out = format!("{number:>4}");
    // FLAGS bit 0 after OUT-LINE: reset, so a leading space is allowed.
    let mut suppress = false;
    let mut i = 0;
    while i < body.len() {
        let b = body[i];
        if b == 0x0E {
            i += 6; // hidden five-byte number
            continue;
        }
        if b >= 0xA5 {
            let index = usize::from(b - 0xA5);
            let text = TOKENS[index];
            let first_is_letter = text.as_bytes()[0].is_ascii_alphabetic();
            if index >= 0x20 && first_is_letter && !suppress {
                out.push(' ');
            }
            out.push_str(text);
            let last = text.as_bytes()[text.len() - 1];
            if (last == b'$' || last >= b'A') && index >= 3 {
                out.push(' ');
                suppress = true;
            } else {
                suppress = false;
            }
        } else {
            out.push(char::from(b));
            suppress = b == b' ';
        }
        i += 1;
    }
    out
}

/// Every line of a stored program, as LIST prints them.
///
/// # Errors
/// Returns an error if the bytes end partway through a line.
pub fn list(program: &[u8]) -> Result<Vec<String>, String> {
    let mut lines = Vec::new();
    let mut at = 0;
    while at < program.len() {
        let header = program.get(at..at + 4).ok_or("program ends inside a line header")?;
        let number = u16::from_be_bytes([header[0], header[1]]);
        let length = usize::from(u16::from_le_bytes([header[2], header[3]]));
        let body = program.get(at + 4..at + 4 + length).ok_or("program ends inside a line")?;
        let body = body.strip_suffix(&[0x0D]).unwrap_or(body);
        lines.push(list_line(number, body));
        at += 4 + length;
    }
    Ok(lines)
}
```

  Add unit tests in `list.rs`, all listing tokenised input:
  - `"10 PRINT a;INKEY$;RND"` gives `"  10 PRINT a;INKEY$;RND"`;
  - `"10 IF x<=y THEN STOP"` gives `"  10 IF x<=y THEN STOP "`;
  - `"9999 REM  two  spaces"` gives `"9999 REM  two  spaces"`;
  - `TOKENS[0xF5 - 0xA5] == "PRINT"`, `TOKENS[0xCB - 0xA5] == "THEN"` and `TOKENS[0xEC - 0xA5] == "GO TO"`.

- [ ] **Step 5: Run the tests**

Run: `env -u RUSTUP_TOOLCHAIN cargo test -p format198x-sinclair-zx-spectrum-bas`
Expected: PASS, including `rom_list`. A mismatch means the implementation or the table is wrong. The ROM capture is the authority; never edit `cases.list` by hand.

- [ ] **Step 6: Commit**

```bash
git add crates/format198x-sinclair-zx-spectrum-bas
git commit -m "feat: list Spectrum programs exactly as the 48K ROM does

The rules follow PO-TOKENS and PO-SEARCH, and the tests compare against
LIST output captured from the genuine ROM in Emu198x, so the ROM rather
than our reading of it is the authority."
```

### Task 5: Stop storing the space before a keyword

**Files:**
- Modify: `crates/format198x-sinclair-zx-spectrum-bas/src/listing.rs`
- Test: same file, `mod tests`

**Interfaces:**
- Produces: `pub fn listed_form(source: &str) -> Result<Vec<(usize, String)>, String>`, which returns (0-based source line index, the line as LIST would print it) for each non-blank line, in source order. The lint uses it in Part B.

- [ ] **Step 1: Write the failing tests**

```rust
#[test]
fn a_space_the_rom_supplies_before_a_keyword_is_not_stored() {
    let spaced = tokenise_listing("10 IF a=1 THEN STOP").expect("spaced").bytes;
    let tight = tokenise_listing("10 IF a=1THEN STOP").expect("tight").bytes;
    assert_eq!(spaced, tight);
}

#[test]
fn spaces_the_rom_does_not_supply_are_kept() {
    // `<=` gets no leading space from the ROM, and strings/REM are content.
    let b = tokenise_listing("10 IF a <= b THEN PRINT \"a  THEN\": REM  x").expect("t").bytes;
    assert!(b.windows(2).any(|w| w == [b' ', 0xC7]), "space before <= kept");
    assert!(b.windows(7).any(|w| w == b"a  THEN"), "string untouched");
}

#[test]
fn listed_form_round_trips_canonical_lines() {
    let src = "  10 PRINT CHR$ (147)\n  20 IF a=1 THEN GO TO 20\n  30 PRINT \"a = b\";INKEY$;RND";
    let listed = listed_form(src).expect("listed");
    for ((_, got), want) in listed.iter().zip(src.lines()) {
        assert_eq!(got.trim_end(), want);
    }
}
```

Run: `env -u RUSTUP_TOOLCHAIN cargo test -p format198x-sinclair-zx-spectrum-bas space`
Expected: `a_space_the_rom_supplies_before_a_keyword_is_not_stored` FAILS (a stored `0x20` before `0xCB`), and `listed_form` fails to compile.

- [ ] **Step 2: Implement.** In `lex_body`'s keyword branch, before pushing the keyword piece, check whether the ROM would print a leading space before this token: `token >= 0xC5` and the token text starts with a letter. That is the same rule as `list_line`; take it from a shared `fn rom_leading_space(token: u8) -> bool` in `list.rs`. If so, and the last pushed piece is a `Space` piece of exactly one space, pop that piece. Implement `listed_form` as: for each non-blank line, `lex_line`, concatenate the pieces' bytes, then `list_line(number, &bytes)`.

- [ ] **Step 3: Run the crate's tests**

Run: `env -u RUSTUP_TOOLCHAIN cargo test -p format198x-sinclair-zx-spectrum-bas`
Expected: PASS, including `rom_list` (the ROM output is unchanged by dropping an invisible space).

- [ ] **Step 4: Commit**

```bash
git add crates/format198x-sinclair-zx-spectrum-bas/src
git commit -m "fix: stop storing the space the ROM prints before a keyword

The ROM adds a leading space before THEN, TO, AND and similar tokens
when it lists them, so a source space there cost a byte and changed
nothing on screen."
```

### Task 6: C64 positioned token stream and lister

**Files:**
- Modify: `crates/format198x-commodore-c64-bas/src/lib.rs`
- Create: `crates/format198x-commodore-c64-bas/src/list.rs`

**Interfaces:**
- Produces:
  - `pub struct Piece { pub kind: PieceKind, pub bytes: Vec<u8>, pub column: usize, pub text: String }`;
  - `pub enum PieceKind { Keyword(u8), Name, Number, Str, Rem, Space, Punct }`;
  - `pub struct LexLine { pub number: u16, pub body_column: usize, pub pieces: Vec<Piece> }`;
  - `pub fn lex_line(line: &str) -> Result<LexLine, String>`;
  - `pub fn list_line(number: u16, body: &[u8]) -> String`;
  - `pub fn list(prg: &[u8]) -> Result<Vec<String>, String>`, which takes a PRG including its load address;
  - `pub fn listed_form(source: &str) -> Result<Vec<(usize, String)>, String>`.

- [ ] **Step 1: Write the failing tests**

```rust
#[test]
fn c64_lists_stored_characters_without_adding_spaces() {
    let prg = tokenise("10 PRINT CHR$(147)\n20 FORI=1TO10:NEXT").expect("t").bytes;
    assert_eq!(list(&prg).expect("list"), vec!["10 PRINT CHR$(147)", "20 FORI=1TO10:NEXT"]);
}

#[test]
fn lowercase_source_lists_uppercase_and_round_trips_uppercase() {
    let prg = tokenise("10 print \"hi\"").expect("t").bytes;
    assert_eq!(list(&prg).expect("list"), vec!["10 PRINT \"HI\""]);
    let upper = tokenise("10 PRINT \"HI\"").expect("t").bytes;
    assert_eq!(prg, upper);
}

#[test]
fn c64_listed_form_flags_extra_space_after_the_number() {
    let listed = listed_form("10  PRINT 1").expect("listed");
    assert_eq!(listed[0].1, "10  PRINT 1"); // the second space is stored, so it lists
    assert_eq!(listed_form("10 PRINT 1").expect("l")[0].1, "10 PRINT 1");
}
```

Run: `env -u RUSTUP_TOOLCHAIN cargo test -p format198x-commodore-c64-bas`
Expected: FAIL to compile (`list`, `listed_form` missing).

- [ ] **Step 2: Implement.**
  - Refactor `tokenise_line` into `lex_body` producing `Piece`s, as in Task 3. `tokenise` concatenates their bytes, so its output stays byte-identical.
  - `list_line`:
    - writes the number, then a space;
    - writes each byte: bytes `0x80`–`0xCB` become the keyword text from `KEYWORDS` (build a reverse lookup `fn keyword_text(code: u8) -> Option<&'static str>`); other bytes go through `petscii_to_ascii`, the inverse of the existing `ascii_to_petscii` for the range it maps;
    - does not tokenise inside strings or after `REM`, because those bytes are stored as characters.
  - `list` walks the PRG from offset 2, reads each line's link, number and body up to `0x00`, and stops at a zero link.

- [ ] **Step 3: Run the tests**

Run: `env -u RUSTUP_TOOLCHAIN cargo test -p format198x-commodore-c64-bas`
Expected: PASS, including the carried tests.

- [ ] **Step 4: Commit**

```bash
git add crates/format198x-commodore-c64-bas/src
git commit -m "feat: expose C64 BASIC pieces with source columns, and list programs as the C64 does"
```

### Task 7: Open the Format198x PR and release

- [ ] **Step 1:** `env -u RUSTUP_TOOLCHAIN cargo test --workspace && env -u RUSTUP_TOOLCHAIN cargo clippy --workspace --all-targets -- -D warnings`. Expected: PASS and clean.
- [ ] **Step 2:** Push and open the PR:

```bash
git push -u origin feat/basic-tokenisers
gh pr create --base main --title "feat: graduate the BASIC tokenisers, with listers checked against the ROM"
```

  The body gives:
  - the spec link;
  - the graduation reason;
  - the ROM-capture method;
  - the one behaviour change (Task 5).
- [ ] **Step 3:** When CI is green, merge (merges are delegated for green PRs). Before merging release-plz's release PR, write each new crate's CHANGELOG entry by hand. Merging it publishes `0.1.0` of both crates. Check with `cargo search format198x-sinclair-zx-spectrum-bas` and `cargo search format198x-commodore-c64-bas`. Expected: both listed at 0.1.0 (or the version release-plz chose).

---

## Part B — Build198x (repo `Build198x/build198x`, branch `feat/basic-verb`)

Steve's checkout is on a stale branch; leave it alone and use a worktree:
`cd /Users/stevehill/Projects/198x/Build198x/build198x && git fetch -q && git worktree add ../build198x-basic -b feat/basic-verb origin/main`.

### Task 8: Demand gate and the build verb

**Files:**
- Create: `decisions/demand-gate-basic.md`
- Create: `crates/build198x/src/basic/mod.rs`
- Modify: `crates/build198x/src/lib.rs` (`pub mod basic;`)
- Modify: `crates/build198x/Cargo.toml`: add `format198x-sinclair-zx-spectrum-bas = "0.1"`, `format198x-commodore-c64-bas = "0.1"` and `format198x-sinclair-zx-spectrum-tap = "0.1"`, with a comment matching the other Format198x deps.
- Modify: `crates/build198x/src/main.rs`: add `"basic" => basic_command(rest)` to the dispatch, add a usage line to `top_usage`, and add `basic_command`.
- Test: `crates/build198x/tests/basic_cli.rs`

**Interfaces:**
- Produces, in `build198x::basic`:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Machine { SinclairZxSpectrum, CommodoreC64 }
impl Machine {
    pub fn from_id(id: &str) -> Option<Self>;           // "sinclair-zx-spectrum" | "commodore-c64"
    pub fn id(self) -> &'static str;
    pub fn extension(self) -> &'static str;             // "tap" | "prg"
}
pub struct Built { pub bytes: Vec<u8>, pub lines: usize, pub program_length: usize, pub autorun: Option<u16> }
/// Tokenise and package one listing. `name` is the tape header name (Spectrum).
pub fn build(machine: Machine, source: &str, name: &str, autorun: bool) -> Result<Built, String>;
```

- [ ] **Step 1: Write the decision record.** Base it on `decisions/demand-gate-tape-master.md`'s shape and the spec's Stage 2 text:
  - **need:** about 170 Spectrum BASIC lesson pages with no runnable build;
  - **in:** stock-ROM tokenisation, a self-starting TAP, a `$0801` PRG, lint;
  - **out:** TZX, disk images, 128K-only keywords, loading screens and other machines;
  - **next:** the language project, as a separate sub-project.

- [ ] **Step 2: Write the failing CLI tests** (`tests/basic_cli.rs`). Copy `TempDir` and `run_in` from `tests/cli.rs`.

```rust
#[test]
fn spectrum_listing_builds_an_autorunning_tap() {
    let dir = TempDir::new("bas-zx");
    std::fs::write(dir.path().join("hello.bas"), "  10 PRINT \"HELLO\"\n  20 GO TO 10\n").unwrap();
    let (code, _, err) = run_in(dir.path(), &["basic", "hello.bas", "--machine", "sinclair-zx-spectrum", "-o", "hello.tap"]);
    assert_eq!(code, 0, "{err}");
    let tap = std::fs::read(dir.path().join("hello.tap")).unwrap();
    let blocks = format198x_sinclair_zx_spectrum_tap::decode(&tap).unwrap();
    assert_eq!(blocks.len(), 2);
    let header = &blocks[0].data;               // flag 0x00 header block
    assert_eq!(&header[2..12], b"hello     "); // name from -o stem
    assert_eq!(u16::from_le_bytes([header[14], header[15]]), 10); // autorun line
}

#[test]
fn c64_listing_builds_a_prg_at_0801() {
    let dir = TempDir::new("bas-c64");
    std::fs::write(dir.path().join("a.bas"), "10 PRINT CHR$(147)\n").unwrap();
    let (code, _, err) = run_in(dir.path(), &["basic", "a.bas", "--machine", "commodore-c64", "-o", "a.prg"]);
    assert_eq!(code, 0, "{err}");
    assert_eq!(&std::fs::read(dir.path().join("a.prg")).unwrap()[..2], &[0x01, 0x08]);
}

#[test]
fn wrong_extension_and_bad_lines_fail_and_write_nothing() {
    let dir = TempDir::new("bas-err");
    std::fs::write(dir.path().join("a.bas"), "  10 PRINT 1\nPRINT 2\n").unwrap();
    let (code, _, _) = run_in(dir.path(), &["basic", "a.bas", "--machine", "commodore-c64", "-o", "a.tap"]);
    assert_ne!(code, 0);
    let (code, _, err) = run_in(dir.path(), &["basic", "a.bas", "--machine", "sinclair-zx-spectrum", "-o", "a.tap"]);
    assert_ne!(code, 0);
    assert!(err.contains("a.bas:2:"), "{err}");
    assert!(!dir.path().join("a.tap").exists());
}

#[test]
fn no_autorun_writes_line_32768() {
    let dir = TempDir::new("bas-noauto");
    std::fs::write(dir.path().join("a.bas"), "  10 STOP\n").unwrap();
    let (code, _, err) = run_in(dir.path(), &["basic", "a.bas", "--machine", "sinclair-zx-spectrum", "-o", "a.tap", "--no-autorun"]);
    assert_eq!(code, 0, "{err}");
    let tap = std::fs::read(dir.path().join("a.tap")).unwrap();
    let header = &format198x_sinclair_zx_spectrum_tap::decode(&tap).unwrap()[0].data;
    assert_eq!(u16::from_le_bytes([header[14], header[15]]), 32768);
}
```

  Check the TAP crate's `TapBlock` field names and `Header::new` parameter order against `format198x-sinclair-zx-spectrum-tap/src/lib.rs` (lines 56–204) before relying on `.data` and the offsets. Adjust the test to the crate's accessors if they differ; the offsets are the standard 17-byte ROM header after the flag byte. Add `format198x-sinclair-zx-spectrum-tap` to `[dev-dependencies]` if it isn't already in `[dependencies]`.

  Run: `env -u RUSTUP_TOOLCHAIN cargo test -p build198x --test basic_cli`
  Expected: FAIL (`unknown command basic`).

- [ ] **Step 3: Implement `basic/mod.rs`**

```rust
//! `build198x basic`: a numbered BASIC listing to the file its machine loads.
//! Spectrum: a ROM header + data block pair (TAP), autorun at the first line.
//! C64: a PRG at $0801, which the tokeniser already emits.

use format198x_sinclair_zx_spectrum_tap::{Header, HeaderKind, TapBlock, encode};

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Machine { SinclairZxSpectrum, CommodoreC64 }

impl Machine {
    pub fn from_id(id: &str) -> Option<Self> {
        match id {
            "sinclair-zx-spectrum" => Some(Self::SinclairZxSpectrum),
            "commodore-c64" => Some(Self::CommodoreC64),
            _ => None,
        }
    }
    pub fn id(self) -> &'static str {
        match self { Self::SinclairZxSpectrum => "sinclair-zx-spectrum", Self::CommodoreC64 => "commodore-c64" }
    }
    pub fn extension(self) -> &'static str {
        match self { Self::SinclairZxSpectrum => "tap", Self::CommodoreC64 => "prg" }
    }
}

pub struct Built { pub bytes: Vec<u8>, pub lines: usize, pub program_length: usize, pub autorun: Option<u16> }

/// # Errors
/// Returns the tokeniser's message, which names the source line.
pub fn build(machine: Machine, source: &str, name: &str, autorun: bool) -> Result<Built, String> {
    let lines = source.lines().filter(|l| !l.trim().is_empty()).count();
    match machine {
        Machine::SinclairZxSpectrum => {
            let program = format198x_sinclair_zx_spectrum_bas::tokenise_listing(source)?;
            let length = u16::try_from(program.bytes.len()).map_err(|_| "BASIC program too large")?;
            let first = u16::from_be_bytes([program.bytes[0], program.bytes[1]]);
            let start = if autorun { first } else { 32768 };
            let bytes = encode(&[
                Header::new(HeaderKind::Program, name, length, start, length).block(),
                TapBlock::data(program.bytes),
            ]);
            Ok(Built { bytes, lines, program_length: usize::from(length), autorun: autorun.then_some(first) })
        }
        Machine::CommodoreC64 => {
            let program = format198x_commodore_c64_bas::tokenise(source)?;
            let program_length = program.bytes.len().saturating_sub(2);
            Ok(Built { bytes: program.bytes, lines, program_length, autorun: None })
        }
    }
}
```

  In `main.rs`, `basic_command(args)` does the following:
  - dispatches `lint` to `basic_lint(&args[1..])` (Task 9);
  - handles `--help`;
  - otherwise parses `<in.bas>`, `--machine`, `-o`/`--output`, `--name`, `--no-autorun` and `--format text|json`, using the `adf_master` loop style;
  - rejects an `-o` extension other than `machine.extension()`;
  - defaults the name to the `-o` file stem, cut to 10 characters;
  - runs lint on the source (Task 9: for now, a call that returns no findings) and refuses to build if there are findings;
  - calls `build`, prefixing errors with `"{in}:"` and rewriting the tokenisers' `Source line N:`/`Line N:` prefix as `N:`;
  - writes the output atomically (temp file plus rename, as `image` does), with no clobber check, since Makefiles rebuild;
  - prints the text or JSON report. JSON keys: `tool_version`, `machine`, `input`, `output`, `lines`, `program_length`, `autorun`.

- [ ] **Step 4: Run the tests**

Run: `env -u RUSTUP_TOOLCHAIN cargo test -p build198x --test basic_cli && env -u RUSTUP_TOOLCHAIN cargo clippy -p build198x --all-targets -- -D warnings`
Expected: PASS and clean.

- [ ] **Step 5: Commit**

```bash
git add decisions/demand-gate-basic.md crates/build198x/src/basic crates/build198x/src/lib.rs crates/build198x/src/main.rs crates/build198x/Cargo.toml crates/build198x/tests/basic_cli.rs Cargo.lock
git commit -m "feat: add the basic verb, building Spectrum tapes and C64 PRGs from listings

Code198x's BASIC lessons had no runnable build because no family tool
tokenised a listing. The verb uses the tokenisers that graduated to
Format198x."
```

### Task 9: `basic lint` and `--fix`

**Files:**
- Create: `crates/build198x/src/basic/lint.rs`
- Modify: `crates/build198x/src/basic/mod.rs` (`pub mod lint;`)
- Modify: `crates/build198x/src/main.rs` (`basic_lint`; the build path calls `lint::check`)
- Test: `crates/build198x/tests/basic_lint.rs`

**Interfaces:**
- Produces:

```rust
pub struct Finding { pub line: usize, pub column: usize, pub rule: &'static str, pub message: String }
/// All findings for one listing, sorted by line then column. Lines/columns are 1-based.
pub fn check(machine: Machine, source: &str) -> Result<Vec<Finding>, String>;
/// The source with every line replaced by its listed form. Returns None when nothing changes.
pub fn fix(machine: Machine, source: &str) -> Result<Option<String>, String>;
```

- [ ] **Step 1: Write the failing tests**

```rust
use build198x::basic::{Machine, lint::{check, fix}};
const ZX: Machine = Machine::SinclairZxSpectrum;
const C64: Machine = Machine::CommodoreC64;
fn rules(m: Machine, src: &str) -> Vec<(usize, &'static str)> {
    check(m, src).unwrap().into_iter().map(|f| (f.line, f.rule)).collect()
}

#[test] fn listing_form_flags_what_list_would_change() {
    assert_eq!(rules(ZX, "10 PRINT CHR$(147)\n"), vec![(1, "listing-form")]);
    assert!(rules(ZX, "  10 PRINT CHR$ (147)\n").is_empty());
}
#[test] fn stored_spaces_outside_strings_are_flagged() {
    // LIST shows these exactly as typed, so listing-form cannot see them.
    assert_eq!(rules(ZX, "  10 LET n = n + 1\n"), vec![(1, "stored-space"); 4]);
    assert!(rules(ZX, "  10 PRINT \"a = b\": REM x = y\n").is_empty());
    // A space inside a numeric variable name is part of the name the ROM allows.
    assert!(rules(ZX, "  10 LET my score=0\n").is_empty());
}
#[test] fn fix_rewrites_to_the_listed_form_and_is_idempotent() {
    let fixed = fix(ZX, "10 PRINT CHR$(147)\r\n20 IF a = 1 THEN STOP\n").unwrap().unwrap();
    assert_eq!(fixed, "  10 PRINT CHR$ (147)\n  20 IF a=1 THEN STOP\n");
    assert_eq!(fix(ZX, &fixed).unwrap(), None);
}
#[test] fn string_var_names_must_be_one_letter() {
    assert_eq!(rules(ZX, "  10 LET name$=\"x\"\n"), vec![(1, "string-var-name")]);
    assert!(rules(ZX, "  10 LET n$=\"name$\": PRINT CHR$ 65\n").is_empty());
}
#[test] fn keyword_named_variables_are_flagged() {
    assert_eq!(rules(ZX, "  10 LET ink=2\n"), vec![(1, "keyword-var-name")]);
    assert!(rules(ZX, "  10 LET inky=2\n").is_empty());
}
#[test] fn c64_two_letter_clash() {
    let found = rules(C64, "10 SCORE=1\n20 SCALE=2\n30 SC$=\"A\"\n");
    assert_eq!(found, vec![(2, "var-name-clash")]);
    assert!(rules(C64, "10 print \"lower\"\n").is_empty());
}
#[test] fn line_order() {
    assert_eq!(rules(ZX, "  20 STOP\n  10 STOP\n  10 STOP\n"),
               vec![(2, "line-order"), (3, "line-order")]);
}
```

  In `tests/basic_cli.rs`, add a test that `fix` leaves an already-canonical file untouched. Run `basic lint --fix` on a canonical file and assert its modified time and bytes are unchanged.

  Run: `env -u RUSTUP_TOOLCHAIN cargo test -p build198x --test basic_lint`
  Expected: FAIL to compile.

- [ ] **Step 2: Implement `lint.rs`.** The rules:
  - **`listing-form`:** for each `(index, listed)` from the dialect's `listed_form(source)`, compare `listed.trim_end()` with the source line after `trim_end()`, stripping any `\r`. Report column 1 with the message `` LIST shows `{listed}` ``. The column is 1 because the whole line is compared.
  - **`stored-space`** (Spectrum): a `Space` piece from `lex_line` that is *not* between two `Name` pieces. By Task 5, `Space` pieces are the spaces the ROM stores and lists, outside strings and `REM`, other than the one it supplies next to a keyword. A space between two name pieces is part of a numeric variable name, which the ROM allows and ignores when it looks the name up (`LET now we=6: PRINT nowwe` prints 6; Vickers, *ZX Spectrum BASIC Programming*, 1983, chapters 7 and 24). It is left alone. One finding per flagged piece, at its column. Message: `a space here is stored and listed; the Spectrum's own display spacing needs none`. For the Spectrum, `fix` drops the flagged pieces before listing each line, so its output passes both `stored-space` and `listing-form`.
  - **`string-var-name`** (Spectrum): for each line's `lex_line` pieces, a `Name` piece whose text ends in `$` and has more than two characters. Column = `body_column + piece.column + 1`. Message: `` string variables are one letter and $ (`a$`); the ROM rejects `{text}` ``.
  - **`keyword-var-name`** (Spectrum): a `Name` piece directly after a `LET`/`FOR`/`NEXT`/`INPUT`/`READ`/`DIM` keyword piece (codes `0xF1 0xEB 0xF3 0xEE 0xE3 0xE9`), ignoring `Space` pieces, whose uppercase text equals a keyword's name in `TOKENS`. Message: `` `{text}` is also a keyword; elsewhere in the program the tokeniser may store it as one ``. Export `TOKENS` from the Spectrum crate as `pub const KEYWORD_NAMES: [&str; 91]` if it isn't public. That is a patch release of the crate: add it in Part A Task 4 if you get there first.
  - **`var-name-clash`** (C64): collect `Name` pieces outside strings and REM. Key = the first two characters, uppercased, plus the type suffix (`$`, `%` or none). Report the first occurrence of each second or later distinct name with that key. Message: `` `{b}` is the same variable as `{a}`; BASIC V2 reads only the first two characters ``.
  - **`line-order`:** walk the lines in source order, tracking the highest number so far. A number equal to one already seen, or lower than the highest, is flagged at its line, column 1.

  `check` returns `Err` only for a line the tokeniser cannot read, with the tokeniser's message. `fix` builds the new text from `listed_form` (Spectrum: after dropping the flagged `Space` pieces, as above): each listed line with its end trimmed, joined with `\n`, with a trailing `\n`. It returns `None` when that equals the source normalised the same way.

  In `main.rs`, `basic_lint(args)`:
  - takes `--machine`, one or more paths, `--fix` and `--format`;
  - prints `path:line:column: rule: message` per finding;
  - with `--fix`, writes only files whose `fix` returned `Some`, then re-checks and prints what remains;
  - exits 1 on any remaining finding and 2 on usage errors.

  The build path calls `check` and refuses to build on findings, printing them in the same form.

- [ ] **Step 3: Run the tests**

Run: `env -u RUSTUP_TOOLCHAIN cargo test -p build198x && env -u RUSTUP_TOOLCHAIN cargo clippy -p build198x --all-targets -- -D warnings`
Expected: PASS and clean. In `basic_cli.rs`, the Task 8 fixtures are already canonical (`  10 PRINT "HELLO"` and `  20 GO TO 10`). The C64 fixture `10 PRINT CHR$(147)` is canonical for the C64.

- [ ] **Step 4: Commit**

```bash
git add crates/build198x/src/basic crates/build198x/src/main.rs crates/build198x/tests/basic_lint.rs crates/build198x/tests/basic_cli.rs
git commit -m "feat: lint BASIC listings against what the machine lists, and refuse to build failures

Six rules from real mistakes in Code198x's samples: the listed form
and stored spaces (both with --fix), long Spectrum string names,
keyword-named variables, C64 two-letter clashes, and line order."
```

### Task 10: C64 corpus against petcat, docs, release

**Files:**
- Create: `crates/build198x/tests/basic_corpus.rs`
- Modify: `README.md` (a `basic` section), and the `top_usage`/`basic --help` text

- [ ] **Step 1: Write the corpus test.** It's ignored by default, like `tests/wild.rs`.

```rust
/// Every C64 listing in code-samples must tokenise byte-identically to VICE's
/// petcat. petcat reads uppercase as shifted graphics, so it gets the
/// listing lowercased. Needs CODE_SAMPLES_PATH and petcat on PATH.
#[test]
#[ignore = "needs CODE_SAMPLES_PATH and petcat"]
fn c64_listings_match_petcat() { /* walk $CODE_SAMPLES_PATH/commodore-64/basic for *.bas;
    for each: build(C64, src) vs `petcat -w2 -o tmp.prg -- lowercased.bas`; collect mismatches;
    assert!(mismatches.is_empty(), "{mismatches:#?}") */ }
```

  Write the body as the comment describes, using `std::process::Command` and `TempDir`.

  Run: `CODE_SAMPLES_PATH=/Users/stevehill/Projects/198x/Code198x/code-samples env -u RUSTUP_TOOLCHAIN cargo test -p build198x --test basic_corpus -- --ignored`
  Expected: PASS for all 86 listings (the 2026-09-26 spike matched 86 of 86).

- [ ] **Step 2: Write the README section.** Cover the command, the two machines, the listed-form rule with a Spectrum before/after example, the lint rules table from the spec, and the `lint --fix` workflow.

- [ ] **Step 3: Commit, push and open the PR**

```bash
git add crates/build198x/tests/basic_corpus.rs README.md crates/build198x/src/main.rs
git commit -m "test: check every C64 sample listing against petcat; docs: the basic verb"
git push -u origin feat/basic-verb
gh pr create --base main --title "feat: build198x basic — BASIC listings to tapes and PRGs, with lint"
```

- [ ] **Step 4: Release.** Merge when green, then merge the release PR after writing its changelog by hand. Confirm the release has archives: `gh release view build198x-v<new> -R build198x/build198x`. Expected: four target archives listed.

---

## Part C — Code198x code-samples and website

### Task 11: Check against the ROM on real tapes

- [ ] **Step 1:** For one final-unit listing per Spectrum BASIC game (23 folders under `code-samples/sinclair-zx-spectrum/basic/`), build a tape with the released `build198x basic`. The listing must already be canonical, so do this after Task 12. Boot each in Emu198x headless with the genuine ROM, using the Task 4 capture script's loader: `load('tape-1','tap',bytes)`, then `autoload(400)`. Read `screen.text.lines` after 5 seconds. Expected: each shows its program's first screen text, not a report code such as `C Nonsense in BASIC`. Record the table in the code-samples PR.

### Task 12: Rewrite the Spectrum listings to the listed form

**Files:**
- Modify: every `code-samples/sinclair-zx-spectrum/**/*.bas` that `lint` flags
- Create (one-off, kept in the PR description only): the before/after byte check

- [ ] **Step 1:** Create a code-samples branch: `cd /Users/stevehill/Projects/198x/Code198x/code-samples && git fetch -q && git worktree add ../code-samples-bas -b fix/basic-listed-form origin/main`.
- [ ] **Step 2:** Record the stored bytes before, in the scratchpad. For each `.bas`, save `tokenise_listing(src)` using the Format198x crate *before* Task 5's fix: build a tiny scratch binary against the git tag of Task 1's commit.
- [ ] **Step 3:** `build198x basic lint --machine sinclair-zx-spectrum --fix $(find sinclair-zx-spectrum -name '*.bas')`.
- [ ] **Step 4: The safety check.** For each file, tokenise the new text with the same pre-fix binary. The new bytes must equal the old bytes with some `0x20` bytes removed, and none of the removed ones may be inside a string or after `REM`. Walk both byte sequences in step, tracking quote and REM state. Any other difference fails the file. Also confirm `build198x basic lint` reports nothing on the whole tree. Expected: every file passes. A failure means `--fix` changed a program, so stop and investigate it rather than committing.
- [ ] **Step 5:** Run `build198x basic lint --machine commodore-c64` on the C64 tree. Expected: no `listing-form` findings. Fix any `var-name-clash` or `line-order` findings by hand, one commit each, explaining the program's behaviour change if there is one.
- [ ] **Step 6:** Commit with an explicit pathspec, push, and open a PR. The body gives the rule (`unit.md`), the counts of files and lines changed, and the safety-check result.

### Task 13: Makefiles for BASIC lessons, and the Emu198x issue

**Files:**
- Modify: `code-samples/generate-makefiles.sh`: add `spectrum_basic_makefile` and `c64_basic_makefile` templates, and update the header comment that says BASIC tracks "are not assembled at all"
- Create: `Makefile` in each `<system>/basic/<module>/unit-NN/` that a lesson page uses

- [ ] **Step 1: Find each lesson's program.** For each `website/src/content/curriculum/{sinclair-zx-spectrum,commodore-64}/basic/<module>/unit-NN.mdx`, take the last `CodeFromFile` `src` ending in `.bas`. That listing is the unit's program. The run strip reads `public/code-samples/<system>/basic/<module>/unit-NN/`, so the Makefile goes in `code-samples/<system>/basic/<module>/unit-NN/` and builds from that listing's path, relative to that folder.
- [ ] **Step 2: Template.** The Spectrum version:

```make
# Build the program this unit's lesson shows, under its own name, so the
# lesson's run strip runs exactly the listing on the page.
BUILD198X ?= build198x

all: sonar.tap

sonar.tap: ../teaching/unit-04/steps/step-03.bas
	$(BUILD198X) basic $< --machine sinclair-zx-spectrum -o $@

clean:
	rm -f sonar.tap

.PHONY: all clean
```

  The C64 version is identical except for `--machine commodore-c64` and a `.prg` output. The generator writes one per unit from the list in Step 1. The output name is the listing's own filename stem, unless a unit already has a Makefile.
- [ ] **Step 3:** `./generate-makefiles.sh --check`, then `./generate-makefiles.sh`. Run `make` in every new folder. Expected: every folder builds.
- [ ] **Step 4:** Commit and open a PR, stacked on Task 12's branch.
- [ ] **Step 5:** File the Emu198x issue. It isn't a PR; per the family rule, Emu198x changes belong to its own session.

```bash
gh issue create -R emu198x/emu198x --title "Use the published format198x BASIC tokenisers" \
  --body "format-sinclair-zx-spectrum-bas and format-commodore-c64-bas graduated to Format198x as format198x-sinclair-zx-spectrum-bas and format198x-commodore-c64-bas (0.1.x). Switch the runtime and web crates to them and remove the local copies. Behaviour change to expect: the Spectrum tokeniser no longer stores the space the ROM prints before a keyword (THEN, TO, AND...). Spec: build198x/docs specs/2026-09-26-basic-verb.md"
```

### Task 14: Website

**Files:**
- Modify: `website/scripts/build-artefacts.sh` (`BUILD198X_VERSION`)
- Modify: the 11 lesson pages with spaced inline BASIC. Find them with ``rg '`[^`]*\b(LET|FOR|IF)\b[^`]* [=<>+-] [^`]*`' src/content/curriculum/sinclair-zx-spectrum``

- [ ] **Step 1:** Bump `BUILD198X_VERSION` to the release from Task 10.
- [ ] **Step 2:** Rewrite each inline snippet in its listed form, running it through `build198x basic lint --fix` as a one-line listing where it has a line number. Expected: `rg` from above returns nothing.
- [ ] **Step 3:** `npm run build` with `CODE_SAMPLES_PATH` pointing at the Task 13 branch. Expected: `check-browser-player --built` reports more lesson strips than the 90 of #526, the BASIC lessons included.
- [ ] **Step 4:** Commit, push, and open a PR. It must merge after #526 and after the code-samples PRs.
