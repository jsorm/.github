# 👋 Welcome to JSORM

<p align="center">
  <img src="https://raw.githubusercontent.com/jsorm/.github/main/assets/logo-dark.svg" alt="JSORM Logo" width="120" />
</p>

<h1 align="center">JSORM</h1>

<p align="center">
  <strong>A modern JSON-first TypeScript ORM.</strong><br/>
  Schema-driven models, deterministic query planning, multi-database support, and universal runtimes.
</p>

<p align="center">
  <a href="https://github.com/jsorm/monorepo/actions">
    <img src="https://github.com/jsorm/monorepo/actions/workflows/ci.yml/badge.svg" alt="CI">
  </a>
  <a href="https://www.npmjs.com/org/jsorm">
    <img src="https://img.shields.io/npm/v/@jsorm/core" alt="npm">
  </a>
  <a href="https://github.com/jsorm/monorepo/blob/main/LICENSE">
    <img src="https://img.shields.io/github/license/jsorm/monorepo" alt="license">
  </a>
</p>

---

## Define Models Once

JSORM uses TypeScript as the source of truth.

No decorators. No schema DSL. No code generation.

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
  select: {
    name: true,
    email: true,
  },
  where: {
    active: true,
  },
});
```

---

## Why JSORM?

| Feature               | Description                                                            |
| --------------------- | ---------------------------------------------------------------------- |
| JSON-first Queries    | Describe reads and writes with plain objects instead of builder chains |
| Full Type Inference   | Types derived directly from your model definitions                     |
| Deterministic AST     | Every query compiles through a versioned AST contract                  |
| Multi Runtime         | Node.js, Bun, Edge, Serverless                                         |
| Multi Database        | PostgreSQL, MySQL, SQLite                                              |
| Config First          | `jsorm.config.ts` is the single source of truth                        |
| Optional Migrations   | Connect immediately, migrate when ready                                |
| Rust-Powered Planning | High-performance SQL planning and compilation                          |

---

## Packages

| Package          | Purpose                                                |
| ---------------- | ------------------------------------------------------ |
| `@jsorm/core`    | Models, queries, configuration, migrations primitives  |
| `@jsorm/node`    | CLI, deployment workflows, Node.js runtime integration |
| `@jsorm/runtime` | Runtime-safe APIs for universal applications           |
| `@jsorm/fetch`   | HTTP transport for Edge environments                   |
| `@jsorm/pg`      | PostgreSQL adapter                                     |
| `@jsorm/mysql`   | MySQL adapter                                          |
| `@jsorm/sqlite`  | SQLite adapter                                         |

Install only the packages you need.

---

## Quick Start

### PostgreSQL

```bash
pnpm add @jsorm/core @jsorm/pg pg
```

### MySQL

```bash
pnpm add @jsorm/core @jsorm/mysql mysql2
```

### SQLite

```bash
pnpm add @jsorm/core @jsorm/sqlite better-sqlite3
```

### Initialize a Project

```bash
pnpm dlx @jsorm/node init
```

or

```bash
bunx @jsorm/node init
```

---

## CLI

```bash
jsorm init
jsorm deploy
jsorm db:check
jsorm migrate
jsorm migrate:generate
```

---

## Runtime Support

| Runtime    | Database Support                                |
| ---------- | ----------------------------------------------- |
| Node.js    | PostgreSQL, MySQL, SQLite                       |
| Bun        | SQLite, PostgreSQL, MySQL                       |
| Edge       | Any HTTP-accessible database via `@jsorm/fetch` |
| Serverless | Supported through runtime-specific adapters     |

---

## Architecture

JSORM follows a ports-and-adapters architecture.

```txt
Models
   ↓
JSON Query
   ↓
Normalized AST
   ↓
Query Planner
   ↓
SQL Generation
   ↓
Adapter Execution
   ↓
Database
```

The same query API works across runtimes and databases while preserving type safety and deterministic behavior.

---

## Resources

* 📚 Documentation
* 🚀 Getting Started
* 🛣️ Roadmap
* 🤝 Contributing
* 📦 npm Organization
* 💬 Discussions

---

<p align="center">
  Built for modern TypeScript applications.
</p>

<p align="center">
  MIT License · TypeScript · PostgreSQL · MySQL · SQLite · Edge Runtime
</p>
