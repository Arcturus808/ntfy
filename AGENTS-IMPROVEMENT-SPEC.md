# AGENTS-IMPROVEMENT-SPEC.md

Audit of the newly created `AGENTS.md` and concrete improvements to apply.

---

## What's Good

- **Project overview** — module path, tech stack, and dual-backend (SQLite/PostgreSQL) are documented.
- **Repository layout** — all top-level packages are listed with one-line descriptions.
- **Build system** — key `make` targets are tabulated with purpose; CGO requirements and build tags are noted.
- **Dev setup** — prerequisites, quick-start paths for server-only vs full build, and test environment variables are covered.
- **CI matrix** — all four workflows and their triggers are listed.
- **Security pointer** — directs to `SECURITY.md` and the private reporting address.
- **Key env vars** — the most important runtime and test variables are listed.

---

## What's Missing

### 1. Build tags are incomplete

`AGENTS.md` lists `sqlite_omit_load_extension`, `osusergo`, `netgo`, and `noserver`.
The codebase also uses:

| Tag | Effect |
|---|---|
| `nofirebase` | Replaces Firebase sender with a no-op stub |
| `nopayments` | Replaces Stripe integration with a no-op stub |
| `nowebpush` | Replaces Web Push with a no-op stub |
| `race` | Disables a specific test file (`server_race_off_test.go`) |

Agents that add features gated by these tags, or that run tests, need to know they exist.

### 2. Test helper pattern is undocumented

Tests use `newTestServer(t, newTestConfig(t, databaseURL))` defined at the bottom of
`server/server_test.go`. The `db/test` package provides `CreateTestPostgresSchema` for
isolated per-test PostgreSQL schemas. Agents writing new tests will reinvent or break
this pattern without guidance.

### 3. Local dev server workflow (`.gitpod.yml`) is absent

`.gitpod.yml` documents the intended live-reload workflow:
- `nodemon` watches `*.go` and restarts `go run main.go serve` on port 2586
- `npm start` serves the web app on port 3000
- `mkdocs serve` serves docs on port 8000

This is the fastest inner loop for server development and is not mentioned in `AGENTS.md`.

### 4. `server/site/` and `server/docs/` stub requirement is buried

The requirement to run `make cli-deps-static-sites` before `go test` or `go build` is
mentioned once under "Running tests" but not under "Build System" or as a general
prerequisite. Agents that jump straight to `go test ./...` will get a compile error
because `server/server.go` embeds these directories.

### 5. No guidance on where new code belongs

There is no decision guide for:
- When to add a new file vs extend an existing `server_<feature>.go`
- Where to put shared types (answer: `model/` for domain types, `util/` for helpers)
- How to add a new CLI subcommand (answer: new file in `cmd/`, register in `cmd/app.go`)

### 6. Database layer conventions are missing

- SQLite path: `db/db.go` (wraps `database/sql` + `mattn/go-sqlite3`)
- PostgreSQL path: `db/pg/pg.go` (wraps `jackc/pgx/v5`)
- User/auth management lives in `user/manager.go`, not in `db/`
- Schema migrations are embedded SQL strings in the manager files — no migration tool

Agents that touch persistence will make wrong assumptions without this.

### 7. Web app conventions are absent

- Framework: React + Vite (not Create React App)
- UI library: MUI (Material UI)
- i18n: `i18next` with translation files under `web/src/`
- Entry point: `web/src/index.jsx`
- Built output goes to `server/site/` — this directory is `.gitignore`d and must not be committed

### 8. No PR / contribution checklist

There is no PR template (`.github/PULL_REQUEST_TEMPLATE.md` does not exist).
`AGENTS.md` should document the expected pre-PR checklist so agents don't skip steps.

### 9. Release process is undocumented for agents

The release workflow requires updating `docs/install.md` and `docs/releases.md` with
the new version tag before `make release` will succeed (`release-checks` enforces this).
An agent asked to cut a release will fail without this knowledge.

---

## What's Wrong

### 1. `devcontainer.json` uses the wrong Go version

`devcontainer.json` references `mcr.microsoft.com/devcontainers/go:1.24` in a comment
as the recommended Go image, but `go.mod` requires Go 1.25 and CI uses `1.25.x`.
`AGENTS.md` correctly states Go 1.25+ but the devcontainer comment is misleading.
This is a devcontainer issue, not an `AGENTS.md` error, but `AGENTS.md` should note
the discrepancy so agents don't use the suggested image.

### 2. `make check` description is slightly misleading

`AGENTS.md` says `make check` runs "Tests + fmt-check + vet + web-lint + lint + staticcheck".
The actual target also runs `web-fmt-check` (Prettier check). Minor but an agent relying
on the table to understand what CI validates could miss a Prettier failure.

### 3. Commit message guidance is too vague

The examples given (`Bump`, `Tighten web push endpoint allow list`) are accurate but
the guidance "short imperative subject line, no period" is incomplete — the repo has
no enforced convention and many commits are merge commits or Weblate translation commits.
Agents should be told to use a descriptive subject and that there is no enforced format.

