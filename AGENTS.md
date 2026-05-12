# AGENTS.md — ntfy

Agent guidance for working in this repository.

---

## Project Overview

**ntfy** is a pub/sub notification service written in Go with a React web app.
The server exposes an HTTP API; clients publish and subscribe to topics.
The codebase supports SQLite (default) and PostgreSQL backends, Firebase Cloud
Messaging, Web Push, SMTP ingress, Stripe payments, and Prometheus metrics.

Module path: `heckel.io/ntfy/v2`

---

## Repository Layout

```
cmd/            CLI commands (serve, publish, subscribe, user, access, tier, token, webpush)
server/         HTTP server, config, middleware, account/admin/payments/matrix handlers
db/             SQLite and PostgreSQL database managers + schema
user/           User/tier/token management
model/          Shared domain types
message/        Message parsing and formatting
client/         Go client library
web/            React web app (Vite + MUI)
docs/           MkDocs documentation source
log/            Structured logger
util/           Shared utilities
attachment/     Attachment storage (local + S3)
mail/           SMTP sender
payments/       Stripe integration
webpush/        Web Push (RFC 8030) integration
s3/             S3 attachment backend
test/           Integration test helpers
```

---

## Build System

All common tasks are driven by `make`. Key targets:

| Target | Purpose |
|---|---|
| `make build` | Full build: web app + docs + CLI binaries |
| `make cli-linux-server` | Build server binary for current arch (no GoReleaser) |
| `make cli-client` | Build client-only binary (CGO_ENABLED=0) |
| `make web` | Build React web app into `server/site/` |
| `make docs` | Build MkDocs documentation into `server/docs/` |
| `make test` | Run Go tests (excludes `test/`, `examples/`, `tools/`) |
| `make race` | Run tests with `-race` |
| `make coverage` | Run tests with coverage report |
| `make check` | Tests + fmt-check + web-fmt-check + vet + web-lint + lint + staticcheck |
| `make fmt` | Format Go and web sources |
| `make vet` | `go vet ./...` |
| `make lint` | `golint` |
| `make staticcheck` | `staticcheck ./...` |

The server binary requires `CGO_ENABLED=1` (SQLite). The client-only binary
uses `CGO_ENABLED=0` with `-tags noserver`.

### Build Tags

| Tag | Effect |
|---|---|
| `noserver` | Client-only build; excludes all server code |
| `nofirebase` | Replaces Firebase sender with a no-op stub (`server_firebase_dummy.go`) |
| `nopayments` | Replaces Stripe integration with a no-op stub (`server_payments_dummy.go`) |
| `nowebpush` | Replaces Web Push with a no-op stub (`server_webpush_dummy.go`) |
| `sqlite_omit_load_extension` | Always set for server builds; disables SQLite extension loading |
| `osusergo,netgo` | Static linking; used in production Linux builds |

---

## Development Setup

### Prerequisites

- Go 1.25+
- Node.js 24+ with npm
- Python 3 + pip (for docs)
- `gcc` (for CGO/SQLite)
- Cross-compilers for ARM/Windows targets (optional)

> ⚠️ The `devcontainer.json` comment suggests `mcr.microsoft.com/devcontainers/go:1.24`
> but the project requires Go 1.25+. Use a Go 1.25 image or the universal image.

### Stub directories (required before any build or test)

> **`server/server.go` embeds `server/docs/` and `server/site/` at compile time.**
> If either directory is missing the build fails with an embed error. Always run
> this once after a fresh checkout before `go build`, `go test`, or `make test`:
>
> ```bash
> make cli-deps-static-sites
> ```
>
> This creates empty stub files so the embed directives resolve. The full
> `make web` and `make docs` targets replace the stubs with real content.

### Quick start (server binary only)

```bash
make cli-deps-static-sites   # create stub server/docs and server/site
make cli-linux-server        # build binary to dist/ntfy_linux_server/ntfy
```

### Quick start (full build)

```bash
make build-deps-ubuntu       # install system deps (Ubuntu/Debian)
make build                   # web + docs + CLI
```

### Running tests

Tests require `server/docs/` and `server/site/` to exist (stubs are fine):

```bash
make cli-deps-static-sites
make test
```

PostgreSQL integration tests are enabled when `NTFY_TEST_DATABASE_URL` is set:

```bash
export NTFY_TEST_DATABASE_URL="postgres://ntfy:ntfy@localhost:5432/ntfy_test?sslmode=disable"
make test
```

S3 tests are enabled when `NTFY_TEST_S3_URL` is set.

### Local Dev Workflow

The `.gitpod.yml` defines the intended live-reload setup:

| Process | Command | Port |
|---|---|---|
| Server (live reload) | `nodemon --watch './**/*.go' --ext go --exec "CGO_ENABLED=1 go run main.go serve ..."` | 2586 |
| Web app | `cd web && npm start` | 3000 |
| Docs | `mkdocs serve` | 8000 |

