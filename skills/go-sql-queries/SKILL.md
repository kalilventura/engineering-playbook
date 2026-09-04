---
name: go-sqlx-queries
description: Use when writing or reviewing Go code that touches a SQL database with sqlx/goqu - INSERT/SELECT query shape, bulk inserts, choosing between sqlx and the goqu query builder, or a query that's grown too complex to maintain
---

# Go SQL Queries with sqlx + goqu

## Overview

Stack assumption: `sqlx` is the default for known/fixed queries. `goqu` is used
only where queries are genuinely dynamic (optional filters, dynamic sort).
Do not reach for goqu just to avoid writing SQL — that trade removes SQL
readability for cases that don't need it.

## When to Use

- Writing a new INSERT or SELECT against Postgres/MySQL in Go.
- Deciding between sqlx raw SQL and the goqu query builder.
- Structuring repository/DAO functions or bulk-insert logic.
- Reviewing a query that's become hard to read or maintain.

## 1. INSERT queries — use NamedExec with struct tags

Never hand-roll `INSERT INTO (a, b, c) VALUES ($1, $2, $3)` with positional
args pulled from a struct one field at a time. Tag the struct once and let
sqlx bind by name.

```go
type User struct {
    ID    int    `db:"id"`
    Name  string `db:"name"`
    Email string `db:"email"`
}

func InsertUser(db *sqlx.DB, u User) error {
    _, err := db.NamedExec(`
        INSERT INTO users (name, email)
        VALUES (:name, :email)
    `, u)
    return err
}
```

Rules:
- The struct's `db` tags are the single source of truth for column names.
  Don't duplicate column lists as separate string constants unless you also
  need them for SELECT (see below).
- If you need the inserted ID back, use `NamedQuery` (or `QueryRowx` with
  `RETURNING id` on Postgres) instead of `NamedExec`.
- For inserting more than one row, don't loop single-row `NamedExec` calls
  — see "Bulk inserts" below for the batched alternatives and when to use
  each.

### Bulk inserts — pick based on batch size

**Anti-pattern: looping single-row inserts.** Never do this for more than a
handful of rows — it's N round-trips to the DB and N separate transactions'
worth of overhead:

```go
// DON'T: one round-trip per row
for _, u := range users {
    db.NamedExec(`INSERT INTO users (name, email) VALUES (:name, :email)`, u)
}
```

**Small-to-medium batches (roughly tens to a few hundred rows): single
multi-VALUES statement via `db.NamedExec` with a slice.** sqlx expands a
slice argument to `NamedExec` into one `VALUES (...), (...), (...)`
statement automatically — one round-trip, still fully parameterized:

```go
func InsertUsers(db *sqlx.DB, users []User) error {
    _, err := db.NamedExec(`
        INSERT INTO users (name, email)
        VALUES (:name, :email)
    `, users)
    return err
}
```

This is the default choice for bulk inserts in application code: simple,
safe, one round-trip. Watch the driver's parameter limit though (Postgres
caps at 65535 bind params per statement) — for very wide tables or very
large batches, chunk the slice (e.g. 500 rows per call) rather than sending
everything in one statement.

```go
func InsertUsersChunked(db *sqlx.DB, users []User, chunkSize int) error {
    for i := 0; i < len(users); i += chunkSize {
        end := i + chunkSize
        if end > len(users) {
            end = len(users)
        }
        if _, err := db.NamedExec(`
            INSERT INTO users (name, email)
            VALUES (:name, :email)
        `, users[i:end]); err != nil {
            return err
        }
    }
    return nil
}
```

**Large batches (thousands+ rows, e.g. bulk import/ETL jobs): use
`pq.CopyIn` (Postgres) instead of INSERT entirely.** `COPY` is
dramatically faster than batched INSERTs for large volumes because it
skips per-row planning/logging overhead. This bypasses `sqlx`'s named-param
convenience and goes through the raw `*sql.DB`/`*sql.Tx`:

```go
import "github.com/lib/pq"

func BulkInsertUsersCopy(db *sqlx.DB, users []User) error {
    txn, err := db.Begin()
    if err != nil {
        return err
    }
    defer txn.Rollback() // no-op if committed

    stmt, err := txn.Prepare(pq.CopyIn("users", "name", "email"))
    if err != nil {
        return err
    }

    for _, u := range users {
        if _, err := stmt.Exec(u.Name, u.Email); err != nil {
            return err
        }
    }
    if _, err := stmt.Exec(); err != nil { // flush
        return err
    }
    if err := stmt.Close(); err != nil {
        return err
    }
    return txn.Commit()
}
```

Trade-off: `COPY` doesn't support `ON CONFLICT`/upsert semantics or
returning generated values per row. If you need conflict handling on a
large batch, either fall back to chunked multi-VALUES INSERT with
`ON CONFLICT`, or `COPY` into a temporary staging table and then run a
single `INSERT ... SELECT ... ON CONFLICT` from staging into the real
table.

**Rule of thumb:**
| Batch size | Approach |
|---|---|
| 1 row | plain `NamedExec` |
| ~2–500 rows | `NamedExec` with a slice (chunk if very wide/many columns) |
| 500+ rows, no conflict handling needed | `pq.CopyIn` |
| 500+ rows, need upsert | `COPY` into staging table, then `INSERT ... SELECT ... ON CONFLICT` |

## 2. SELECT queries — use Select/Get, keep column lists as constants

`Get` (single row) and `Select` (multiple rows) already eliminate manual
`rows.Scan` boilerplate via `StructScan` internally. The remaining
boilerplate — writing the column list — should be centralized once per
struct, not repeated per query:

```go
const userCols = `id, name, email`

func GetUser(db *sqlx.DB, id int) (User, error) {
    var u User
    err := db.Get(&u, `SELECT `+userCols+` FROM users WHERE id = $1`, id)
    return u, err
}

func ListActiveUsers(db *sqlx.DB) ([]User, error) {
    var users []User
    err := db.Select(&users, `SELECT `+userCols+` FROM users WHERE active = true`)
    return users, err
}
```

Rules:
- One `const <entity>Cols` per struct that maps to a table, next to the
  struct definition. Update it in one place when a column is added/removed.
- Prefer `SELECT <explicit columns>` over `SELECT *` even with StructScan —
  explicit columns don't silently break when the table gains a column sqlx
  can't map, and make diffs reviewable.
- Always use `$1, $2, ...` (or driver-appropriate) positional/named params.
  Never format user input into the query string with `fmt.Sprintf`, string
  concatenation, or `+`, even for values that "look safe" (IDs, enums,
  booleans). This is the actual SQL-injection boundary — not raw SQL vs.
  builder.

## 3. When to switch to goqu: dynamic/conditional queries

The signal for goqu is **the WHERE/ORDER BY clause itself changes shape at
runtime** — not "this query is long" or "I don't want to write SQL by hand".
Classic case: a GET endpoint with several optional query params.

```go
func ListOrders(db *sqlx.DB, status string, minPrice float64, sortBy string) ([]Order, error) {
    query := goqu.From("orders").Select("*")

    if status != "" {
        query = query.Where(goqu.Ex{"status": status})
    }
    if minPrice > 0 {
        query = query.Where(goqu.C("price").Gte(minPrice))
    }

    // ORDER BY column must come from an allowlist — never pass a raw
    // query param straight into Order()/goqu.I(), even though goqu
    // parameterizes values, dynamic identifiers are a different risk.
    allowedSorts := map[string]bool{"created_at": true, "price": true, "name": true}
    if !allowedSorts[sortBy] {
        sortBy = "created_at"
    }
    query = query.Order(goqu.I(sortBy).Asc())

    sql, args, err := query.ToSQL()
    if err != nil {
        return nil, err
    }

    var orders []Order
    err = db.Select(&orders, sql, args...)
    return orders, err
}
```

Rules:
- Keep goqu scoped to the handful of endpoints/functions that actually need
  conditional filters. Don't convert fixed CRUD queries to goqu just for
  consistency — that adds reflection overhead and an indirection layer for
  no benefit.
