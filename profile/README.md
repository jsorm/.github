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

Define models once. Run in any environment. No `any`, no heavy runtime.

```text
Defined model → AST → Provider → Database → Typed result
```

---

## Bootstrap

```bash
# Interactive setup (wizard)
pnpm dlx @jsorm/cli configure
```
## Install packages manually
```bash
pnpm add @jsorm/client @jsorm/core
pnpm add -D @jsorm/cli
```

## Generate types and runtime
```bash
pnpm jsorm gen
```
---

## Defining a model

```typescript
import { defineModel, t, r } from "@jsorm/core";

export const User = defineModel("users", {
  db: "main",
  fields: {
    id:      t.uuid().primary().autoCreate(),
    name:    t.string(100),
    email:   t.string(255).unique(),
    active:  t.boolean().default(true),
    createdAt: t.dateTime().autoCreate(),
    updatedAt: t.dateTime().autoUpdate(),
  },
  relations: {
    posts: r.hasMany("post"),
    role: r.belongsTo("role"),
    roles: r.manyToMany("role", {
      through: "user_roles",
      extra: { role: t.string(50) },
    }),
  },
});
```

**Available types:** `t.uuid()`, `t.string(n)`, `t.boolean()`, `t.integer()`, `t.decimal(p, s)`, `t.text()`, `t.dateTime()`, `t.json()`, `t.enum(values)`.

**Modifiers:** `primary()`, `autoCreate()`, `autoUpdate()`, `default(val)`, `unique()`, `nullable()`, `hidden()`.

---

## Using the API

```typescript
import { jsorm } from "./models/jsorm.server.js";
```

### Reading

```typescript
// First record
const { data: user } = await jsorm.users.first({
  select: { name: true, email: true },
  where: { email: "juan@test.com" },
});

// List with filters and select
const { data: users } = await jsorm.users.get({
  select: { id: true, name: true, roles: { name: true } },
  where: { active: true },
  orderBy: { createdAt: "desc" },
});
// ← users: { id: string; name: string; roles: { name: string }[] }[]

// Offset-based pagination
const { data: page, meta } = await jsorm.users.get({
  select: { id: true, name: true },
  pagination: { page: 1, perPage: 10 },
});
// ← meta: { page, perPage, total, lastPage, hasTruncated? }

// Cursor-based pagination
const { data: cursor, meta: cursorMeta } = await jsorm.users.cursor({
  select: { id: true, name: true },
  orderBy: { id: "asc" },
  limit: 20,
});
// ← cursorMeta.cursor.hasNext, cursorMeta.cursor.next, cursorMeta.cursor.previous

// Count / Exists
const { data: total } = await jsorm.users.count({ where: { active: true } });
const { data: exists } = await jsorm.users.exists({ where: { email: "juan@test.com" } });
```

**Select with alias:**

```typescript
await jsorm.users.get({ select: { name: { as: "Nombre" } } });
```

### Creating

```typescript
// Single
const { data: user } = await jsorm.users.create({ name: "Juan", email: "juan@test.com" });

// Bulk
const { data: users } = await jsorm.users.createMany([{ name: "Ana" }, { name: "Carlos" }]);
```

### Updating

```typescript
// Single
const { data: affected } = await jsorm.users.update({
  where: { id: user.id },
  data: { name: "Juan Updated" },
});
// ← affected: number of rows affected

// Bulk
const { data: affected } = await jsorm.users.updateMany([{ id: "1", name: "A" }, { id: "2", name: "B" }]);
```

### Deleting

```typescript
// Single — where required
await jsorm.users.delete({ where: { id: user.id } });

// Bulk — where required; throws if no where clause is provided, to prevent accidental mass deletes
await jsorm.users.deleteMany({ where: { active: false } });
```

### Many-to-Many attach / detach / toggle

```typescript
await jsorm.users.attach(userId, "roles", [roleId1, roleId2]);
await jsorm.users.detach(userId, "roles", [roleId2]);
await jsorm.users.toggle(userId, "roles", [roleId3]);
```

### Raw SQL escape hatch

```typescript
const { data: result } = await jsorm.raw({ sql: "SELECT 1" });
```

### Compile without executing (debug)

```typescript
const { sql, params } = jsorm.compile(queryAST);
// sql: "SELECT * FROM users WHERE active = ?"
// params: [true]
```

