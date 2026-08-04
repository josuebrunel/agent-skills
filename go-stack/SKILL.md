---
name: go-stack
description: Josue's default backend stack for new Go services and features. Use this whenever scaffolding a new Go project, adding a web endpoint, setting up auth, writing a migration, adding a background job, or building a UI fragment — even if the user doesn't name the tools explicitly. Covers Echo v5, Templ, EzAuth, Goose (isolated NewProvider), RiverQueue, HTMX, and DaisyUI v5. Make sure to check this skill before reaching for a different framework, ORM, or auth library, or before writing raw net/http, plain SQL migration tooling, or a different frontend approach — this stack is the default unless the user explicitly asks for something else.
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

### RiverQueue
- Use River for background/async work (emails, notifications, scheduled jobs, retries) instead of goroutines with manual retry logic.
- Define job args as typed structs implementing River's `Kind()`/args interface; register workers explicitly.

### HTMX + Templ + DaisyUI
- Server-rendered fragments returned via Templ components, targeted with HTMX attributes (`hx-get`, `hx-post`, `hx-target`, `hx-swap`) rather than building a separate JSON API + JS frontend for typical CRUD/interactive UI.
- Style with DaisyUI v5 component classes on top of Tailwind; avoid hand-rolled CSS unless DaisyUI doesn't cover the case.

### Documentation
- Keep project documentation (README, setup docs, API/route docs) in sync with code changes — when a change adds/removes a route, config option, CLI flag, or setup step, update the relevant docs in the same change rather than leaving them stale.

## Coding standards, security & DRY

Cross-cutting standards that apply on top of the per-tool conventions above.

### Idiomatic, standards-compliant code
- Follow standard Go idioms: `gofmt`/`goimports`, `go vet`/linter clean, standard naming (MixedCaps, short receiver names, doc comments on exported identifiers starting with the identifier name).
- Handle errors explicitly — wrap with `fmt.Errorf("...: %w", err)` for context; don't swallow errors or use `panic` for expected/recoverable failures.
- Keep package boundaries idiomatic (no import cycles, small focused packages) rather than one large `utils` dumping ground.

### Security
- Never build SQL by string concatenation — use Bob's query builder/parameter binding for all queries (see `references/bob-scan.md`), the primary injection defense in this stack.
- Validate/sanitize external input (HTTP params, form data, HTMX payloads) at the handler boundary before it reaches business logic or the DB.
- Never hardcode secrets/API keys/credentials — load from env/config, and never log them.
- Rely on EzAuth's built-in CSRF/session protections (see `references/ezauth.md`) instead of hand-rolling auth/CSRF; wire every route touching user data through its auth middleware.
- Enforce least-privilege authz checks (`HasRole` etc.) on every protected handler, not just at the router-group level.

### DRY
- Before writing a new helper, query, or component, search the codebase for an existing one that does the same thing — reuse/extend it instead of duplicating.
- Extract shared logic (validation, formatting, query fragments) into a common function/package once used in more than one place, instead of copy-pasting.
- Applies across layers — Go helpers, Bob query fragments, and Templ components — extending the Templ-composition note above.

### Testing
- New handlers, jobs, and non-trivial helpers ship with tests in the same change, not as a follow-up.
- Use table-driven tests with the standard `testing` package as the default; only reach for `testify` if the project already depends on it.
- Test files live alongside the code they cover (`foo.go` → `foo_test.go`), matching Go convention.

### Structured logging
- Use `log/slog` with structured key-value fields instead of `fmt.Println`/`log.Printf` — makes logs greppable/parseable.
- Use consistent log levels (`Debug`/`Info`/`Warn`/`Error`) rather than logging everything at one level.
- Never log secrets, tokens, or full request/session payloads — ties directly into the Security subsection above.

### Context propagation
- Thread `context.Context` through handlers → Bob queries → River job args/workers, rather than starting a fresh `context.Background()` deep in the call stack.
- Respect cancellation/timeouts from the incoming request context so slow DB calls or downstream requests don't outlive a client that's gone away.

### Transaction handling
- Wrap multi-step DB writes (e.g. create-then-update-related-row) in an explicit Bob transaction with rollback on error, instead of issuing sequential unguarded writes that can leave partial state on failure.
- Keep transaction scope tight — open it right before the first write, commit/rollback right after the last, don't hold it open across unrelated work.

### Centralized error handling
- Use a single Echo `HTTPErrorHandler` (or equivalent centralized middleware) that maps errors to consistent HTTP status codes and response shape, instead of ad hoc `c.JSON(500, ...)` scattered per handler.
- Handlers return/propagate errors (`return err` / `return echo.NewHTTPError(...)`) rather than writing the response directly on every error path.

### Config management
- Load configuration from environment variables into a single typed config struct once at startup, rather than scattered `os.Getenv` calls throughout the codebase.
- Fail fast at startup if required config is missing/invalid, instead of discovering it later at request time.

## When NOT to apply this

If the user explicitly asks for a different framework, database tool, job queue, or frontend approach (e.g. "use Chi instead" or "no HTMX, I want a React SPA here"), follow their explicit request instead of defaulting to this stack.
