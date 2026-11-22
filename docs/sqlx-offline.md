# Compile-time checked SQL (sqlx offline mode)

Every production query in this backend goes through sqlx's compile-time
checked macros (`sqlx::query!`, `sqlx::query_as!`, `sqlx::query_scalar!`)
instead of the runtime-checked `sqlx::query()` / `sqlx::query_as()` /
`sqlx::query_scalar()` functions. A typo in a column name, a wrong bind
count, or a type mismatch is now a compile error instead of a runtime
failure.

The macros need the output of SQLite's `DESCRIBE` for each statement.
That metadata is committed under `.sqlx/` as one `query-<hash>.json` file
per query, where `<hash>` is the SHA-256 of the query string. With the
metadata present, the macros compile without a live database
(`SQLX_OFFLINE=true` or simply no `DATABASE_URL` set).

## Regenerating the cache

### When to regenerate

Regenerate `.sqlx/` whenever you:

- change the SQL text of an existing `sqlx::query!` / `sqlx::query_as!` / `sqlx::query_scalar!` call, or
- add a new query macro, or
- change the result type of an existing macro query (column alias, override, or struct field).

The hash in each `query-<hash>.json` filename is the SHA-256 of the query string, so any
edit to the query string produces a new file and orphans the old one. If you forget to
regenerate, CI will catch it (see below).

### The command

Point `DATABASE_URL` at a database that has had all migrations applied, then run:

```bash
# Apply migrations if you have not already (first time or after adding a migration):
DATABASE_URL=sqlite://zenith.db sqlx migrate run

# Regenerate every query-*.json file in .sqlx/:
DATABASE_URL=sqlite://zenith.db cargo sqlx prepare
```

`cargo sqlx prepare` re-describes every macro query against the live database and
rewrites the matching files under `.sqlx/`. The output lands in `.sqlx/` at the
workspace root — one `query-<hash>.json` file per distinct query string.

### Committing the result

The regenerated `query-*.json` files are part of the source tree and **must be committed
in the same commit as the query change** that produced them. Reviewers should see the
metadata and the query change together.

### Makefile

There is no dedicated `make` target for `cargo sqlx prepare`. Run the command directly
as shown above.

### CI check

CI runs `cargo sqlx prepare --check` against a freshly migrated database on every
push/PR. It fails if `.sqlx/` is missing a file for any query or if any existing file
differs from what the live database describes — that is exactly the case when you forgot
to run `cargo sqlx prepare` after editing a query.

## Building without a database

The Docker build and `cargo build`/`cargo test` have no database available.
The macros fall back to the committed `.sqlx/` metadata automatically when
`DATABASE_URL` is unset. The Dockerfile also sets `SQLX_OFFLINE=true` in
the build stage so a stray `DATABASE_URL` can't trigger a live describe
against a non-existent database.

## Column overrides

SQLite's planner cannot always prove a column's type or nullability (for
expressions, aggregates, and `TEXT PRIMARY KEY` columns, which SQLite
reports as nullable). Where the inferred type would be wrong, the query
uses a column override in the SQL itself:

```sql
SELECT
    id AS "id!",                       -- "!": force non-null
    triggered AS "triggered: bool",    -- ": bool": force this Rust type
    SUM(x) AS "total: _"               -- ": _": infer from the struct field
FROM ...
```

- `name!` — force the column to be treated as non-null.
- `name: Type` — force the column's Rust type.
- `name: _` — infer the column's Rust type from the corresponding field of
  the record struct (used for aggregates whose type the planner can't prove).

The override is part of the column's alias, so it appears in the query
string and therefore in the `.sqlx/` metadata. The macro strips it before
matching the column to a struct field.

## Dynamic queries

`GET /api/v1/positions` (`list_positions`) takes optional `status` and
`strategy_id` filters. Instead of branching into several static queries, it
uses one query whose filters are `(? IS NULL OR status = ?)` — a `NULL`
argument disables that filter. This keeps a single checked statement for all
four filter combinations. The optional-filter behaviour is covered by
`tests/positions_test.rs` (filtering by status, by strategy, and the
unfiltered case).

## Test fixtures

The `#[cfg(test)]` modules and `tests/*_test.rs` build throwaway rows with
the runtime `sqlx::query()` functions on purpose: test setup benefits from
runtime flexibility (varying column lists, hand-built rows), and these
queries are not on any production path. Everything the application actually
runs at request time is behind a checked macro.
