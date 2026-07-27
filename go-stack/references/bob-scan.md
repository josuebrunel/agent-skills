# Bob + Scan reference

`github.com/stephenafamo/bob` — SQL query builder (Postgres/MySQL/SQLite dialects) + a lightweight SQL executor. `github.com/stephenafamo/scan` — scans `database/sql` (or pgx) rows directly into structs; Bob's executor uses this under the hood, so in most cases you don't call `scan`/`stdscan`/`pgxscan` directly — you call Bob's wrapper functions instead. Read this instead of re-deriving the API each project.

## Building queries

Import the dialect package plus the query-mod packages for the query types you need (Postgres shown; `mysql`/`sqlite` mirror this):

```go
import (
    "github.com/stephenafamo/bob/dialect/psql"
    "github.com/stephenafamo/bob/dialect/psql/sm" // select mods
    "github.com/stephenafamo/bob/dialect/psql/im" // insert mods
    "github.com/stephenafamo/bob/dialect/psql/um" // update mods
    "github.com/stephenafamo/bob/dialect/psql/dm" // delete mods
)

psql.Select(sm.From("users"), sm.Where(...))
psql.Insert(im.Into("users"), im.Values(...))
psql.Update(um.Table("users"), um.Set(...), um.Where(...))
psql.Delete(dm.From("users"), dm.Where(...))
psql.Raw("...")
```

Every query object exposes:

```go
query.Build(ctx)                // (string, []any, error)
query.BuildN(ctx, start)        // same, with a custom arg start index (subqueries)
query.MustBuild(ctx)            // panics instead of returning error — fine for queries built once at init/startup
query.MustBuildN(ctx, start)
```

Build once and reuse when a query is static and reused often (`MustBuild` at package/init scope); build fresh per-call when the query varies by input (typical handler case — just call `.Build(ctx)` inline).

## Running queries — prefer Bob's executor over building+scanning by hand

Open a Bob-wrapped connection once (`bob.Open(driverName, dsn)`, or wrap an existing `*sql.DB`) — this gives you Bob's `Executor` (a thin interface: `QueryContext` returning `scan.Rows`, plus `ExecContext`). Then use these wrapper functions instead of manual `db.QueryContext` + scan loops:

| Function | Use for |
|---|---|
| `bob.Exec(ctx, db, query)` | Statements with no rows to scan back (INSERT/UPDATE/DELETE without RETURNING) |
| `bob.One(ctx, db, query, scan.StructMapper[T]())` | Exactly one row → `(T, error)` |
| `bob.All(ctx, db, query, scan.StructMapper[T]())` | All rows → `([]T, error)` |
| `bob.Cursor(ctx, db, query, scan.StructMapper[T]())` | Large result sets you want to stream row-by-row (works like `*sql.Rows`) rather than materializing a full slice |
| `bob.Each(ctx, db, query, scan.StructMapper[T]())` | Range-over-func iteration (Go 1.23+ `for x, err := range ...`) — cleanest for simple row-by-row processing without manually managing a cursor |

```go
type userObj struct {
    ID   int
    Name string
}

q := psql.Select(sm.From("users"), sm.Where(psql.Quote("id").EQ(psql.Arg(id))))

user, err := bob.One(ctx, db, q, scan.StructMapper[userObj]())
users, err := bob.All(ctx, db, q, scan.StructMapper[userObj]())
```

`scan.StructMapper[T]()` maps columns to struct fields by name — the query's selected column names must match (or be aliased to match) the struct's field names.

For a slice with methods on it (not just `[]T`), use `bob.Allx` with `bob.SliceTransformer`:

```go
type userSlice []userObj
func (u userSlice) SomeMethod() {}

users, err := bob.Allx[bob.SliceTransformer[userObj, userSlice]](ctx, db, q, scan.StructMapper[userObj]())
```

## When to use `scan`/`stdscan`/`pgxscan` directly

Only bypass Bob's executor for raw SQL that never goes through the query builder (e.g. a `.sql` file executed as-is). In that case:

```go
import "github.com/stephenafamo/scan/stdscan" // *sql.DB / *sql.Tx / *sql.Conn
// or "github.com/stephenafamo/scan/pgxscan"  // pgx pools/conns

user, err := stdscan.One(ctx, db, scan.StructMapper[userObj](), `SELECT id, name FROM users WHERE id = $1`, id)
users, err := stdscan.All(ctx, db, scan.StructMapper[userObj](), `SELECT id, name FROM users`)
```

`scan.SingleColumnMapper[T]` is the mapper to use when scanning a single column (e.g. `SELECT COUNT(*)`, or `SELECT id FROM users` into `[]int`) rather than a struct.

## Gotchas worth remembering

- Bob's `Executor` interface is intentionally minimal (`QueryContext` + `ExecContext`) — don't assume `*sql.DB`-only methods are available on it; go through `bob.Open`/the wrapper functions rather than reaching into the underlying DB handle mid-query-building.
- `MustBuild`/`MustBuildN` panic on error — only use them where a build failure is truly a programmer error (static, well-tested queries), never on queries built from user input.
- Prefer `bob.Each`/`bob.Cursor` over `bob.All` for result sets that could be large — `bob.All` materializes the full slice in memory.