For a minimal server-only loop without nodemon:

```bash
make cli-deps-static-sites
CGO_ENABLED=1 go run main.go serve --listen-http :2586 --debug
```

---

## Testing

- Tests live alongside source files (`*_test.go`).
- Integration helpers are in `test/`.
- Run `make check` before opening a PR — it runs tests, formatting, vetting,
  linting, and staticcheck.
- The CI workflow (`.github/workflows/test.yaml`) runs `make checkv` and
  `make coverage` against Go 1.25 with a live PostgreSQL 17 instance.

### Writing Tests

Use the helpers defined at the bottom of `server/server_test.go`:

```go
config := newTestConfig(t, databaseURL)   // creates a temp-dir config (SQLite or Postgres)
s := newTestServer(t, config)             // starts a test server; cleanup is registered automatically
```

Pass an empty string for `databaseURL` to use an in-memory SQLite database.

For tests that require an isolated PostgreSQL schema, use the helper from
`heckel.io/ntfy/v2/db/test`:

```go
databaseURL := dbtest.CreateTestPostgresSchema(t)  // skips if NTFY_TEST_DATABASE_URL is unset
```

Use `require` (not `assert`) from `github.com/stretchr/testify/require` — the
project uses `require` exclusively in tests.

---

## Where New Code Belongs

| What | Where |
|---|---|
| New HTTP endpoint | `server/server_<feature>.go`; register the handler in `server/server.go` |
| New CLI subcommand | New file in `cmd/`; register in `cmd/app.go` |
| Shared domain types | `model/` |
| Shared utilities | `util/` |
| Database queries (SQLite) | `db/db.go` |
| Database queries (PostgreSQL) | `db/pg/pg.go` |
| User / auth / tier logic | `user/manager.go` |
| Schema changes | Embedded SQL strings in the relevant manager file; bump the schema version constant |

---

## Code Conventions

### Go

- Standard `gofmt` formatting. Run `make fmt-check` to verify.
- Errors are returned, not panicked. Use the project's `log` package for
  structured logging.
- Config is loaded from `server/server.yml` (YAML) and overridden by CLI flags
  or environment variables (prefix `NTFY_`).
- Database access goes through the interfaces in `db/` — do not call SQLite or
  PostgreSQL drivers directly from `server/` or `cmd/`.
- New server endpoints belong in `server/server.go` or a dedicated
  `server/server_<feature>.go` file following the existing pattern.

### Web App

- Framework: React + Vite
- UI library: MUI (Material UI v5)
- i18n: `i18next`; translation files are in `web/src/`
- Entry point: `web/src/index.jsx`
- Built output: `server/site/` — gitignored, do not commit
- Format with Prettier (`make web-fmt`), lint with ESLint (`make web-lint`)
- Run `make web-fmt-check` and `make web-lint` before committing (both run as part of `make check`)

### Commit Messages

No enforced format. Use a short, descriptive subject line that states what changed
and why. Avoid generic messages like "fix" or "update". The repository history
contains many Weblate translation commits and Dependabot bumps — do not use those
as style guides for hand-written commits.

---

## Pre-PR Checklist

```bash
make cli-deps-static-sites   # ensure embed stubs exist
make check                   # tests + fmt + web-fmt-check + vet + lint + staticcheck + web checks
```

With PostgreSQL coverage:

```bash
export NTFY_TEST_DATABASE_URL="postgres://ntfy:ntfy@localhost:5432/ntfy_test?sslmode=disable"
make check
```

CI runs the same suite on every push and PR. A PR that fails `make check` locally
will fail CI.

---

## CI / GitHub Actions

| Workflow | Trigger | What it does |
|---|---|---|
| `test.yaml` | push/PR to `main` | `make checkv` + `make coverage` (with Postgres) |
| `build.yaml` | push/PR to `main` | `make build` (full build) |
| `docs.yaml` | push/PR to `main` | Builds and deploys documentation |
| `release.yaml` | tag push | GoReleaser release |

---

## Security

Report vulnerabilities privately to [security@mail.ntfy.sh](mailto:security@mail.ntfy.sh).
Do not open public issues for security bugs. See `SECURITY.md`.

---

## Key Environment Variables

| Variable | Purpose |
|---|---|
| `NTFY_CONFIG_FILE` | Path to server config file |
| `NTFY_BASE_URL` | Externally visible base URL |
| `NTFY_LISTEN_HTTP` | HTTP listen address (default `:80`) |
| `NTFY_TEST_DATABASE_URL` | PostgreSQL DSN for integration tests |
| `NTFY_TEST_S3_URL` | S3 URL for attachment tests |

All server config keys can be set as `NTFY_<KEY>` environment variables.
