# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Dev Commands

GreptimeDB uses a **nightly Rust toolchain** (pinned in `rust-toolchain.toml`) and requires `protoc >= 3.15`.

```bash
# Build
make build                          # debug build
cargo run -- standalone start       # build + run standalone

# Test (uses cargo-nextest, NOT cargo test)
make test                           # all tests with pg/mysql kvbackend features
cargo nextest run -p <crate>        # single crate
cargo nextest run <test_name>       # single test by name
cargo sqlness bare                  # SQL regression tests (tests/cases/)

# Lint & Format
make fmt-check                      # cargo fmt + snafu/super-imports checks
make clippy                         # clippy with -D warnings
make check-udeps                    # unused dependency check

# Integration tests (need Docker services)
cd tests-integration/fixtures && docker compose up -d --wait
cargo nextest run -p tests-integration
```

PR checklist order: format → nextest → clippy → check-udeps.

## Architecture

Single binary (`greptime`) with five roles: **standalone**, **frontend**, **datanode**, **metasrv**, **flownode**. All implement the `App` trait in `src/cmd/`.

```
Protocols (MySQL/PG/gRPC/HTTP/PromRW/OTLP/InfluxDB/Loki/ES)
    → src/servers (protocol decode)
    → src/frontend + src/operator (query routing, DML/DDL)
    → src/query (DataFusion-based, forked v52.1) + src/promql
    → src/datanode (RegionServer)
    → Storage engines: src/mito2 (primary LSM-tree TSDB) | src/metric-engine (Prometheus-style multiplexing over Mito)
    → WAL: raft-engine (local) or Kafka (remote) via src/log-store
    → SSTs: Parquet on object store (S3/GCS/local) via src/object-store
```

**Key crates:**
- `src/mito2` — Primary storage engine. WorkerGroup serializes writes per region: WAL → memtable → flush to Parquet SSTs → compaction. Indexes via `src/index` + `src/puffin`.
- `src/metric-engine` — Multiplexes many logical regions onto two physical Mito regions (data + metadata) for high-cardinality Prometheus workloads.
- `src/meta-srv` — Cluster coordinator: etcd-backed leader election, region placement, DDL procedure orchestration, region migration.
- `src/flow` — Streaming/continuous dataflow compute (substrait plans).
- `src/common/procedure` — Persistent distributed procedure framework for DDL operations.
- `src/store-api` — `RegionEngine` trait contract between datanode and engines.
- `src/catalog` — `CatalogManager` + `information_schema`/`pg_catalog` virtual tables.
- `src/pipeline` — VRL-based log processing transforms at ingest time.

## Error Handling Pattern

Every crate has its own `error.rs` using **snafu** + `#[stack_trace_debug]` (from `common_macro`). This is the universal pattern:

```rust
#[derive(Snafu)]
#[snafu(visibility(pub))]
#[stack_trace_debug]
pub enum Error {
    #[snafu(display("Failed to do X: {detail}"))]
    SomeVariant {
        detail: String,
        #[snafu(implicit)]
        location: Location,
    },
    // Internal GreptimeDB errors: field MUST be named `source`
    #[snafu(display("Wrapped"))]
    Internal { source: other::Error, #[snafu(implicit)] location: Location },
    // External/third-party errors: field MUST be named `error` with #[snafu(source)]
    #[snafu(display("External"))]
    External { #[snafu(source)] error: ext::Error, #[snafu(implicit)] location: Location },
}
```

Every `Error` enum must implement `ErrorExt` mapping variants to `StatusCode`. Use `with_context()` over `context()` when constructing error strings requires allocation.

## Code Conventions

- **Module files**: use `foo.rs` + `foo/bar.rs`, NOT `foo/mod.rs`
- **Mod before use**: place all `mod` declarations before `use` statements
- **Doc comments**: all public items need `///` with `[]` links to referenced types
- **Logging**: use `common_telemetry` macros (`error!(e; "msg")`, `warn!`, `info!`), not `tracing` directly
- **Unimplemented**: use `unimplemented!()`, not `todo!()`
- **Clippy**: `print_stdout`, `print_stderr`, `dbg_macro` are warned; `unknown_lints` is denied

## Testing

- **Unit tests**: `#[cfg(test)]` modules, run via `cargo nextest run`
- **SQL regression** (`tests/cases/`): `.sql` + `.result` pairs under `standalone/` and `distributed/`. Re-run `cargo sqlness bare` to update `.result` files after intentional changes.
- **Integration tests** (`tests-integration/`): spin up real cluster instances via `GreptimeDbClusterBuilder`; require Docker services (etcd, Kafka, Postgres, etc.)
- **Fuzz tests** (`tests-fuzz/`): `cargo fuzz run <target> --fuzz-dir tests-fuzz`

## Config

Example configs in `config/` (standalone, datanode, frontend, metasrv, flownode). After modifying config structs, run `make config-docs` to regenerate docs.