---

## Multi-database

Models assigned to a non-main database are accessed via a namespace:

```typescript
import { jsorm } from "./models/jsorm.db.js";

// Main database — direct access
const { data: users } = await jsorm.users.get();

// Non-main database — namespaced access
const { data: sessions } = await jsorm.cache.sessions.get();
const { data: analytics } = await jsorm.analytics.event.get();
```

---

## CLI commands

| Command | Description |
|---|---|
| `pnpm jsorm configure` | Interactive wizard |
| `pnpm jsorm gen` | Generates types + runtime |
| `pnpm jsorm watch` | Automatically regenerates when models change |
| `pnpm jsorm gen --dry-run` | Preview without writing files |

### `configure` flags

| Flag | Values |
|---|---|
| `--config` | `full`, `add`, `edit`, `reconfigure` |
| `--access` | `server`, `client` |
| `--provider` | `sqlite-node`, `postgres-node`, `indexeddb` |
| `--naming-fields` | `camelCase`, `snake_case` |
| `--usermodel` | `yes`, `no` |
| `--install` | `yes`, `no` |
| `--db-mode` | `file`, `memory` |
| `--db-path` | Database path |
| `--models` | Custom path for models |

---

## Packages

| Package | What it does | Runtime |
|---|---|---|
| `@jsorm/core` | Types, `defineModel()`, `t.*`, `r.*`, `defineConfig()`, `env()` | Edge, Browser, Node |
| `@jsorm/client` | `createJsorm()`, Proxy, TableMethods, ProviderManager | Edge, Browser, Node |
| `@jsorm/cli` | CLI binary (Rust engine): `configure`, `gen`, `watch` | Node 22+ |
| `@jsorm/compiler` | AST to SQL Compiler | Node 22+ |
| `@jsorm/provider-sqlite-node` | SQLite provider (better-sqlite3) | Node 22+ |
| `@jsorm/provider-pg-node` | PostgreSQL provider (pg) | Node 22+ |
| `@jsorm/provider-indexeddb` | IndexedDB provider (browser) | Browser |

All packages are ESM-only (`.mjs` + `.d.ts`).

---

## Architecture

```
Your code → Client (Proxy → TableMethods → AST) → Provider (Compiler → SQL) → Database
```

- **Core** defines models and types. It never knows about SQL or Node.
- **Client** builds the AST that describes queries. Edge-compatible (no `fs`, `path`, `process`).
- **Provider** turns the AST into SQL and executes it. Each provider is self-contained (driver + compiler + connection manager + schema sync).
- **CLI** generates the types and runtime code. Rust engine using tree-sitter. Not part of the production bundle.

**Full flow:**

```
defineModel() → CLI (tree-sitter parser) → .jsorm/types.ts + runtime files
                                        ↓
jsorm.users.get({ select, where }) → Proxy → TableMethods → AST builder → QueryAST
                                        ↓
                            ProviderManager (routes by db name)
                                        ↓
                            Provider.execute(QueryAST) → compiler → SQL
                                        ↓
                            driver.query(sql, params) → { data: T[] }
```

---

## Documentation

| Section | Content |
|---|---|
| [Quick start](docs/en/README.md) | Full step-by-step guide |
| [Full API](docs/en/api/API.md) | Every method documented |
| [API structure](docs/en/api/API_STRUCTURE.md) | Rules and patterns of the API |
| [Architecture](docs/en/architecture/ARCHITECTURE_AND_RULES.md) | Layers, rules, design |
| [Design decisions](docs/en/architecture/DESIGN.md) | Why it was built this way |
| [Principles](docs/en/API_PRINCIPLES.md) | Design philosophy (11 immutable rules) |
| [Conventions](docs/en/conventions/CONVENTIONS.md) | Naming: models, tables, relations |
| [CLI](docs/en/cli/COMMANDS.md) | Commands and flags |
| [Configure](docs/en/cli/CONFIGURE.md) | Configuration wizard |
| [Packages](docs/en/packages/) | Details of each package |
| [RULES.md](RULES.md) | SOLID, DRY, strict rules |

---

## License

**MPL-2.0** — you can use JSORM in proprietary software with no obligation to open-source your own code, but you must publish any changes you make to the ORM's `.ts` files.

[Full license text](LICENSE) — [Security policy](security.md)

