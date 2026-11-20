# Trailboss specification

The formats other programs rely on when they work with Trailboss: every command's `--json` output, and the local server that answers it over HTTP. Other interfaces get their own folder next to `json/`.

- [VERSIONING.md](VERSIONING.md): what can change, what can't, and how a frontend checks the CLI it found.
- [json/](json/): a JSON Schema per command (`<command>.schema.json`) and examples made from synthetic data (`examples/`).
- [serve/](serve/): the protocol of `trailboss serve`, the local server whose endpoints answer what the commands print with `--json`.

This repository is the source of truth: the CLI's tests check its output against these schemas. Report problems in [trailboss-ai/trailboss](https://github.com/trailboss-ai/trailboss/issues).

License: MIT ([LICENSE](LICENSE)).
