# Versioning

How the CLI's machine-readable output changes, so a frontend (the Herdr mod, a tmux plugin, your own script) knows what it can rely on.

## What a frontend checks

`trailboss --json version` returns:

```json
{ "version": "0.4.0", "api": 1 }
```

- **`version`** is the CLI's release. A frontend that needs a field added in some release says so as a minimum version ("needs Trailboss ≥ 0.4.0") and checks it.
- **`api`** is the version of the `--json` output as a whole. It goes up only for an incompatible change (below). A frontend states the `api` it was written for and treats a higher one as "this frontend needs updating".
- A CLI older than the `version` command has no `api`: treat it as too old.

## What can change without a new `api`

- **New fields** in an object, and **new commands**. Frontends must ignore fields they don't know. Schemas never forbid extra fields for this reason.
- **New values** where a field holds one of a set of strings (a `status`, an event `kind`, a resume `action`). Frontends must handle a value they don't know, usually by showing it as is.
- **A field becoming nullable** that wasn't before, only when it's documented in the CHANGELOG. (Prefer adding a new field.)

## What needs a new `api`

- Renaming or removing a field or a command.
- Changing a field's type or meaning (a cost in dollars becoming cents, an array becoming an object).
- Changing the top-level shape of a command's output (an array becoming an object).

When `api` goes up, the CHANGELOG lists every incompatible change under **Changed** or **Removed**, and the previous behaviour stays available for at least one minor release where that's possible.

## Where it's written down

- `json/<command>.schema.json`: a [JSON Schema](https://json-schema.org/) (draft 2020-12) of each command's `--json` output. `review AGENT:ID` and `review AGENT:ID --diff PATH` have their own (`review-session`, `review-diff`).
- `json/examples/<name>.json`: an example of each, made from synthetic data. `<name>` is the schema's name, optionally followed by `-<variant>` (`usage-by-day` is an example of `usage`).
- `tests/specification.rs` checks every example against its schema, and the CLI's real output on the test fixtures against the schemas, on every test run.

## Changing the output

In the same commit as the change to the CLI:

1. Update the command's schema (a new property, a new nullable type).
2. Update its example: `python3 scripts/specification_examples.py` regenerates the ones made from fixtures (after `cargo build --release`); edit the hand-written ones (`live`, `events`, `resume`, `quota`, `stats`, `review*`) by hand.
3. Add a CHANGELOG line. If it's incompatible, raise `API` in `src/main.rs` and follow the rules above.
