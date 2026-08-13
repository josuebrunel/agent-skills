---
name: go-stack
description: Josue's default backend stack for new Go services and features. Use this whenever scaffolding a new Go project, adding a web endpoint, setting up auth, writing a migration, adding a background job, or building a UI fragment — even if the user doesn't name the tools explicitly. Covers Echo v5, Templ, EzAuth, Goose (isolated NewProvider), RiverQueue, HTMX, DaisyUI v5, and Chart.js, plus the required Makefile and docker-compose.yml scaffolding. Builds on the cross-stack dev-principles skill for general coding standards (DRY, testing, security, logging, error handling, config, transactions) — this skill adds their Go-specific instantiation. Make sure to check this skill before reaching for a different framework, ORM, or auth library, or before writing raw net/http, plain SQL migration tooling, or a different frontend approach — this stack is the default unless the user explicitly asks for something else.
---

# Josue's Go Stack

This is the default stack for backend/full-stack Go work. Reach for these tools automatically rather than defaulting to generic net/http, a different template engine, or a different job queue — only deviate if explicitly asked.

## Stack overview

| Concern            | Tool          |
|---------------------|---------------|
| Web framework       | Echo v5       |
| Templates           | Templ         |
| Auth                | EzAuth        |
| Query builder       | Bob (github.com/stephenafamo/bob) |
| SQL scanning        | Scan (github.com/stephenafamo/scan) — used via Bob's executor |
| Migrations          | Goose (isolated instance via `NewProvider`) |
| Background jobs     | RiverQueue    |
| Frontend interactivity | HTMX      |
| UI components/styling | DaisyUI v5 (Tailwind-based) |
| Charts | Chart.js (pinned CDN version) |
| Config | xenv (github.com/josuebrunel/gopkg/xenv) |

## Conventions

### Echo v5
- Use Echo v5's router and middleware idioms (not v4 patterns — check for v5-specific API changes when unsure, e.g. context/middleware signatures).
- Group routes logically (e.g. `/api`, `/auth`, `/htmx` fragments) rather than one flat router.

### Templ
- `.templ` files live alongside the handlers that render them, or in a dedicated `views/` package — ask which layout the project uses if unclear.
- Compose templates with layout + partial components rather than duplicating markup; favor small, composable `templ` components that HTMX fragments can target directly.

### EzAuth
- Use EzAuth for session/auth handling rather than hand-rolling JWT or session logic.
- Wire it through Echo middleware so protected routes/groups get auth checked consistently.
- **Read `references/ezauth.md` before writing any EzAuth integration code.** It has the setup pattern, middleware table, in-handler helpers, CSRF behavior, full route list, and hook system — read it instead of re-deriving the API from the source or README each time.

