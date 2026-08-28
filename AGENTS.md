# Build198x docs

> Read [`PRINCIPLES.md`](PRINCIPLES.md) first. [`MANIFESTO.md`](MANIFESTO.md) is why the project exists.

Documentation repo for Build198x. Part of the `build198x` org container; see [`../AGENTS.md`](../AGENTS.md) for org layout and [`../../AGENTS.md`](../../AGENTS.md) for the 198x umbrella.

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
