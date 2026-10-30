# store

An **adaptor-agnostic database storage library** for
[Brood](https://github.com/broodlang/brood) — like Ecto, a bit better. A query is written
with `query/from`, in Brood, and is plain data underneath: compiled to parameterized SQL,
printable, and built on with ordinary functions; `schema`, `changeset`, and `repo` build on
top. The actual database I/O is done by a swappable **adaptor** that implements a small
contract, so the library depends on no specific driver — e.g.
[`store-postgres`](https://github.com/broodlang/store-postgres).

## Usage

```brood fragment
;; `u.name` is a column of the table `u` names; every other symbol or call is a Brood value,
;; bound as a parameter
(query/from u :users
  :where (ilike u.name (str "%" name "%"))
  :order-by (desc u.inserted_at)
  :limit 10)

;; a repo runs them over an adaptor (see store-postgres)
(let (users (repo/all my-repo the-query)
      email (repo/value my-repo (query/from u :users :where (= u.id id) :select u.email)))
  (repo/insert my-repo :users {:name "Ada" :email "ada@example.com"}))
```

Joins, aggregates and subqueries read the same way:

```brood fragment
(query/from o :organisations
  :join m :members (= m.organisation_id o.id)
  :group-by o.name
  :select {:name o.name :members (count)})

(query/from p :projects
  :where (not (exists (from r :releases :where (= r.project_id p.id)))))
```

The builders make the same maps a step at a time, for a query put together in parts, and
upserts and atomic increments are data too — no SQL strings:

```brood fragment
(-> (query/table :organisations :o)
    (query/select [:o.name [:as [:count] :members]])
    (query/join :members :m [:= :m.organisation_id [:col :o.id]])
    (query/group-by [:o.name]))

(repo/update my-repo :releases {:downloads [:+ :downloads 1]} [:= :id id] {:returning :none})
(repo/insert my-repo :counts {:k "x" :n 1}
  {:on-conflict {:target [:k] :do {:n [:+ :counts.n [:excluded :n]]}}})
```

Subqueries, lateral joins, CTEs (recursive too), unions, windows, `CASE`, JSON paths and a
parameterised `[:fragment …]` escape hatch are data as well, and a write can take the same
query a read does:

```brood fragment
(repo/fetch-by my-repo :users {:email email})          ; nil, the row, or an error if two match

(repo/update-all my-repo
  (-> (query/table :members :m)
      (query/join :organisations :o [:= :o.id [:col :m.organisation_id]])
      (query/where {:o.name "acme"}))
  {:role "admin"})
```

A query that cannot mean what it says is an error when it compiles, not a query that
quietly matches nothing: `[:= :token nil]` (nothing equals NULL; write `[:is-nil :token]`),
a misspelt key (`:wher`), an unknown order direction. [docs/compared-to-ecto.md](docs/compared-to-ecto.md)
puts every form next to its Ecto equivalent.

**Changesets** cast form params to the schema's types, validate, and turn a database
constraint into a field error:

```brood fragment
(let (cs (-> (changeset/cast {} params [:email :age] users-schema)
             (changeset/validate-required [:email])
             (changeset/validate-number :age {:greater-than-or-equal-to 0})
             (changeset/unique-constraint :email {:name "users_email_idx"})))
  (match (repo/insert-changeset my-repo cs)
    ([:ok user] …)
    ([:error cs] (changeset/error-messages cs))))   ; {:email ["has already been taken"]}
```

**Migrations** are versioned data, applied once under an advisory lock and recorded in
`schema_migrations`; `migration/check` proves (with no database) that they build the
app's schemas, and `migration/generate` writes the one that closes a gap:

```brood fragment
(def all
  [{:version 20261018090000
    :name "users"
    :up [[:create-table :users {:primary-key :id :fields {:id :serial :email [:text {:null false}]}}]
         [:create-index :users {:columns [:email] :unique true :name "users_email_idx"}]]}])

(migration/migrate my-repo all)          ; → the migrations applied
(migration/rollback my-repo all)         ; undo the newest (steps reverse themselves)
(migration/check all [users-schema])     ; → [] when they agree
```

A schema's table constraints are data too — `:constraints [{:unique [:name]} {:check
[:<> [:is-nil :name] [:is-nil :user_id]]}]`, named the way Postgres names them
(`<table>_name_key`, `<table>_check`) — and `:add-constraint` / `:drop-constraint` change
them in a migration.

Add it to a project with:

```bash
nest add store :version "^0.6.0"
```

## Publishing

Releases go to [hive](https://github.com/broodlang/hive), the Brood package
registry at <https://brood.fly.dev>.

**One-time setup** — register and mint an API token:

1. Create an account at <https://brood.fly.dev/register>.
2. Mint an API token on your <https://brood.fly.dev/settings> page (it's shown
   once), then expose it to `nest`:

   ```bash
   export HIVE_TOKEN=<your token>
   # or, persistently, add to ~/.config/brood/config.blsp:  :registry-token "<your token>"
   ```

**Each release:**

1. Bump `:version` in `project.blsp` — releases are **immutable**, so a version
   can never be republished.
2. Confirm the tests pass:

   ```bash
   nest test
   ```

3. Publish:

   ```bash
   nest publish
   ```

`nest publish` builds a source tarball (excluding `_deps/`, `tests/`, `.git/`,
and the lock file), records its sha256, and POSTs it to the registry. Only
`:version` (registry) dependencies are recorded in the published metadata. Docs
build automatically and appear at `https://brood.fly.dev/packages/store`.

## License

AGPL-3.0-only.
