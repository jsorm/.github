<p align="center">
  <h1 align="center">JSORM</h1>
  <p align="center"><strong>A modern, fully-typed, edge-compatible TypeScript ORM, JSON-first.</strong></p>
  <p align="center"> schema-driven, deterministic AST compilation, multi-runtime.</p>
</p>

<p align="center">
  <a href="https://github.com/jsorm/monorepo/actions"><img src="https://github.com/jsorm/monorepo/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
  <a href="https://www.npmjs.com/org/jsorm"><img src="https://img.shields.io/npm/v/@jsorm/core" alt="npm"></a>
  <a href="https://github.com/jsorm/monorepo/blob/main/LICENSE"><img src="https://img.shields.io/github/license/jsorm/monorepo" alt="license"></a>
</p>

---

```text
Model → AST → Provider → Database → Typed result
```

---

<!--toc:start-->
- [JSORM](#jsorm)
  - [Bootstrap](#bootstrap)
  - [Define a model](#define-a-model)
  - [Query API](#query-api)
    - [Read](#read)
    - [Create](#create)
    - [Update](#update)
    - [Delete](#delete)
    - [Many-to-many attach / detach / toggle](#many-to-many-attach--detach--toggle)
    - [Raw SQL](#raw-sql)
    - [Compile without running](#compile-without-running)
  - [Multi-database](#multi-database)
  - [Schema workflow](#schema-workflow)
  - [CLI](#cli)
  - [Packages](#packages)
  - [Architecture](#architecture)
  - [Documentation](#documentation)
  - [License](#license)
<!--toc:end-->

---

## Bootstrap

```bash
# Interactive project setup (wizard)
pnpm dlx @jsorm/cli configure
```

```bash
# Packages
pnpm add @jsorm/client @jsorm/core
pnpm add -D @jsorm/cli
```

```bash
# Generate types and runtime files
pnpm jsorm gen
```

---

## Define a model

```typescript
import { defineModel, t, r } from "@jsorm/core";

export const User = defineModel("users", {
  db: "main",
  fields: {
    id:        t.uuid().primary().autoCreate(),
    name:      t.string(100),
    email:     t.string(255).unique(),
    active:    t.boolean().default(true),
    roleId:    t.uuid().nullable(),
    createdAt: t.dateTime().autoCreate(),
    updatedAt: t.dateTime().autoUpdate(),
    startsOn:  t.date().nullable(),
    status:    t.enum(["draft", "published"]),
  },
  indexes: [
    ["status", "updatedAt"],
    { columns: ["name"], unique: true, where: "active" },
    { columns: ["createdAt"], name: "idx_users_created" },
  ],
  checks: [{ name: "users_name_check", expression: "length(name) > 0" }],
  relations: {
    posts: r.hasMany("posts"),
    role: r.belongsTo("roles", { localKey: "roleId" }),
    roles: r.manyToMany("roles", {
      through: "user_roles",
      columns: { grantedAt: t.dateTime().autoCreate() },
    }),
  },
});
```

**Field types:** `t.uuid()`, `t.string(n)`, `t.boolean()`, `t.integer()`, `t.decimal(p, s)`, `t.text()`, `t.dateTime()`, `t.date()`, `t.json()`, `t.enum(values)`.

**Modifiers:** `primary()`, `autoCreate()`, `autoUpdate()`, `default(value)`, `unique()`, `index()`, `nullable()`, `hidden()`, `renamedFrom("oldName")`.

- `t.date()` is a calendar date (`DATE`), typed as a `YYYY-MM-DD` string; `t.dateTime()` is a timestamp.
- `t.enum()` becomes a native `ENUM` type on PostgreSQL and `TEXT` plus a `CHECK` on SQLite; the generated type is the union of its values.
- `indexes[].columns` names **fields**; the engine maps them to columns with `naming.fields`, and the default name uses those columns: `["status", "updatedAt"]` becomes `idx_users_status_updated_at` under `snake_case`. A `where` predicate and a `checks` expression are raw SQL, so they use **column** names, quoted when the column keeps camelCase (default naming: `"deletedAt"`).
- `.index()` creates one single-column index named `idx_<table>_<column>`. `indexes` is for what a field modifier cannot express: several columns (`["status", "updatedAt"]`), a custom `name`, a `unique` index or a partial `where`. The shorthand needs at least two columns; a single column uses `.index()` or `{ columns: ["a"] }`.
- `.unique()` on a field creates a UNIQUE constraint (`<table>_<column>_key`); `{ columns: [...], unique: true }` creates a unique index. Both enforce uniqueness, so declare one of them, not both.
- `jsorm gen` and `jsorm db:check` warn when a `.index()` repeats the leading column of a composite index declared in `indexes`: the composite already serves those lookups. The warning never blocks, and with `--json` it goes to stderr.
- `.index()` and a model-level index that land on the same name must declare the same thing: two different definitions are rejected instead of silently keeping the first. Two `.index()` fields create two single-column indexes, never a composite.
- A `belongsTo` `onDelete` needs its key to be a field of the model, and `setNull` needs that field to be `.nullable()`; a nullable key defaults to `setNull` and a required one to `restrict`.
- A composite primary key declares `primary()` on each key field.
- `.renamedFrom()` and the model-level `renamedFrom` are planning hints: `db:sync` renames the column or table instead of dropping the data.

Full reference: [docs/en/api/API.md](docs/en/api/API.md).

---

## Query API

```typescript
import { jsorm } from "./models/jsorm.server.js";
```

### Read

```typescript
// First match
const { data: user } = await jsorm.users.first({
  select: { name: true, email: true },
  where: { email: "juan@test.com" },
});

// List with filters and a projection
const { data: users } = await jsorm.users.get({
  select: { id: true, name: true, roles: { name: true } },
  where: { active: true },
  orderBy: { createdAt: "desc" },
});
// users: { id: string; name: string; roles: { name: string }[] }[]

// Offset pagination
const { data: page, meta } = await jsorm.users.get({
  select: { id: true, name: true },
  pagination: { page: 1, perPage: 10 },
});
// meta: { page, perPage, total, lastPage, hasTruncated? }

// Cursor (keyset) pagination
const { data: cursor, meta: cursorMeta } = await jsorm.users.cursor({
  select: { id: true, name: true },
  orderBy: { id: "asc" },
  limit: 20,
});
// cursorMeta.cursor: { hasNext, next, previous }

// Count / exists
const { data: total } = await jsorm.users.count({ where: { active: true } });
const { data: exists } = await jsorm.users.exists({ where: { email: "juan@test.com" } });
```

Select supports aliases: `select: { name: { as: "displayName" } }`.

### Create

```typescript
const { data: user } = await jsorm.users.create({ name: "Juan", email: "juan@test.com" });

const { data: users } = await jsorm.users.createMany([{ name: "Ana" }, { name: "Carlos" }]);
```

`createMany` sends homogeneous rows as one multi-row `INSERT`; heterogeneous rows fall back to one statement per row inside a transaction.

### Update

```typescript
const { data: affected } = await jsorm.users.update({
  where: { id: user.id },
  data: { name: "Juan Updated" },
});
// affected: number of rows

const { data: updated } = await jsorm.users.updateMany([
  { id: "1", name: "A" },
  { id: "2", name: "B" },
]);
```

### Delete

```typescript
await jsorm.users.delete({ where: { id: user.id } });
await jsorm.users.deleteMany({ where: { active: false } });
```

`where` is required on both; an empty or missing `where` throws instead of matching every row.

### Many-to-many attach / detach / toggle

```typescript
await jsorm.users.attach(userId, "roles", [roleId1, roleId2]);
await jsorm.users.detach(userId, "roles", [roleId2]);
await jsorm.users.toggle(userId, "roles", [roleId3]);
```

### Raw SQL

```typescript
const { data: rows } = await jsorm.raw({ sql: "SELECT 1" });

// Pass parameters instead of interpolating values
const { data: found } = await jsorm.raw({
  sql: "SELECT id, email FROM users WHERE active = ?",
  params: [true],
});
```

`raw()` returns the statement's rows (`[]` for a write). `?` placeholders work on both SQL providers (PostgreSQL receives `$n`). A guardrail blocks `DROP DATABASE`, `DROP SCHEMA`, `TRUNCATE`, `GRANT`, `REVOKE`, `PRAGMA`, `CREATE USER` and `ALTER USER` unless you pass `allowUnsafe: true`: it prevents accidents, not injection, so always bind values with `params`.

### Compile without running

```typescript
const { sql, params } = jsorm.compile(queryAST);
// sql: "SELECT * FROM users WHERE active = ?"
// params: [true]
```

---

## Multi-database

A model belongs to exactly one database. Non-main databases are reached through their namespace:

```typescript
import { jsorm } from "./models/jsorm.server.js";

// Main database: direct access
const { data: users } = await jsorm.users.get();

// Other databases: namespaced access
const { data: sessions } = await jsorm.cache.sessions.get();
const { data: events } = await jsorm.analytics.event.get();
```

Each database declares its provider in the third argument of `createJsorm`, keyed by the database name; `ctx` already carries the resolved `options`:

```typescript
import { schema } from "./jsorm.schema.js";

const jsorm = createJsorm<DB>(config, models, {
  main: (ctx) => createPostgresNodeProvider(ctx.models, { ...ctx.options, schema: schema["main"] }),
  cache: (ctx) => createSqliteNodeProvider(ctx.models, { ...ctx.options, schema: schema["cache"] }),
});
```

There are no relations or joins across databases: composition happens in the application. `access` only accepts `"server"` or `"client"` — `"both"` is a capability in the provider catalog, never a configuration value.

More: [docs/en/packages/client.md](docs/en/packages/client.md).

---

## Schema workflow

`jsorm gen` writes the types and `jsorm.schema.ts`, but the schema of a SQL database changes through
the `db:*` cycle. There are no migration files: every plan compares the models, the head snapshot of
the history (`_jsorm_schema_history`) and the live database.

```text
db:check → db:sync (or db:plan + db:apply) → db:pull → db:rollback
```

| Step | Command | What it does |
| --- | --- | --- |
| Inspect | `pnpm jsorm db:check` | Reports every change with its side and risk. Writes nothing |
| Apply | `pnpm jsorm db:sync` | Plans, confirms by risk class, applies and records a snapshot |
| Review | `pnpm jsorm db:plan --out release.plan.json` | Writes a self-contained plan; `db:apply` runs it as written, with its data guards |
| Adopt | `pnpm jsorm db:pull` | Writes database drift into the models, or creates new model files |
| Undo | `pnpm jsorm db:rollback` | Returns to the previous snapshot; irreversible steps need `--accept-irreversible` and `--backup-confirmed` |
| Audit | `pnpm jsorm db:history` / `pnpm jsorm db:status` | Lists snapshots and shows the head, chain validity and pending summary |

Exit codes for CI: `0` in sync, `1` only safe changes pending, `2` something blocks the sync (data loss, drift or conflict), `3` the command could not run. A database with `_jsorm_schema_history` is managed: the provider's startup sync runs nothing and the changes go through `db:check` / `db:sync` only.

Full reference: [docs/en/cli/SCHEMA.md](docs/en/cli/SCHEMA.md).

---

## CLI

| Command | Description |
| --- | --- |
| `jsorm configure` | Interactive setup wizard |
| `jsorm gen` | Generates types and runtime files (`--dry-run` previews) |
| `jsorm watch` | Regenerates on model changes |
| `jsorm db:check` | Reports pending schema changes |
| `jsorm db:plan` | Writes a self-contained plan |
| `jsorm db:apply` | Runs a plan file |
| `jsorm db:sync` | Plans, confirms and applies |
| `jsorm db:pull` | Writes database drift into the models |
| `jsorm db:rollback` | Returns to a previous snapshot |
| `jsorm db:history` | Lists the snapshots |
| `jsorm db:status` | Shows the head and the pending summary |
| `jsorm skills` | Lists, prints or installs the Agent Skill for coding assistants |

`configure` flags:

| Flag | Values |
| --- | --- |
| `--config` | `full`, `add`, `edit`, `reconfigure` |
| `--models` | Models path (default: `src/models`) |
| `--access` | `server`, `client` |
| `--provider` | `sqlite-node`, `postgres-node`, `indexeddb` |
| `--naming-fields` | `camelCase`, `snake_case` |
| `--usermodel` | `yes`, `no` |
| `--install` | `yes`, `no` |
| `--db-mode` | `file`, `memory` (SQLite only) |
| `--db-path` | SQLite file path |

`jsorm skills install` writes the embedded Agent Skill into `.agents/skills/jsorm` (and `.claude/skills/jsorm` when `.claude/` exists) so a coding agent knows the commands, the model DSL and the schema cycle.

Full reference: [docs/en/cli/COMMANDS.md](docs/en/cli/COMMANDS.md).

---

## Packages

| Package | What it does | Runtime |
| --- | --- | --- |
| `@jsorm/core` | Types, `defineModel()`, `t.*`, `r.*`, `defineConfig()`, `env()` | Edge, Browser, Node |
| `@jsorm/client` | `createJsorm()`, proxies, `TableMethods`, `ProviderManager` | Edge, Browser, Node |
| `@jsorm/compiler` | AST → SQL for the SQL providers (dialects, cursors, result mapping); application code never imports it | Node 22+ |
| `@jsorm/cli` | CLI binary (Rust engine): `configure`, `gen`, `watch`, `skills` and the `db:*` cycle | Node 22+ |
| `@jsorm/provider-sqlite-node` | SQLite provider (better-sqlite3) | Node 22+ |
| `@jsorm/provider-pg-node` | PostgreSQL provider (pg) | Node 22+ |
| `@jsorm/provider-indexeddb` | IndexedDB provider | Browser |

Every package is ESM-only (`.mjs` + `.d.ts`). Details: [docs/en/packages/](docs/en/packages/).

---

## Architecture

```text
Your code → Client (Proxy → TableMethods → AST) → Provider (Compiler → SQL) → Database
```

- **Core** defines models and types. It never knows about SQL or Node.
- **Client** builds the AST that describes a query. Edge-compatible: no `fs`, `path` or `process`.
- **Provider** turns the AST into SQL and runs it. Each provider is self-contained: driver, compiler, connection manager and schema sync.
- **CLI** generates the types and runtime files. Rust engine with tree-sitter; it is not part of the production bundle.

Full flow:

```text
defineModel() → CLI (tree-sitter parser) → .jsorm/types.ts + jsorm.server.ts + jsorm.schema.ts
                                        ↓
jsorm.users.get({ select, where }) → Proxy → TableMethods → AST builder → QueryAST
                                        ↓
                            ProviderManager (routes by database name)
                                        ↓
                            Provider.execute(QueryAST) → compiler → SQL
                                        ↓
                            driver.query(sql, params) → { data: T[] }
```

More: [docs/en/architecture/ARCHITECTURE_AND_RULES.md](docs/en/architecture/ARCHITECTURE_AND_RULES.md) and [ARCHITECTURE.md](ARCHITECTURE.md).

---

## Documentation

The same documentation ships in English and Spanish: read it on the website or in this repository.

| English | Español |
| --- | --- |
| [Documentation](https://jsorm.dev/docs/getting-started/installation) | [Documentación](https://jsorm.dev/es/docs/getting-started/installation) |
| [docs/en](docs/en/README.md) | [docs/es](docs/es/README.md) |

| Guide | Link |
| --- | --- |
| Full API | [API reference](docs/en/api/API.md) |
| API rules and patterns | [API structure](docs/en/api/API_STRUCTURE.md) |
| Architecture and rules | [Architecture](docs/en/architecture/ARCHITECTURE_AND_RULES.md) |
| Design decisions | [Design](docs/en/architecture/DESIGN.md) |
| API principles | [Principles](docs/en/API_PRINCIPLES.md) |
| Conventions | [Conventions](docs/en/conventions/CONVENTIONS.md) |
| CLI | [Commands](docs/en/cli/COMMANDS.md) · [Configure](docs/en/cli/CONFIGURE.md) · [Schema](docs/en/cli/SCHEMA.md) |
| Packages | [docs/en/packages](docs/en/packages/) |
| Coding rules | [RULES.md](RULES.md) |
| For AI agents | [llms.txt](https://jsorm.dev/llms.txt) · [llms-full.txt](https://jsorm.dev/llms-full.txt) |

---

## License

**MPL-2.0** — you can use JSORM in proprietary software without publishing your own code, but you must publish the changes you make to the ORM's `.ts` files.

[Full license text](LICENSE) — [Security policy](security.md)


