# store

An **adaptor-agnostic database storage library** for
[Brood](https://broodlang.org) — like Ecto, a bit better. Queries are plain data
(no macro DSL, no pin operator), composed with ordinary functions and compiled
to parameterized SQL; `schema`, `changeset`, and `repo` build on top. The actual
database I/O is done by a swappable **adaptor** that implements a small contract,
so the library depends on no specific driver — e.g.
[`store-postgres`](https://github.com/broodlang/store-postgres).

## Usage

```brood
(:alias query)
(:alias repo)

;; queries are data, composed with plain functions
(-> (query/from :users)
    (query/where [:ilike :name "%ada%"])
    (query/order-by :inserted_at :desc)
    (query/limit 10))

;; a repo runs them over an adaptor (see store-postgres)
(let (users (repo/all my-repo the-query))
  (repo/insert my-repo :users {:name "Ada" :email "ada@example.com"}))
```

Joins, aggregates, upserts and atomic increments are data too — no SQL strings:

```brood
(-> (query/from :organisations :o)
    (query/select [:o.name [:as [:count] :members]])
    (query/join :members :m [:= :m.organisation_id [:col :o.id]])
    (query/group-by [:o.name]))

(repo/update my-repo :releases {:downloads [:+ :downloads 1]} [:= :id id] {:returning :none})
(repo/insert my-repo :counts {:k "x" :n 1}
  {:on-conflict {:target [:k] :do {:n [:+ :counts.n [:excluded :n]]}}})
```

**Changesets** cast form params to the schema's types, validate, and turn a database
constraint into a field error:

```brood
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

```brood
(def all
  [{:version 20261018090000
    :name "users"
    :up [[:create-table :users {:primary-key :id :fields {:id :serial :email [:text {:null false}]}}]
         [:create-index :users {:columns [:email] :unique true :name "users_email_idx"}]]}])

(migration/migrate my-repo all)          ; → the migrations applied
(migration/rollback my-repo all)         ; undo the newest (steps reverse themselves)
(migration/check all [users-schema])     ; → [] when they agree
```

Add it to a project with:

```bash
nest add store :version "^0.5.0"
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
