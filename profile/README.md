<p align="center">
  <h1 align="center">JSORM</h1>
  <p align="center"><strong>JSON-first TypeScript ORM</strong> — schema-driven, deterministic AST compilation, multi-runtime.</p>
</p>

<p align="center">
  <a href="https://github.com/jsorm/monorepo/actions"><img src="https://github.com/jsorm/monorepo/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
  <a href="https://www.npmjs.com/org/jsorm"><img src="https://img.shields.io/npm/v/@jsorm/core" alt="npm"></a>
  <a href="https://github.com/jsorm/monorepo/blob/main/LICENSE"><img src="https://img.shields.io/github/license/jsorm/monorepo" alt="license"></a>
</p>

---

Define models once. Get full type inference — no decorators, no code generation.

```ts
import { createJsorm, defineModel, t } from "@jsorm/core";

const User = defineModel("users", {
  id: t.number().primary(),
  name: t.string(),
  email: t.string().optional(),
  active: t.boolean().default(true),
});

const db = await createJsorm().init();

const users = await db.get(User, {
  select: { name: true, email: true },
  where: { active: { eq: true } },
});
// users: Array<{ name: string; email?: string }>  ← fully inferred
```

---

### Why JSORM

| | |
|---|---|
| **JSON-first queries** | Reads and writes described with plain objects, not builder chains |
| **Full type inference** | `InferModel`, `InferInput`, `InferRow` — all derived from `defineModel` |
| **Deterministic AST** | Every query compiles through a versioned, inspectable AST contract |
| **Multi-runtime** | Node (Rust engine), Bun (`Bun.SQL`), Edge (`@jsorm/fetch`) |
| **Multi-database** | PostgreSQL, MySQL, SQLite — switch with `db.use("name")` |
| **Optional migrations** | Connect to existing databases immediately, opt into migrations later |

---

### Packages

| Package | Purpose |
|---|---|
| `@jsorm/core` | Models, queries, config, migration primitives |
| `@jsorm/node` | CLI (`jsorm init`, `jsorm deploy`) + Node wrapper |
| `@jsorm/runtime` | Runtime-safe imports for edge/fetch-only code |
| `@jsorm/fetch` | HTTP transport for edge runtimes |
| `@jsorm/pg` | PostgreSQL adapter |
| `@jsorm/mysql` | MySQL adapter |
| `@jsorm/sqlite` | SQLite adapter |

Core has zero database driver dependencies. Install only what you use.

---

### Quick Install

#### Bootstrap

```bash
pnpm dlx @jsorm/node init
```

`--yes` skips prompts. `--adapter` picks postgres, mysql, or sqlite. A `--no-migrations` flag is also available. Non-interactive bootstrap generates a project in one command:

```bash
pnpm dlx @jsorm/node init --yes --adapter postgres
```

#### Install directly

```bash
# PostgreSQL
pnpm add @jsorm/core @jsorm/pg pg
```
```bash
# MySQL
pnpm add @jsorm/core @jsorm/mysql mysql2
```
```bash
# SQLite
pnpm add @jsorm/core @jsorm/sqlite better-sqlite3
```
```bash
# Bun + SQLite
bun add @jsorm/core @jsorm/sqlite
```

Add the CLI to your `package.json` scripts so you can run `pnpm jsorm`:

```json
{
  "scripts": {
    "jsorm": "jsorm"
  },
  "devDependencies": {
    "@jsorm/node": "latest"
  }
}
```

---

### CLI

```bash
jsorm init                  # Bootstrap project
jsorm deploy                # Production-safe migration deploy
jsorm db:check              # Verify connectivity
jsorm migrate               # Run pending migrations
jsorm migrate:generate      # Diff models → reviewed migration file
```

---

### Runtime Support

| Runtime | Database | Verified |
|---|---|---|
| Node.js 22+ | PostgreSQL, MySQL, SQLite | Ubuntu + Windows CI, 337 tests |
| Bun | SQLite (via `Bun.SQL`) | Bun smoke CI |
| Edge | Any (via `@jsorm/fetch`) | Unit-tested transport |

---
## Documentation

| Section | Topics |
|---|---|
| [Getting Started](https://github.com/jsorm/docs/blob/main/getting-started/installation.md) | Install, quick start, first model, first query |
| [Runtime](https://github.com/jsorm/docs/blob/main/runtime/node-runtime.md) | Node, Bun, Edge, runtime profiles, workers |
| [Schema](https://github.com/jsorm/docs/blob/main/schema/define-model.md) | Models, field types, relations, indexes, naming |
| [Migrations](https://github.com/jsorm/docs/blob/main/migrations/migration-overview.md) | Generate, migrate, snapshots, diffing |
| [Adapters](https://github.com/jsorm/docs/blob/main/adapters/pg.md) | PG, MySQL, SQLite, fetch, capabilities |
| [CLI](https://github.com/jsorm/docs/blob/main/cli/cli-overview.md) | Init, deploy, db:check, migrate commands |
| [Deployment](https://github.com/jsorm/docs/blob/main/deployment/overview.md) | Cloud, Docker, environment, CI/CD |
| [Concepts](https://github.com/jsorm/docs/blob/main/concepts/architecture.md) | Runtime architecture, relations, transactions, errors |
| [Advanced](https://github.com/jsorm/docs/blob/main/advanced/custom-adapters.md) | Custom adapters, custom queries, multi-tenancy, testing |
| [Examples](https://github.com/jsorm/docs/blob/main/examples/) | Blog platform, e-commerce, multi-tenant patterns |

Full tree: [docs/SUMMARY.md](https://github.com/jsorm/docs/blob/main/SUMMARY.md)

---
<p align="center">MIT · v0.2.0 · Production-ready beta</p>
