---
name: go-stack
description: Josue's default Go backend stack — Echo v5, Templ, EzAuth, Bob+Scan, Goose (isolated NewProvider), RiverQueue, HTMX, DaisyUI v5, Chart.js, xenv; ships Makefile + docker-compose.yml. Use for any new Go project, endpoint, auth, migration, job, or UI; check before reaching for another framework/ORM/frontend — this stack is the default unless the user asks otherwise. Adds Go-specific instantiation of dev-principles.
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
- The shared base layout's `<head>` ships `<title>`, meta description, and Open Graph/Twitter Card tags (see `web-design` §"Structure & accessibility") — don't lose them on incremental work that skips `web-design`.

### EzAuth
- Use EzAuth for session/auth handling rather than hand-rolling JWT or session logic.
- Wire it through Echo middleware so protected routes/groups get auth checked consistently.
- Always add app-side `GET` handlers (e.g. `/login`, `/register`) that render the form and POST to EzAuth's `/auth/login`/`/auth/register` — never link straight to the `/auth/*` POST endpoints. Point `EZAUTH_LOGIN_PAGE_URL`/`EZAUTH_REGISTER_PAGE_URL` at them to avoid redirect loops.
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

The `dev-principles` skill holds the cross-stack rules; apply them through these Go-specific mechanisms rather than re-deriving them.

| Principle | Go instantiation |
|---|---|
| Idiomatic code | `gofmt`/`goimports`, `go vet`/linter clean, MixedCaps, short receivers, doc comments on exported identifiers. |
| Error handling | Wrap with `fmt.Errorf("...: %w", err)`; no panics for expected/recoverable failures. |
| Security | Bob parameter binding for all SQL, never string concatenation (see `references/bob-scan.md`); validate/sanitize HTTP params, form data, HTMX payloads at the handler boundary; rely on EzAuth's CSRF/sessions (see `references/ezauth.md`), never hand-rolled auth; auth middleware on every route touching user data; least-privilege `HasRole` authz per protected handler. |
| DRY | Shared Go helpers, Bob query fragments, Templ components across layers. |
| Testing | New handlers/jobs/helpers ship tests in the same change; table-driven tests with std `testing` (only `testify` if already a dep); `foo_test.go` beside `foo.go`. |
| Structured logging | `log/slog` key-value fields, consistent levels, never log secrets/tokens/full payloads. |
| Context propagation | Thread `context.Context` through handlers → Bob queries → River jobs; respect cancellation/timeouts, no fresh `context.Background()` deep in the stack. |
| Transactions | Explicit Bob transaction (rollback on error) for multi-step writes; tight scope. |
| Centralized errors | Single Echo `HTTPErrorHandler` mapping errors → status/response shape; handlers propagate (`return err` / `echo.NewHTTPError(...)`), don't write responses per path. |
| Config | One typed struct via `github.com/josuebrunel/gopkg/xenv` at startup (`env`/`default`/`required` tags, nested structs, `xenv.Options{Prefix}` → `APP_` keys); fail fast on missing/invalid values. |
| Concurrency | Fan out parallel I/O with goroutines + channels + `errgroup`, bounded (semaphore/worker pool), cancellation propagated, message passing over shared state. River (durable, retried) only when jobs must survive restarts in larger services; goroutines otherwise. |

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

If the user explicitly asks for a different framework, ORM, job queue, or frontend approach, follow them — don't default to this stack (see `dev-principles` for the general rule).
