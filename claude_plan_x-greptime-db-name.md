# Plan: Make `x-greptime-db-name` HTTP header name configurable

## Context

GreptimeDB uses a hardcoded header `x-greptime-db-name` for clients to specify the target database on HTTP requests (Prometheus remote write, OTLP, InfluxDB, SQL, etc.). Some deployment environments need a different header name (e.g., behind a reverse proxy that rewrites or reserves certain headers). This change adds an `http.db_name_header` config option, defaulting to the current `x-greptime-db-name` to maintain backward compatibility. Scope: HTTP path only (gRPC keeps the hardcoded name).

## Approach

Thread the configured header name through `HttpOptions` → `AuthState` → `extract_catalog_and_schema()`.

### Step 1: Add config field to `HttpOptions`

**File:** `src/servers/src/http.rs` (lines 149–182)

Add a new field to `HttpOptions`:
```rust
pub struct HttpOptions {
    // ... existing fields ...
    
    /// Custom HTTP header name for specifying the database name.
    /// Defaults to "x-greptime-db-name".
    pub db_name_header: String,
}
```

Update `Default` impl (line 170) to set `db_name_header: constants::GREPTIME_DB_HEADER_NAME.to_string()`.

### Step 2: Extend `AuthState` to carry the header name

**File:** `src/servers/src/http/authorize.rs` (lines 44–54)

Add a `db_name_header` field to `AuthState`:
```rust
#[derive(Clone)]
pub struct AuthState {
    user_provider: Option<UserProviderRef>,
    db_name_header: HeaderName,
}
```

Update `AuthState::new()` to accept and store the `HeaderName`.

### Step 3: Pass header name through extraction

**File:** `src/servers/src/http/authorize.rs` (lines 133–151)

Change `extract_catalog_and_schema` to accept a `&HeaderName` parameter instead of using `GreptimeDbName::name()`:
```rust
pub fn extract_catalog_and_schema<B>(request: &Request<B>, db_name_header: &HeaderName) -> (String, String) {
    let dbname = request
        .headers()
        .get(db_name_header)  // was: GreptimeDbName::name()
        ...
}
```

Update the call in `inner_auth` (line 56) — modify `inner_auth` signature to accept `db_name_header: &HeaderName`, and update `check_http_auth` (line 118) to extract it from `AuthState` and pass it through.

### Step 4: Wire config into `AuthState` at server build time

**File:** `src/servers/src/http.rs` (line 884)

Change the `AuthState::new(...)` call in `HttpServer::build()` to also pass the configured header name:
```rust
AuthState::new(
    self.user_provider.clone(),
    HeaderName::from_str(&self.options.db_name_header).unwrap_or(GREPTIME_DB_HEADER_NAME.clone()),
)
```

Use `HeaderName::from_str` with a fallback to the default constant in case the config value is invalid.

### Step 5: Update `check_http_auth` to forward the header name

**File:** `src/servers/src/http/authorize.rs` (line 118–127)

```rust
pub async fn check_http_auth(
    State(auth_state): State<AuthState>,
    req: Request<Body>,
    next: Next,
) -> Response {
    match inner_auth(auth_state.user_provider, &auth_state.db_name_header, req).await {
        ...
    }
}
```

## Files to modify

1. `src/servers/src/http.rs` — Add `db_name_header` to `HttpOptions`, wire into `AuthState` at line 884
2. `src/servers/src/http/authorize.rs` — Extend `AuthState`, update `inner_auth` and `extract_catalog_and_schema` signatures
3. `src/servers/src/http/header.rs` — No changes needed (keep constants for default value and gRPC path)

## What stays unchanged

- gRPC header extraction (`src/servers/src/grpc/context_auth.rs`) — keeps using the hardcoded constant
- gRPC client injection (`src/client/src/database.rs`) — keeps using the hardcoded literal
- The `GreptimeDbName` typed header struct — stays for backward compat, but `extract_catalog_and_schema` will use the raw `HeaderName` instead
- All test files — they'll continue to use `GREPTIME_DB_HEADER_NAME` constant which remains the default

## Verification

1. `cargo check -p servers` — compiles
2. `cargo nextest run -p servers` — unit tests pass (existing tests use default header, so they should pass without changes)
3. `cargo nextest run -p tests-integration` — integration tests pass
4. `make clippy` — no warnings
5. Manual test: configure `db_name_header = "x-custom-db"` in standalone config, send a request with that header, verify it routes to the correct database
