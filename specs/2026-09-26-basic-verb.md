# `basic` verb: BASIC listings to loadable media

**Status:** approved design, not yet built (2026-09-26); amended the same day to add the lister, the lint rules and the canonical pass.
**Scope:** a `build198x basic` subcommand that turns a plain-text BASIC listing
into the file its machine loads, for the ZX Spectrum and the Commodore 64 first.
It is built so that further BASICs can be added one at a time.
**Consumers:** Code198x code-samples Makefiles, and through them the website's
lesson run strips.

## Purpose

Code198x's BASIC lessons have no runnable build. About 170 Spectrum BASIC lesson
pages show a listing but give the reader nothing to run. The assembly lessons
build in their code-samples Makefiles (`asm198x --tapbas`), and the website runs
whatever those Makefiles produce. No command-line tool does the same for BASIC:

- asm198x only wraps machine code in a BASIC loader;
- Emu198x's tokenisers are private library crates, reachable only from inside
  the emulator and its browser package;
- the retired `zmakebas`/`bas2tap` path is outside the family.

`build198x basic` closes that gap. A code-samples Makefile calls it the same way
the assembly Makefiles call asm198x.

## Command surface

```
build198x basic <in.bas> --machine <id> -o <out> [options]
  --machine <id>     sinclair-zx-spectrum | commodore-c64 (the ids mediaspec198x uses)
  -o <out>           output file: .tap for the Spectrum, .prg for the C64
  --name <name>      tape header name (Spectrum; default: the -o filename stem,
                     cut to 10 characters, as asm198x derives its headers)
  --no-autorun       write a tape that loads without running (Spectrum; default:
                     auto-run from the program's first line)
  --format <text|json>  report format (default text)

build198x basic lint <in.bas>... --machine <id> [--fix] [--format <text|json>]
```

- **One machine, one BASIC.** `--machine` picks the machine; the BASIC and the
  output container follow from it. A machine that has several BASICs, such as
  the 128K Spectrum or the BBC Micro's BASIC versions, will add a `--dialect`
  override when it is added. Nothing in this version needs one, so none is built.
- **Output extension is checked** against the machine: `.tap` for the Spectrum,
  `.prg` for the C64. A mismatch is an error, not a silent rename.
- **Errors name the source line** (`in.bas:12: …`), using the line information
  the tokenisers already report, and exit non-zero without writing the output.
- **Building lints first.** `basic` runs every lint rule before it writes
  anything, so a Makefile cannot build a listing that fails lint.
- **The JSON report** follows the `image` verb's shape: tool version, machine,
  input, output, program length, line count and autorun line.

## The listed form and lint

The source file is written exactly as the machine's LIST displays it, line by
line; screen wrapping is ignored. This is the binding rule in Code198x
`docs/specifications/unit.md`. The reader sees the same program on the page and
in the emulator.

Each dialect crate gains a lister, `list()`, that renders tokenised bytes the way
the machine's LIST does. For the Spectrum that is the ROM's token printing
(`PO-TOKENS`/`PO-SEARCH` in Logan and O'Hara's disassembly):

- line numbers right-aligned in four columns;
- a leading space before tokens from `OR` onwards whose first character is a
  letter, unless the previous character printed was a space;
- a trailing space after tokens ending in a letter or `$`, except `RND`,
  `INKEY$` and `PI`.

The C64 prints the stored characters with tokens expanded and adds nothing.

`build198x basic lint` reports `file:line:column: rule: message` and exits
non-zero on any finding. The rules:

| Rule | Machine | Catches |
|---|---|---|
| `listing-form` | both | a line that differs from its listed form; `--fix` rewrites it |
| `stored-space` | Spectrum | a space outside strings and `REM` that the ROM stores and lists, such as `LET n = n + 1`; `--fix` removes it. A space inside a numeric variable name (`my score`), which the ROM allows and ignores, is left alone |
| `string-var-name` | Spectrum | string variables longer than one letter (`name$`), which the ROM rejects |
| `keyword-var-name` | Spectrum | a variable named like a keyword (`ink`), which the tokeniser can turn into a token |
| `keyword-in-name` | C64 | a keyword with name letters on both sides (`SCORE` holds `OR`), which BASIC V2 turns into a token, so it is not one variable. One-sided contact (`FORI`, `PRINTA`) is ordinary unspaced BASIC V2 and is not flagged |
| `var-name-clash` | C64 | variables that share their first two characters and type (`SCORE`, `SCALE`), which BASIC V2 treats as one |
| `line-order` | both | duplicate or descending line numbers |

The lint rules run on a positioned token stream: the tokeniser's pieces, each
with its kind and its source column. That stream is the foundation for the
parser and language server, the planned second sub-project. See *Not in this
design*.

## Structure

Each machine is two pieces, joined by the verb:

| Piece | Job | Spectrum | C64 |
|---|---|---|---|
| Dialect | Listing text to tokenised program bytes | `format198x-sinclair-zx-spectrum-bas` | `format198x-commodore-c64-bas` |
| Container | Program bytes to the machine's loadable file | TAP header + data block via `format198x-sinclair-zx-spectrum-tap` | PRG: two-byte load address `$0801` + program |