### Bob (query builder) + Scan (row scanning)
- Use Bob to build SQL (`psql`/`mysql`/`sqlite` dialect packages) instead of hand-writing query strings or reaching for an ORM like GORM. Use Bob's executor (`bob.One`/`bob.All`/`bob.Cursor`/`bob.Each`/`bob.Exec`) to run the built query and scan results straight into structs — don't drop down to raw `*sql.DB` query/scan loops when Bob covers the case.
- Scan (the mapper library Bob's executor uses under the hood) is what actually maps rows to structs via `scan.StructMapper[T]()` — reach for it directly (via `stdscan`/`pgxscan`) only for queries that aren't going through Bob's query builder (e.g. a raw SQL file executed outside Bob).
- **Read `references/bob-scan.md` before writing any query-building or row-scanning code.** It covers the dialect/query-mod import pattern, `Build`/`MustBuild`, the executor functions and which to use when, and how Scan's struct mapping fits in.

### Goose — isolated instance pattern
- Always use Goose's `NewProvider` to create an isolated migration provider rather than relying on Goose's package-level global state. This avoids global config collisions when the app also runs tests or multiple DB connections.
- Migrations go in a dedicated `migrations/` directory, run through the provider at startup or via a CLI command — not ad hoc SQL scripts.
- Always expose a `-migrate <up|down|revert>` CLI flag/command (or equivalent subcommand) wired to the Goose provider, so migrations can be run or rolled back without a separate tool — don't ship a service with Goose wired in but no way to invoke it from the CLI.
- Write migration DDL with `IF EXISTS` / `IF NOT EXISTS` guards (per the dev-principles skill) so each numeric migration is idempotent and re-appliable without manual fixups.

### RiverQueue
- Use River for background/async work (emails, notifications, scheduled jobs, retries) instead of goroutines with manual retry logic.
- Define job args as typed structs implementing River's `Kind()`/args interface; register workers explicitly.

### HTMX + Templ + DaisyUI
- Server-rendered fragments returned via Templ components, targeted with HTMX attributes (`hx-get`, `hx-post`, `hx-target`, `hx-swap`) rather than building a separate JSON API + JS frontend for typical CRUD/interactive UI.
- Style with DaisyUI v5 component classes on top of Tailwind; avoid hand-rolled CSS unless DaisyUI doesn't cover the case.

### Chart.js
- Use Chart.js for any chart/graph need (dashboards, analytics, reports, stats) — don't hand-roll SVG/canvas drawing or generate chart images server-side.
- Load it from a pinned CDN version (exact version number, no `@latest`) so behavior is reproducible.
- Serve chart data as JSON embedded in the Templ fragment (via `templ.JS`/`templ.JSON` into the chart config) rather than a separate chart-data JSON API.
- Because HTMX swaps fragments, re-initialize charts on swap: destroy the previous Chart.js instance before re-creating on the swapped `<canvas>` (e.g. an `hx-on::after-swap` handler keyed off the canvas) to avoid double-instance/canvas-reuse errors.

### Documentation
- Keep project documentation (README, setup docs, API/route docs) in sync with code changes — when a change adds/removes a route, config option, CLI flag, or setup step, update the relevant docs in the same change rather than leaving them stale.

## Coding standards, security & DRY

**Read the `dev-principles` skill for the underlying cross-stack principles (idiomatic code, error handling, security, DRY, testing, logging, context propagation, transactions, centralized error handling, config management).** Below is how each principle is instantiated with this stack's specific tools — apply the general rule via these concrete mechanisms rather than re-deriving it.

- **Idiomatic code** → `gofmt`/`goimports`, `go vet`/linter clean, standard Go naming (MixedCaps, short receiver names, doc comments on exported identifiers starting with the identifier name).
- **Error handling** → wrap with `fmt.Errorf("...: %w", err)` for context; don't swallow errors or use `panic` for expected/recoverable failures.
- **Security** → never build SQL by string concatenation, use Bob's query builder/parameter binding for all queries (see `references/bob-scan.md`); validate/sanitize external input (HTTP params, form data, HTMX payloads) at the handler boundary; rely on EzAuth's built-in CSRF/session protections (see `references/ezauth.md`) instead of hand-rolling auth/CSRF, wiring every route touching user data through its auth middleware; enforce least-privilege authz checks (`HasRole` etc.) on every protected handler, not just at the router-group level.
- **DRY** → applies across layers: Go helpers, Bob query fragments, and Templ components — extending the Templ-composition note above.
- **Testing** → new handlers, jobs, and non-trivial helpers ship with tests in the same change; use table-driven tests with the standard `testing` package by default, only reach for `testify` if the project already depends on it; test files live alongside the code they cover (`foo.go` → `foo_test.go`).
- **Structured logging** → `log/slog` with structured key-value fields instead of `fmt.Println`/`log.Printf`; consistent levels (`Debug`/`Info`/`Warn`/`Error`); never log secrets, tokens, or full request/session payloads.
- **Context propagation** → thread `context.Context` through handlers → Bob queries → River job args/workers, rather than starting a fresh `context.Background()` deep in the call stack; respect cancellation/timeouts from the incoming request context.
- **Transaction handling** → wrap multi-step DB writes (e.g. create-then-update-related-row) in an explicit Bob transaction with rollback on error; keep scope tight — open right before the first write, commit/rollback right after the last.
- **Centralized error handling** → a single Echo `HTTPErrorHandler` (or equivalent centralized middleware) mapping errors to consistent HTTP status codes and response shape; handlers return/propagate errors (`return err` / `return echo.NewHTTPError(...)`) rather than writing the response directly on every error path.
- **Config management** → load environment variables into a single typed config struct via `github.com/josuebrunel/gopkg/xenv` (struct tags `env`, `default`, `required`) once at startup, instead of scattered `os.Getenv` calls; use nested structs for grouped settings and `xenv.Options{Prefix: "..."}`/`LoadWithOptions` to namespace keys (e.g. `APP_`); fail fast at startup if a `required` field is missing or a value fails to parse.
- **Concurrency** → take advantage of Go's concurrency model (goroutines + channels + `context.Context`) wherever the shape of the work permits — fan-out/fan-in independent calls, parallelize I/O or sub-queries, pipeline stages — instead of serializing work that can run concurrently. Prefer message passing over shared mutable state (reserve `sync.Mutex`/`atomic` for genuine shared-state cases), collect results via `errgroup`/`sync.WaitGroup`, and always bound concurrency (semaphore/worker pool) and propagate cancellation so goroutines can't leak. For small projects and scripts, goroutines/channels are the default even for background work; River, with its durability/retries/scheduling, is the exception reserved for larger services where jobs must survive restarts — there, prefer River over ad hoc goroutines (see the RiverQueue section above).

## Required project scaffolding: Docker Compose & Makefile

Every go-stack project ships both of these — create them when scaffolding the project, not as an afterthought, and keep them in sync as the project evolves (new compose service, new Makefile target), consistent with the Documentation note above.

### Docker Compose
This is go-stack's instantiation of the generic "local dependencies via Docker Compose" rule in `dev-principles`. `docker-compose.yml` at the repo root defines:
- A `postgres` service with a named volume and env-based credentials matching the app's config struct.
- An `app` service built from the project's Dockerfile, so `docker compose up` runs the full stack — DB and app — not just the database.

### Makefile
Go-specific requirement (not part of `dev-principles`). Every go-stack project ships a `Makefile` at the repo root with, at minimum, these targets:
- `build` — compile the binary.
- `run` — run the app locally.
- `test` — run the test suite.
- `lint` — `go vet` / linter (e.g. `golangci-lint`).
- `migrate-up` / `migrate-down` — wired to the Goose CLI subcommand required in the Goose section above.
- `docker-up` / `docker-down` — wrap `docker compose up`/`down` for the compose file above.
- `tidy` — `go mod tidy`.

## When NOT to apply this

If the user explicitly asks for a different framework, database tool, job queue, or frontend approach (e.g. "use Chi instead" or "no HTMX, I want a React SPA here"), follow their explicit request instead of defaulting to this stack.
