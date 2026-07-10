# Build198x documentation

Documentation for [Build198x](https://github.com/build198x), the 198x family build-tools pipeline for asset conversion, data packing, and media mastering.

Build198x is active. The flagship workspace is `../build198x/`; this repo holds design notes and documentation that are not specific to one crate.

## Scope

- Asset-conversion, packing, and media-mastering tools.
- Native output formats emitted by Build198x tools.
- The program-framing handoff with Asm198x.
- Validation of mastered media by loading or checking output in Emu198x where appropriate.
- The membership test that keeps the tool set coherent.

## Design specs

Feature design notes live in [`specs/`](specs/) — the shape of a tool or verb
before it is built, and the record of decisions once it is. Current:

- [ADF read verbs (`verify`, `info`)](specs/2026-07-10-adf-read-verbs.md)
- [ADF `create` verb (general volume builder)](specs/2026-07-10-adf-create-verb.md)

## Not here

- Binding umbrella scope: [`../../decisions/build198x-build-tools.md`](../../decisions/build198x-build-tools.md).
- Hardware facts: the umbrella `reference/` library.
- Emulator behaviour: `Emu198x/emu198x/`.
