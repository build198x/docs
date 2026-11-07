# `build198x tape` Implementation Plan

**Goal:** Master a ZX Spectrum distribution tape (an auto-running BASIC loader, a SCREEN$ loading screen and the program's CODE) in one deterministic step. Today the curriculum builds that tape by joining files with `cat`.

**Issue:** build198x/build198x#60. **Decision:** `build198x/decisions/demand-gate-tape-master.md`, amended in Task 6. **Umbrella:** `decisions/tape-framing-vs-mastering.md` (Build198x owns composed tapes).

**Agreed design (2026-11-07):**

- By default the tool **generates** the loader, so it can check the load address, entry point and RAM limits itself. `--basic <loader.bas>` replaces it with an authored loader, which is checked by lint only. The tool never rewrites an authored loader.
- `--machine` is required and names the system, as `basic --machine` does. Tape formats share an extension but nothing else (a C64 `.tap` is a pulse recording), so the code lives in `build198x::tape::sinclair_zx_spectrum`, and other machines join as siblings.
- The worked consumer is **Meteor Storm** (Code198x, Spectrum assembly module 2), replacing Gloaming, which is being removed. Its `unit-36/Makefile` already joins loader, screen and game with `cat`, so it doubles as a byte-for-byte oracle.

## CLI

```
build198x tape --machine sinclair-zx-spectrum --code <code.bin> --load-address <decimal> -o <out.tap>
               [--entry <decimal>]          # default: the load address
               [--screen <screen.scr>]      # default: a built-in test card
               [--basic <loader.bas>]       # replaces the generated loader
               [--model 48k|16k]            # default 48k; sets the top of RAM
               [--border <0-7>] [--paper <0-7>] [--ink <0-7>]   # generated loader only; default 0
               [--name <name>]              # loader block; default: output stem
               [--code-name <name>] [--screen-name <name>]      # default: --name, "screen"
               [--format text|json] [--force]
```

These follow the `plus3` precedent: atomic output, an existing destination refused without `--force`, refusal when the output aliases an input, and nothing written on any error. Exit codes match `plus3`: 2 usage, 3 unreadable input, 4 invalid content, 5 output.

## Tape layout

Three header and data pairs, in this order, each built with `format198x-sinclair-zx-spectrum-tap`:

1. `Program` header (name, autorun = first stored line, param2 = program length), then the tokenised loader (`format198x-sinclair-zx-spectrum-bas`).
2. `Code` header (screen name, start 16384, length 6912), then the 6912 screen bytes.
3. `Code` header (code name, load address, length), then the code bytes.

## Generated loader

Written in the listed form (each line exactly as the ROM LISTs it) and linted like any other source before tokenising:

```
  10 BORDER b: PAPER p: INK i: CLEAR load-1
  20 LOAD ""SCREEN$ 
  30 POKE 23739,111
  40 LOAD ""CODE 
  50 POKE 23739,244: RANDOMIZE USR entry
```

`POKE 23739,111` points the screen channel's output routine at `$096F`, the `RET` that ends the routine before SA-CONTRL (*The Complete Spectrum ROM Disassembly*). All ROM printing is discarded, so "Bytes: name" does not land on the loading screen when the CODE header arrives. Meteor Storm's authored loader does the same. The generated loader then puts back PRINT-OUT (`$09F4`, `POKE 23739,244`) before `USR`, so a program that prints through `RST $10` still works.

## Validation (generated loader)

- The screen is exactly 6912 bytes; the code is 1 or more bytes.
- `load + len` is at most 65536 (48K) or 32768 (16K).
- The entry lies within `[load, load + len)`.
- The code starts high enough for the loader to `CLEAR` and `LOAD` beneath it. **Measured:** with the 130-byte generated loader, CODE at 24040 loads and runs on the 48K ROM in Emu198x, and at 24039 the ROM stops with "4 Out of memory, 40:1". That is LOAD's free-space check (TEST-ROOM wants 80 spare bytes), not CLEAR's "RAMTOP no good". So the floor is the loader's end plus 155 bytes, pinned by a unit test.

With `--basic`, the screen, extent and model checks still apply. The loader's `CLEAR` and `USR` belong to its author.

## Tasks

Branch `feat/tape-master` in `Build198x/build198x`; one PR.

1. **Library module** `crates/build198x/src/tape/sinclair_zx_spectrum.rs`: `build(options) -> Result<Built, Error>` composes the three pairs, the generated loader and the validation above. Unit tests cover each refusal, the 16K/48K edges, and determinism (two builds are byte-identical).
2. **Load floor**: measure it against the real ROM in Emu198x (a binary search over load addresses), explain it from the ROM disassembly, and pin it in a unit test.
3. **Test card**: a deterministic built-in SCREEN$ (attribute colour bars plus a pixel border), generated in code with no binary asset committed. Its bytes are pinned by a SHA-256 test.
4. **CLI** `build198x tape` in `main.rs` with the flags above, and integration tests in `tests/tape_cli.rs` following `plus3_cli.rs`.
5. **Committed example** under `examples/tape/`: a tiny program that changes the border, mastered with the test card. Its Makefile rebuilds the tape and compares it byte for byte.
6. **Decision record**: amend `demand-gate-tape-master.md`. Gloaming is no longer the consumer, Meteor Storm is, and the loader is generated with an authored override. Steve approves the wording in the PR.
7. **README and help text**: move the tape master from "planned" to shipped.

## Verification

- **Byte identity with the curriculum's tape:** assemble Meteor Storm's `meteor-storm.asm` to a raw binary with asm198x, then run `build198x tape --basic loader.bas --screen meteor-storm.scr --code storm.bin --load-address 32768 --name meteor --code-name storm --screen-name screen`. The output must equal `unit-36/meteor-storm.tap`, which `make` produces with `cat`.
- **Loads through the real ROM tape path:** in `emu198x-spectrum`, `LOAD ""` the generated-loader tape (both the test card and Meteor Storm's screen). Assert that the screen memory holds the SCREEN$ bytes, the code is at the load address, and execution reaches the entry. Do this on the 48K model, and on the 16K model with a 16K-sized example.
- **Independent emulator:** load the same tapes in MAME (`spectrum`, headless). Check the loading screen and that the program runs, with screenshots compared against Emu198x's.
- **A check that can fail:** corrupt one byte of the code and confirm the ROM reports a tape loading error in both emulators.

## Follow-ups (not in this PR)

- Code198x: switch Meteor Storm's `unit-36/Makefile` (and unit 35's) from `cat` to `build198x tape` once a release carries it. That change goes through Code198x's own PR flow.
- TZX, turbo loaders, multiload and 128K paging stay behind their own demand gates (#6, #61).
