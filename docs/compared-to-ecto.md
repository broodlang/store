# store compared to Ecto

store's queries are plain data — maps and vectors, built with ordinary functions — where
Ecto's are a macro DSL over Elixir AST. This page puts the two side by side: what each
query looks like, what each catches, and where they differ on purpose.

## The same queries

| | Ecto | store |
| --- | --- | --- |
| Filter | `where(u, [u], u.email == ^email)` | `(where q {:email email})` / `[:= :email email]` |
| Several conditions | `where(u, email: ^e, active: true)` | `(where q {:email e :active true})` |
| A column against a column | `where(m, [m, o], m.org_id == o.id)` | `[:= :m.org_id [:col :o.id]]` |
| Built at runtime | `dynamic/2` and its own composition rules | it is a map: `cond`, `reduce`, `assoc`, `dissoc` |
| Remove a part | `exclude(q, :order_by)` | `(dissoc q :order)` |
| Inspect | `Ecto.Adapters.SQL.to_sql/3`, needs a repo | `(query/to-sql q)`, or print the map |
| Escape hatch | `fragment("lower(?)", u.name)` | `[:fragment "lower(?)" :name]` |
| Any function | `fragment("date_trunc('day', ?)", c.at)` | `[:call :date_trunc [:literal "day"] :at]` |
| Subquery in FROM / JOIN | `subquery(q)` | `(from q :alias)` / `(join q q2 :alias on)` |
| Scalar subquery | `subquery(q)` in `select` / `where` | the query map itself: `{:members count-query}` |
| Lateral join | `join: x in lateral(...)` (`inner_lateral_join`) | `lateral-join`, `left-lateral-join` |
| CTEs | `with_cte/3`, `recursive_ctes(true)` | `with-cte`, `with-recursive-cte` |
| Set operations | `union`, `union_all`, `except`, `intersect` (+ `_all`) | the same six |
| Window | `over(row_number(), partition_by: …, order_by: …)` | `[:over [:call :row_number] {:partition-by … :order-by …}]` |
| Aggregate filter | `filter(count(), c.ok)` | `[:filter [:count] {:ok true}]` |
| CASE | `fragment("CASE WHEN …")` | `[:case test value … :else value]` |
| JSON path | `json_extract_path(e.data, ["user", "name"])` | `[:->> :data "user" "name"]` |
| Type cast | `type(^x, :integer)` | `[:cast x :integer]` |
| DISTINCT ON | `distinct: [asc: c.project_id]` | `(distinct-on q [:project_id])` |
| Nulls in order | `asc_nulls_last: x` | `(order-by q :x :asc-nulls-last)` |
| Row lock | `lock: "FOR UPDATE SKIP LOCKED"` — a string | `(lock q :update :skip-locked)` — checked |
| Read a whole table | `Repo.all(User)` | `(repo/all repo :users)` / a schema |
| By primary key | `Repo.get(User, id)` | `(repo/fetch repo users-schema id)` |
| By fields | `Repo.get_by(User, email: e)` | `(repo/fetch-by repo :users {:email e})` |
| Update by query | `update_all(q, set: [...], inc: [...])` | `(repo/update-all repo q {:n [:+ :n 1]})` |
| Delete by query | `delete_all(q)` | `(repo/delete-all repo q)` |
| Insert from a query | `insert_all(Post, q)` | `(repo/insert-all repo :posts q)` |
| Upsert | `on_conflict: [set: …], conflict_target: …` | `{:on-conflict {:target … :do {…} :where …}}` |
| Computed RETURNING | `returning: [:id]` (fields only) | `:returning [:id [:as [:= :xmax 0] :inserted]]` |

## What each catches

Both refuse the mistakes that would otherwise run and return the wrong rows:

| Mistake | Ecto | store |
| --- | --- | --- |
| `x == nil` | compile error, or a runtime error for `^nil` | `[:= :x nil]` is an error: write `[:is-nil :x]` or `[:not-distinct-from :x v]` |
| A typo in a query part | compile error | `{:wher …}` is an error naming the keys a query has |
| A typo in an option | usually ignored | `{:returnin :none}` is an error |
| An unknown order direction | compile error | `:dsc` is an error |
| `update_all` with a `limit` | runtime error | `update-all` refuses `:limit`, `:order`, `:group` |
| A value mistaken for SQL | `^` marks values | a value is bound unless it is an expression; a keyword on the right side is an error, since it could be either |
| A non-unique `get_by` | `MultipleResultsError` | `fetch-by` raises the same way |
| A cast into SQL | type checked | a type name is words, a precision and `[]` only, so `[:cast x "text), (SELECT …"]` is an error |
| A fragment with the wrong hole count | compile error | an error naming the fragment |

Ecto catches these at compile time; store catches them when the query compiles to SQL.
That is earlier than the database, and covers a query built at runtime as well as one
written out.

## Where store is ahead

- **A query is a value.** Nothing about it is macro-time: a query passed in, stored,
  merged or generated is the same as one written out, and `dynamic/2` has no equivalent
  because it isn't needed.
- **Migrations come from the schema.** A schema's `:constraints` are data, named the way
  Postgres names them, and migrations add and drop them. `changeset/unique-constraint` maps
  a violation back to a field. Ecto has every piece of this — `create constraint` in a
  migration, `unique_constraint` in a changeset — but the schema doesn't describe the table,
  so nothing ties them together.
- **A row lock is checked data.** `(lock q :update :skip-locked)` refuses an unknown
  strength or wait. Ecto's `lock:` is a string that goes into the SQL as written.
- **Migrations are checked against the schema without a database.** `migration/check`
  fails until the migrations build exactly the schemas, and `migration/generate` writes
  the step that closes the gap.

## Where Ecto is ahead

- **Typed loading.** An Ecto schema turns a column into a `DateTime`, a `Decimal`, a map
  from JSON. store returns what the driver decodes, which already includes exact decimals
  and integers.
- **Join preloads.** Ecto can preload through a join it already made. store's `preload`
  runs one query per level, which also avoids N+1 queries.
- **`Repo.stream`.** store has no cursor-backed stream yet.