- Always validate any identifier that comes from user input (sort column,
  sort direction, dynamic field name) against an explicit allowlist before
  it reaches the builder. goqu parameterizes *values* safely; it does not
  vet column/identifier names for you.
- Log or trace the generated SQL (`query.ToSQL()`) in dev/staging for any
  goqu-built query, since — unlike raw SQL in the codebase — you can't just
  read the source to know what ran. If tracing (e.g. OTel) is set up, attach
  the generated SQL as a span attribute.
- Paginate with `.Limit()` / `.Offset()` on the same builder rather than a
  second handwritten string.

## 4. When a query gets too complex

There's no single right answer here — the options below are trade-offs, not
a ladder to climb in order. Consider the situation and pick deliberately;
whatever you choose, leave a comment saying why, so the next person doesn't
"simplify" it back into the problem you were avoiding.

**Option A — Keep it as raw SQL in sqlx, even if long.**
Multi-line raw SQL (CTEs, window functions, complex joins) is often *more*
maintainable than trying to force it through goqu, which gets awkward or
unsupported for these constructs fast. A well-formatted 40-line query with
a comment explaining intent beats a builder chain nobody can mentally
translate back to SQL. This is usually the right default when the
complexity is inherent to the question being asked of the database.

**Option B — Split into multiple simpler queries, compose in Go.**
If the complexity comes from combining unrelated concerns (e.g. joining
data that's really "fetch A, then fetch related B by A's IDs"), consider
issuing two simpler, indexable queries and joining in application code.
Trade-off: more round-trips to the DB, but each query stays simple, testable,
and cacheable independently. Watch for N+1 if this is done per-row instead
of batched (`WHERE id = ANY($1)`).

**Option C — Push the complexity into the database (view / materialized view).**
If the same complex shape is needed repeatedly across the codebase, a SQL
view (or materialized view, if it's expensive and can tolerate staleness)
moves the complexity to one place in the schema instead of duplicating a
gnarly query across Go call sites.

**Option D — Escape hatch to sqlc for just that one query.**
If a specific query is complex *and* performance/type-safety-critical, it's
fine to generate a typed function for that single query with sqlc while the
rest of the codebase stays on sqlx/goqu. This isn't an all-or-nothing
migration — sqlc-generated code and sqlx can coexist since both ultimately
use `database/sql` underneath.

**Option E — Accept the goqu chain, but extract it into a named function.**
If it must stay dynamic, at least give the builder chain its own
well-named function (`buildOrderSearchQuery(...)`) instead of inlining it
in the handler, and add a test that asserts the generated SQL string for a
few representative input combinations.

## 5. Quick checklist before merging a new query

- [ ] No string concatenation/`fmt.Sprintf` of values into SQL — parameters
      only.
- [ ] Any dynamic identifier (sort column, table name, direction) validated
      against an allowlist.
- [ ] SELECT uses explicit columns, not `*`, backed by a `const ...Cols`.
- [ ] If goqu is used, confirm the query is actually conditional — if it
      always produces the same SQL, move it to sqlx raw SQL instead.
- [ ] If the query is complex, there's a comment stating which option (A–E
      above) was chosen and why.
- [ ] For anything on a hot path or with tracing set up, the final SQL is
      visible in logs/spans, not just the builder call.

## Common Mistakes

- Formatting user input into a query string with `fmt.Sprintf`/concatenation
  instead of using positional/named parameters.
- Looping single-row `NamedExec` calls for a bulk insert instead of a
  multi-VALUES statement or `pq.CopyIn`.
- Using `SELECT *` with StructScan instead of an explicit `const ...Cols`.
- Reaching for goqu on a query that's always the same shape, instead of
  keeping it as plain sqlx.
- Passing a user-supplied sort column/direction straight into goqu without
  validating it against an allowlist first.
- Letting a genuinely complex query balloon in place without picking one of
  the documented options (A–E) and leaving a comment saying why.