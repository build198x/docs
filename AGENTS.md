# Build198x docs

Documentation repo for Build198x. Part of the `build198x` org container; see [`../CLAUDE.md`](../CLAUDE.md) for org layout and [`../../CLAUDE.md`](../../CLAUDE.md) for the 198x umbrella.

Build198x is active. The flagship workspace is `../build198x/`; this repo holds design notes and documentation that are not specific to one crate.

## Working rules

- Put Build198x-wide docs here.
- Put implementation-specific decisions and docs in `../build198x/` when they belong to the flagship workspace.
- Put hardware facts in the umbrella `reference/` layer first.
- Link cross-project scope decisions rather than restating their history.

## Not here

- Binding umbrella scope: [`../../decisions/build198x-build-tools.md`](../../decisions/build198x-build-tools.md).
- The inherited output-format model: [`../../Asm198x/asm198x/decisions/assemble-io-model.md`](../../Asm198x/asm198x/decisions/assemble-io-model.md).
- Hardware facts: [`../../reference/`](../../reference/).
