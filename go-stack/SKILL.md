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

### RiverQueue
- Use River for background/async work (emails, notifications, scheduled jobs, retries) instead of goroutines with manual retry logic.
- Define job args as typed structs implementing River's `Kind()`/args interface; register workers explicitly.

### HTMX + Templ + DaisyUI
- Server-rendered fragments returned via Templ components, targeted with HTMX attributes (`hx-get`, `hx-post`, `hx-target`, `hx-swap`) rather than building a separate JSON API + JS frontend for typical CRUD/interactive UI.
- Style with DaisyUI v5 component classes on top of Tailwind; avoid hand-rolled CSS unless DaisyUI doesn't cover the case.

## When NOT to apply this

If the user explicitly asks for a different framework, database tool, job queue, or frontend approach (e.g. "use Chi instead" or "no HTMX, I want a React SPA here"), follow their explicit request instead of defaulting to this stack.