---

## Concrete Improvements to Apply to `AGENTS.md`

### A. Expand the Build Tags section

Add a dedicated "Build Tags" subsection under "Build System":

```markdown
### Build Tags

| Tag | Effect |
|---|---|
| `noserver` | Client-only build; excludes all server code |
| `nofirebase` | Replaces Firebase sender with a no-op (stub file: `server_firebase_dummy.go`) |
| `nopayments` | Replaces Stripe integration with a no-op (stub file: `server_payments_dummy.go`) |
| `nowebpush` | Replaces Web Push with a no-op (stub file: `server_webpush_dummy.go`) |
| `sqlite_omit_load_extension` | Always set for server builds; disables SQLite extension loading |
| `osusergo,netgo` | Static linking; used in production Linux builds |
```

### B. Add a "Stub directories" warning

Under "Development Setup", add a callout:

```markdown
> **Required before `go build` or `go test`:** `server/server.go` embeds
> `server/docs/` and `server/site/`. If these directories are missing, the
> build fails. Run `make cli-deps-static-sites` to create empty stubs.
```

### C. Add a "Writing Tests" section

```markdown
## Writing Tests

- Test helpers live at the bottom of `server/server_test.go`:
  - `newTestConfig(t, databaseURL)` — creates a temp-dir config pointing at SQLite or Postgres
  - `newTestServer(t, config)` — starts a test server and registers cleanup
- For PostgreSQL-specific tests, use `dbtest.CreateTestPostgresSchema(t)` from
  `heckel.io/ntfy/v2/db/test`. It creates an isolated schema and skips the test if
  `NTFY_TEST_DATABASE_URL` is unset.
- Use `require` (not `assert`) from `github.com/stretchr/testify/require` — the project
  uses `require` exclusively in tests.
```

### D. Add a "Local Dev Workflow" section

```markdown
## Local Dev Workflow

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
```

### E. Add a "Where New Code Belongs" section

```markdown
## Where New Code Belongs

| What | Where |
|---|---|
| New HTTP endpoint | `server/server_<feature>.go` (register handler in `server.go`) |
| New CLI subcommand | New file in `cmd/`, register in `cmd/app.go` |
| Shared domain types | `model/` |
| Shared utilities | `util/` |
| Database queries | `db/db.go` (SQLite) or `db/pg/pg.go` (PostgreSQL) |
| User/auth logic | `user/manager.go` |
| Schema changes | Embedded SQL in the relevant manager file; bump the schema version constant |
```

### F. Add a "Web App Conventions" section

```markdown
## Web App Conventions

- Framework: React + Vite
- UI library: MUI (Material UI v5)
- i18n: `i18next`; translation files are in `web/src/`
- Entry point: `web/src/index.jsx`
- Built output: `server/site/` — gitignored, do not commit
- Format with Prettier (`make web-fmt`), lint with ESLint (`make web-lint`)
```

### G. Add a "Pre-PR Checklist" section

```markdown
## Pre-PR Checklist

Run before opening a pull request:

```bash
make cli-deps-static-sites   # ensure stubs exist
make check                   # tests + fmt + vet + lint + staticcheck + web checks
```

For PostgreSQL coverage:
```bash
export NTFY_TEST_DATABASE_URL="postgres://ntfy:ntfy@localhost:5432/ntfy_test?sslmode=disable"
make check
```
```

### H. Fix the `make check` table entry

Change:

```
| `make check` | Tests + fmt-check + vet + web-lint + lint + staticcheck |
```

To:

```
| `make check` | Tests + fmt-check + web-fmt-check + vet + web-lint + lint + staticcheck |
```

### I. Clarify commit message guidance

Replace:

> Follow the existing style: short imperative subject line, no period.

With:

> No enforced format. Use a short, descriptive subject line. Avoid generic messages
> like "fix" or "update". The repo contains many Weblate and Dependabot commits —
> do not use those as style guides for hand-written commits.

### J. Note the devcontainer Go version mismatch

Add a note under "Prerequisites":

> ⚠️ The `devcontainer.json` comment suggests `mcr.microsoft.com/devcontainers/go:1.24`
> but the project requires Go 1.25+. Use a Go 1.25 image or the universal image.

---

## Priority Order

| Priority | Item | Reason |
|---|---|---|
| High | B (stub dirs warning) | Causes immediate build failure for new agents |
| High | A (build tags) | Needed for any feature work touching Firebase/payments/webpush |
| High | C (test helpers) | Needed for any test authoring |
| Medium | E (where code belongs) | Prevents structural mistakes |
| Medium | G (pre-PR checklist) | Prevents CI failures |
| Medium | D (local dev workflow) | Speeds up iteration |
| Low | F (web app conventions) | Only relevant for web changes |
| Low | H (make check fix) | Minor accuracy issue |
| Low | I (commit message) | Style guidance, not blocking |
| Low | J (devcontainer note) | Informational |