Adding a BASIC later means adding one dialect crate and one container. The verb,
its flags, its errors and its report stay the same.

A data-driven keyword-table design was considered and rejected. The dialects
differ beyond their keyword lists: the Spectrum stores a hidden five-byte number
after every numeric literal, and BBC BASIC encodes line-number references its own
way. A table cannot say either.

## Stage 1: the tokenisers graduate to Format198x

Emu198x's `format-sinclair-zx-spectrum-bas` and `format-commodore-c64-bas` move
to Format198x as `format198x-sinclair-zx-spectrum-bas` and
`format198x-commodore-c64-bas`. This is the umbrella rule in
`decisions/formats-graduate-to-their-own-projects.md`: Build198x is a consumer
that is not the producer.

- Each crate then gains `list()`, the positioned token stream, and (Spectrum)
  a fix so that a source space before a keyword is dropped rather than stored.
  The ROM supplies that space when it lists; storing it wastes a byte. These are
  separate commits after the unchanged move.
- The move takes code, tests and history notes, and gives each crate a README.
  They stay dependency-free and are published with release-plz like the other
  Format198x crates.
- Behaviour does not change in the move. The crates' existing tests come with
  them and must pass unchanged.
- Emu198x switches to the published crates in its own session. That is filed as
  an issue on emu198x/emu198x, not done here.

## Stage 2: the verb in Build198x

- **Decision record:** `decisions/demand-gate-basic.md`, following
  `demand-gate-opening.md`. It names the need (the BASIC lessons with no runnable
  build) and fences the scope:
  - **In:** stock-ROM tokenisation; a self-starting Spectrum TAP; a C64 PRG at
    `$0801`.
  - **Out until their own need arrives:** TZX, disk images, 128K-only keywords,
    loading screens for BASIC tapes, and every other machine.
- **Code:** a `basic` module beside `image` and `beeper`, with the machine table
  in one place so adding a machine touches one list.
- **Dependencies:** the two new Format198x crates and the existing TAP crate,
  from crates.io, as Build198x takes its other Format198x crates.

## Stage 3: code-samples builds its BASIC units

- `generate-makefiles.sh` gains Spectrum and C64 BASIC templates that call
  `build198x basic`. Its header comment, which says BASIC tracks "are not
  assembled at all", is updated.
- A BASIC unit gets a Makefile when a published lesson shows its program. The
  lesson's `CodeFromFile` path decides which listing is the unit's program,
  because BASIC units use two folder layouts (`unit-NN/` and
  `teaching/unit-NN/`). The plan settles the list.
- Before any Makefile lands, a one-off pass rewrites every Spectrum listing to
  its listed form with `lint --fix`. Its safety check: each file's tokenised
  bytes before and after may differ only by removed spaces outside strings,
  `REM` and `DATA`. So no program's behaviour changes. The C64 listings are
  checked the same way; they are expected to need nothing. Lesson prose that
  quotes BASIC inline is corrected by hand to match.
- Outputs are gitignored, as `*.tap` and `*.prg` already are. The 99 Spectrum
  tapes committed under `verification/` folders are evidence and are not
  touched.
- The website's `build-artefacts.sh` already runs every Makefile and stages the
  results, so lesson run strips appear once website #526 is merged. CI needs
  `build198x` installed alongside asm198x.

## Verification

Real programs, not invented ones:

- **C64:** every one of the 86 code-samples C64 listings is tokenised by
  `build198x basic` and by VICE's `petcat`, and the PRGs must match byte for
  byte. `petcat` is an independent implementation, so a mismatch is evidence.
  A listing that petcat reads differently (a known dialect quirk) is recorded
  with the reason, not silently skipped.
- **Spectrum lister against the ROM:** fixture programs cover every token in
  leading- and trailing-space positions, plus a sample of real lesson lines. Each
  is loaded into Emu198x with the genuine 48K ROM, `LIST` is run, and the screen
  text is kept as the expected output. The lister's tests compare against those
  captures, so the ROM, not our reading of it, is the authority.
- **Spectrum:** there is no independent tokeniser installed. A handful of real
  lesson tapes (at least one per BASIC game) are loaded in the Emu198x headless
  runner and must reach their first screen, checked by reading the screen text.
  The existing tests of the tokeniser crate carry over from Emu198x.
- **Round trip:** a TAP written by the verb decodes with the TAP crate to a
  program header naming the right length and autorun line.
- **Errors:** a listing with a bad line number or an unknown character fails
  with the source line named and writes nothing.

## Not in this design

- **The parser and language server.** That's the second sub-project, with its
  own spec. It covers a lossless syntax tree per dialect, and LSP diagnostics,
  hover, completion, jumping to line numbers and formatting. The same
  diagnostics would go to the website's in-page BASIC editors as wasm. Where it
  lives is for that spec to decide.

- Detokenising (`.tap`/`.prg` back to text).
- Other machines: VIC-20, BBC Micro, Electron, Atom and MSX have `basic/` folders
  in code-samples but no listings yet. Each is added when it has lessons.
- Changing the website's player or run strip: they already run TAP and PRG files.
