# jsorm

**JSON-first TypeScript ORM** for schema-driven apps, deterministic AST compilation, and multi-runtime deployments.

Define models once. Keep full type inference without decorators or code generation.

> **Primary runtime:** Node.js 22+
> **Supported runtime paths:** Bun (`Bun.SQL`) • Edge runtime queries via `@jsorm/fetch`
> **Databases:** PostgreSQL • MySQL • SQLite

---

## Packages

| Package | Purpose |
|---|---|
| `@jsorm/core` | Models, queries, config, migration primitives, and Node-only runtime helpers |
| `@jsorm/runtime` | Runtime-safe app imports for edge or fetch-only code |
| `@jsorm/node` | Compatibility wrapper around `@jsorm/core/node` and home of the `jsorm` CLI |
| `@jsorm/fetch` | HTTP/fetch transport for edge runtimes |
| `@jsorm/pg` | PostgreSQL adapter |
| `@jsorm/mysql` | MySQL adapter |
| `@jsorm/sqlite` | SQLite adapter |

Core has zero database driver dependencies. Install only what you use.

---

## Install

```bash
# Node.js + PostgreSQL
pnpm add @jsorm/core @jsorm/pg pg
```
```bash
# Node.js + MySQL
pnpm add @jsorm/core @jsorm/mysql mysql2
```
```bash
# Node.js + SQLite
pnpm add @jsorm/core @jsorm/sqlite better-sqlite3
```
```bash
# Bun + SQLite
bun add @jsorm/core @jsorm/sqlite
```
```bash
# Edge runtime (queries via fetch)
pnpm add @jsorm/runtime @jsorm/fetch
pnpm add -D @jsorm/core @jsorm/pg pg
```

Bootstrap a project:
```bash
pnpm dlx @jsorm/node init

# Non-interactive bootstrap
pnpm dlx @jsorm/node init --yes --adapter postgres
```

## Verified Support Surface

- Node.js 22+ is the primary production path. CI runs the full workspace on Ubuntu and Windows with Node 22 and 24.
- Bun is supported for runtime queries through `Bun.SQL`. Repo CI currently verifies this through Bun SQLite smoke coverage plus adapter/package tests.
- Edge runtimes are supported for runtime queries through `@jsorm/runtime` + `@jsorm/fetch`. CLI and migration workflows remain Node-owned.
- Package tarball smoke tests and the SQLite example are part of the verified support story.

---

## Quick example

After running `jsorm init` and generating a `jsorm.config.*` file:

```ts
import { createJsorm, defineModel, t } from '@jsorm/core';

const User = defineModel('users', {
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
// users: Array<{ name: string; email?: string }>

await db.close();
```

---

## Why JSORM?

- **JSON-first queries** — describe reads and writes with plain objects, not builder chains
- **Full type inference** — `InferModel` and `InferInput` from `defineModel`, zero manual types
- **Deterministic AST** — every query compiles through a versioned, inspectable AST contract
- **Multi-runtime** — Node (Rust engine), Bun (`Bun.SQL` adapter path), Edge (HTTP fetch)
- **Multi-database** — connect to Postgres, MySQL, and SQLite in one runtime, switch with `db.use(name)`
- **Optional migrations** — connect to existing databases immediately, opt into migrations later

---

## CLI

The `jsorm` CLI is shipped by `@jsorm/node`.

```bash
jsorm init          # Bootstrap project
jsorm init --yes --adapter sqlite --sqlite-file data/app.sqlite  # Bootstrap without prompts
jsorm deploy        # Production-safe migration deploy
jsorm db:check      # Verify connectivity
jsorm migrate       # Run pending migrations
jsorm migrate:generate  # Diff models → reviewed migration file
```

---

## Documentation

| Section | Topics |
|---|---|
| [Getting Started](https://github.com/jsorm/docs/getting-started/installation.md) | Install, quick start, first model, first query |
| [Runtime](https://github.com/jsorm/docs/runtime/node-runtime.md) | Node, Bun, Edge, runtime profiles, workers |
| [Schema](https://github.com/jsorm/docs/schema/define-model.md) | Models, field types, relations, indexes, naming |
| [Migrations](https://github.com/jsorm/docs/migrations/migration-overview.md) | Generate, migrate, snapshots, diffing |
| [Adapters](https://github.com/jsorm/docs/adapters/pg.md) | PG, MySQL, SQLite, fetch, capabilities |
| [CLI](https://github.com/jsorm/docs/cli/cli-overview.md) | Init, deploy, db:check, migrate commands |
| [Deployment](https://github.com/jsorm/docs/deployment/overview.md) | Cloud, Docker, environment, CI/CD |
| [Concepts](https://github.com/jsorm/docs/concepts/architecture.md) | Runtime architecture, relations, transactions, errors |
| [Advanced](https://github.com/jsorm/docs/advanced/custom-adapters.md) | Custom adapters, custom queries, multi-tenancy, testing |
| [Examples](https://github.com/jsorm/docs/examples/blog-platform.md) | Blog platform, e-commerce, multi-tenant patterns |
| [Internal](https://github.com/jsorm/docs/internal/architecture.md) | Architecture, engine protocol, Rust worker |

Full tree: [docs/SUMMARY.md](https://github.com/jsorm/docs/SUMMARY.md)

---

## Community

- [GitHub](https://github.com/jsorm)
- [Contributing](CONTRIBUTING.md)
- [License](LICENSE) — MIT

