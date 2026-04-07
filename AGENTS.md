# Repository Guidelines

## Project Structure & Module Organization
GreptimeDB is a Rust workspace rooted at [`Cargo.toml`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/Cargo.toml). Core crates live under `src/`: `frontend`, `datanode`, `meta-srv`, `standalone`, plus shared libraries in `src/common/*`. Integration fixtures and storage-backend tests live in `tests-integration/`, SQL regression cases live in `tests/cases/`, and fuzz targets live in `tests-fuzz/`. Operational assets sit in `config/`, `docker/`, `grafana/`, and `docs/`.

## Build, Test, and Development Commands
Use the Makefile as the default entry point:

- `make build`: build the default `greptime` binary with locked dependencies.
- `cargo run -- standalone start`: run a local standalone node after building.
- `make test`: run workspace tests through `cargo nextest` with repo-standard features.
- `make sqlness-test`: execute SQL regression tests from `tests/cases`.
- `make clippy`: fail on any Clippy warning across the workspace.
- `make fmt`, `make fmt-check`, `make check-toml`: format and verify Rust and TOML files.
- `make check-udeps`: detect unused dependencies before opening a PR.

## Coding Style & Naming Conventions
Follow the Rust style guide in `docs/style-guide.md`. Keep `mod` declarations before `use`, prefer same-name module files over `mod.rs`, and add doc comments for public items. Format with `cargo fmt --all`; `rustfmt.toml` enforces `imports_granularity = "Module"` and `group_imports = "StdExternalCrate"`. Use snake_case for modules, files, and functions; UpperCamelCase for types; SCREAMING_SNAKE_CASE for constants.

## Testing Guidelines
Run `make test` before submitting changes. For SQL behavior, add `.sql` cases under `tests/cases/...` and review the paired `.result` files after `make sqlness-test`. Storage or Kafka integration tests may require a root `.env`; see `tests-integration/README.md` for S3, OSS, AzBlob, Kafka, and TLS setup. Bench and fuzz work belong in crate-level `benches/` and `tests-fuzz/`.

## Commit & Pull Request Guidelines
Commits and PR titles should follow Conventional Commits, matching recent history such as `fix: windows ci` and `refactor(metric-engine): ...`. Keep commit subjects imperative and scoped when helpful. PRs should explain motivation, call out breaking or API changes, link issues, and confirm local checks passed. If configuration files under `config/` change, run `make config-docs` and include the generated docs in the same PR.
