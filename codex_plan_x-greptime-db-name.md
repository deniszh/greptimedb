# Plan: Make `x-greptime-db-name` Configurable

## Goal
Add a server config option to make the HTTP database-selection header name configurable, while preserving the current default behavior.

Proposed config:

```toml
[http]
db_name_header_name = "x-greptime-db-name"
```

## Scope
This change should apply to HTTP request paths that build a `QueryContext` from request headers, including Prometheus remote write and OTLP ingestion. The default value must remain `x-greptime-db-name` for backward compatibility.

## Implementation Steps
1. Extend [`HttpOptions`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/src/servers/src/http.rs#L151) with a new field:
   - `db_name_header_name: String`
   - default value: `"x-greptime-db-name"`

2. Validate the configured header name during HTTP server setup:
   - parse it once with `http::HeaderName::from_bytes(...)`
   - fail fast on invalid configuration instead of deferring to request time

3. Thread the configured header name into the HTTP auth/context middleware:
   - extend [`AuthState`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/src/servers/src/http/authorize.rs#L41) so it carries the configured header name in addition to `user_provider`
   - update [`check_http_auth`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/src/servers/src/http/authorize.rs#L111) and [`inner_auth`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/src/servers/src/http/authorize.rs#L53) to use that state

4. Refactor database extraction so it no longer relies on the hard-coded `GreptimeDbName::name()` lookup:
   - change [`extract_catalog_and_schema`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/src/servers/src/http/authorize.rs#L133) to read the configured header name first
   - keep `?db=...` as fallback
   - keep the default constant in [`header.rs`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/src/servers/src/http/header.rs#L40) as the canonical default value source

5. Update configuration surfaces:
   - [`config/frontend.example.toml`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/config/frontend.example.toml)
   - [`config/standalone.example.toml`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/config/standalone.example.toml)
   - regenerate [`config/config.md`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/config/config.md) with `make config-docs`
   - update config loading expectations in [`load_config_test.rs`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/src/cmd/tests/load_config_test.rs#L105)

## Tests
Add focused coverage for:

- `extract_catalog_and_schema` with a custom configured header name
- middleware/auth flow proving the custom header populates the request `QueryContext`
- precedence rules: configured header should still win over `?db=...`
- one HTTP ingestion test, ideally Prometheus remote write, using non-default `HttpOptions`

Relevant existing tests:

- [`src/servers/tests/http/authorize.rs`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/src/servers/tests/http/authorize.rs)
- [`src/servers/tests/http/prom_store_test.rs`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/src/servers/tests/http/prom_store_test.rs)

## Compatibility Notes
- Default behavior must stay unchanged.
- HTTP header names remain case-insensitive at runtime.
- `?db=...` remains supported as fallback.

## Open Question
This plan targets HTTP only. gRPC currently also uses the same hard-coded header constant in [`context_auth.rs`](/Users/dzhdanov/Documents/github/deniszh/greptimedb/src/servers/src/grpc/context_auth.rs#L35). Unless there is a separate requirement, keep this feature HTTP-scoped and avoid changing gRPC behavior in the first iteration.
