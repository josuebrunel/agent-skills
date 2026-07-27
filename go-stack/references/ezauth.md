# EzAuth reference

`github.com/josuebrunel/ezauth` — Email/Password, JWT sessions, OAuth2 (Google/GitHub/Facebook), passwordless/magic-link, password reset. SQLite/Postgres/MySQL. Read this instead of re-exploring the source or README on every project.

## Setup (embedded-in-app pattern — the one Josue uses)

```go
cfg, err := config.LoadConfig() // reads EZAUTH_* env vars
auth, err := ezauth.New(&cfg, "auth") // path = base mount path, e.g. "auth"
// or: ezauth.NewWithDB(&cfg, existingDB, "auth") to reuse an existing *sql.DB
err = auth.Migrate() // run ezauth's own migrations — separate from app's Goose migrations
```

Mount on Echo: wrap `auth.Handler` (a `net/http.Handler`) — Echo v5 supports mounting stdlib handlers via `e.Any("/auth/*", echo.WrapHandler(auth.Handler))` or group-mount equivalent.

Required env vars minimum: `EZAUTH_API_KEY`, `EZAUTH_JWT_SECRET`, `EZAUTH_DB_DIALECT`, `EZAUTH_DB_DSN`. Optional blocks: SMTP (`EZAUTH_SMTP_*`), OAuth2 per-provider (`EZAUTH_OAUTH2_<PROVIDER>_*`), redirect/page URLs (`EZAUTH_REDIRECT_AFTER_LOGIN`, `EZAUTH_LOGIN_PAGE_URL`, etc.), email template overrides.

## Middleware — pick one per route group

| Middleware | Use when |
|---|---|
| `auth.SessionMiddleware` | Default choice for browser/HTMX routes — manages cookie session + loads user |
| `auth.LoginRequiredMiddleware` | Protect a route: redirects browser requests to login page, 401s API requests |
| `auth.LoadUserMiddleware` | Loads user into context without managing session (custom session setup) |
| `auth.AuthMiddleware` | Protect JSON API routes via JWT Bearer token |

## In-handler helpers (package-level, need the right middleware mounted)

```go
ezauth.IsAuthenticated(ctx)      // bool
ezauth.GetUserID(ctx)            // (string, error) — works with session or JWT auth
ezauth.GetUser(ctx)              // (*models.User, error) — needs LoadUserMiddleware/SessionMiddleware
```

Instance methods (on the `*EzAuth` you created): `auth.GetSessionUser(ctx)`, `auth.GetSessionTokens(ctx)` (`access_token`/`refresh_token`), `auth.GetErrorMessage(ctx)` / `auth.GetSuccessMessage(ctx)` (flash messages, auto-cleared on read).

User model helpers — see the dedicated section below.

## `models.User` helper methods

All hang off `*models.User` (the type returned by `ezauth.GetUser(ctx)` / `auth.GetSessionUser(ctx)`). Use these instead of writing your own role checks or ad hoc metadata parsing.

```go
// Role check — roles come from the "roles" field set at registration or via admin update
user.HasRole(role string) bool

// Typed metadata read/write — user-editable metadata (profile prefs, feature flags the user controls)
models.GetMeta[T any](user *models.User, key string) (T, bool)
user.SetMeta(key string, value any)

// Typed metadata read/write — app-controlled metadata (things your app sets, not the user)
models.GetAppMeta[T any](user *models.User, key string) (T, bool)
user.SetAppMeta(key string, value any)
```

Usage:

```go
if user.HasRole("admin") {
    // gate admin-only UI/routes
}

// UserMetadata — user-editable (e.g. a UI theme preference)
if theme, ok := models.GetMeta[string](user, "theme"); ok {
    // use theme
}
user.SetMeta("theme", "dark")

// AppMetadata — app-controlled (e.g. an internal plan tier your billing logic sets)
if tier, ok := models.GetAppMeta[string](user, "plan_tier"); ok {
    // use tier
}
user.SetAppMeta("plan_tier", "pro")
```

Keep the distinction: `UserMetadata` (`*Meta` functions) is meant for values the user themselves can influence; `AppMetadata` (`*AppMeta` functions) is meant for values only your application logic should set (billing tier, internal flags, admin notes) — don't expose `AppMetadata` writes through user-facing forms.

`SetMeta`/`SetAppMeta` update the in-memory struct only — persist the user via the repository/service layer afterward if the change needs to survive past the current request (check `auth.Service`/`auth.Repo` for the update path, or do it inside a hook's `Before/AfterUserUpdated`).

Known accessible fields on `*models.User` (used directly in ezauth's own docs/examples): `Email`, `ID`. Registration accepts `first_name`, `last_name`, `locale`, `timezone`, `roles`, and arbitrary `meta_*` form fields — these map onto corresponding struct fields/metadata, though the exact field names beyond `Email`/`ID` aren't spelled out in current public docs. If unsure of an exact field name, a quick `grep -r "models.User{" $(go env GOPATH)/pkg/mod/github.com/josuebrunel/ezauth*/pkg/db/models/` on the vendored source is faster than guessing.

## CSRF

Form endpoints (`/auth/login`, `/auth/register`, etc.) auto-enforce CSRF via Fetch Metadata headers (`Sec-Fetch-Site`, `Origin`) — **no hidden token required in Templ forms**. Only generate `ezauth.CSRFTemplateField(r)` / `ezauth.CSRFToken(r)` if a specific integration expects a token to be present. JSON API endpoints (`/auth/api/*`) skip CSRF entirely (Bearer auth, no cookies).

## Routes ezauth mounts for you

Form-based (cookies + redirects): `POST /auth/register`, `/auth/login`, `/auth/logout`, `/auth/password-reset/request`, `/auth/password-reset/confirm`, `/auth/passwordless/request`, `GET /auth/passwordless/login`, `GET /auth/oauth2/{provider}/login`, `GET /auth/oauth2/{provider}/callback`.

JSON API: `POST /auth/api/register`, `/auth/api/login`, `/auth/api/token/refresh`, `/auth/api/password-reset/request`, `/auth/api/password-reset/confirm`, `/auth/api/passwordless/request`, `GET /auth/api/passwordless/login`, `GET /auth/api/userinfo` (protected), `POST /auth/api/logout` (protected), `DELETE /auth/api/user` (protected).

Register form requires `email`, `password`, `password_confirm` (8+ char passwords); optional `first_name`, `last_name`, `locale`, `timezone`, `roles`, `meta_*`.

## Hooks (lifecycle events — for welcome emails, audit logs, banned-domain checks, etc.)

Embed `service.DefaultHook`, override only what's needed:

```go
type MyHook struct {
    service.DefaultHook
    db  *sql.DB
    log *slog.Logger
}

func (h MyHook) BeforeUserCreated(ctx context.Context, u *models.User) error {
    // return non-nil to abort the operation (e.g. banned email domain)
}

func (h MyHook) AfterUserCreated(ctx context.Context, u *models.User) error {
    // errors here are only logged, not fatal — good for async side effects
}
```

Register any time, even post-startup: `auth.SetHook(MyHook{db: sqlDB, log: slog.Default()})`.

Available hooks: `BeforeUserCreated`/`AfterUserCreated`, `BeforeUserUpdated`/`AfterUserUpdated`, `BeforeUserDeleted`/`AfterUserDeleted`, `AfterUserSignedIn`, `AfterUserSignedOut`, `AfterPasswordResetRequested`, `AfterPasswordResetConfirmed`, `AfterOAuth2SignedIn`, `AfterOAuth2Created`. Only the `Before*` hooks can abort (non-nil error); `After*` hook errors are logged only.

## Gotchas worth remembering

- `auth.Migrate()` runs EzAuth's own internal migrations — keep this separate from the app's own Goose-managed migrations; don't try to fold EzAuth's schema into the app's `NewProvider` migration set.
- Session cookie name is `ezauthsess`; tokens live under the `tokens` key inside it.
- `GetUserID` works across both session and JWT auth paths — prefer it over `GetSessionUser` when you only need the ID (cheaper, fewer failure modes).
