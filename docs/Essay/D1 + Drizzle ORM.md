---
title: D1 + Drizzle 简介
tags:
  - ORM
  - TypeScript
categories:
  - 服务端
---

# Drizzle ORM 是什么？

Drizzle ORM 是一个 **TypeScript 优先** 的轻量级 ORM。它的查询 API 接近原生 SQL，Schema 定义即类型来源，编译后几乎零运行时开销。

和 Prisma 这类「生成 Client + 独立查询语言」的方案不同，Drizzle 更偏向 **SQL in TypeScript**：你写的查询读起来像 SQL，IDE 也能完整推导返回类型。

# 核心特点

- **TypeScript 原生** — Schema 即类型，查询结果自动推导，无需额外 codegen
- **轻量无魔法** — 没有重型运行时，bundle 体积小，适合 Edge / Serverless
- **SQL-like API** — `select`、`insert`、`update` 语法贴近 SQL，学习成本低
- **多数据库支持** — PostgreSQL、MySQL、SQLite，包括 Cloudflare D1
- **drizzle-kit** — 配套 CLI，负责 migration 生成与推送
- **关系查询** — 支持 `relations` 定义与 relational query API

# 最小示例


## 0. 安装

```bash
npm install drizzle-orm
npm install -D drizzle-kit
```

创建数据库

```bash
wrangler d1 create my-db
```
执行后会输出数据库 ID，把它配到 wrangler.jsonc：

```json
{
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-db", //数据库名称
      "database_id": "xxxx-xxxx-xxxx-xxxx"
    }
  ]
}

binding = "DB" 表示在代码里通过 c.env.DB 访问这个数据库。
```
## 1. 定义 Schema

```js
// ./db/schema
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core'
import { relations, sql } from 'drizzle-orm'

export const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  role: text('role').default('user'),
  createdAt: text('created_at').default(sql`CURRENT_TIMESTAMP`),
})

export const posts = sqliteTable('posts', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(), // text传入的值表示在数据库里的字段名
  content: text('content').notNull(),
  authorId: integer('author_id').notNull().references(() => users.id), // 和user关联
  createdAt: text('created_at').default(sql`CURRENT_TIMESTAMP`),
})

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}))

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
}))
```

从schema中可以推导出类型，方便在其他地方使用

```ts
import { InferSelectModel, InferInsertModel } from 'drizzle-orm'

// 查询结果的类型
type User = InferSelectModel<typeof users>
// { id: number; name: string; email: string; role: string | null; createdAt: string | null }

// 插入数据的类型（id、role、createdAt 是可选的）
type NewUser = InferInsertModel<typeof users>
// { id?: number; name: string; email: string; role?: string; createdAt?: string }

type Post = InferSelectModel<typeof posts>
// { id: number; title: string; content: string; authorId: number; createdAt: string | null }

type NewPost = InferInsertModel<typeof posts>
// { id?: number; title: string; content: string; authorId: number; createdAt?: string }
```

## 2. 连接数据库并查询

以 Cloudflare D1 为例（其他驱动用法类似，只是 `drizzle()` 的传参不同）：

```js
import { Hono } from 'hono'
import { drizzle } from 'drizzle-orm/d1'
import { users } from './db/schema'

type Bindings = {
  DB: D1Database
}

const app = new Hono<{ Bindings: Bindings }>()

app.get('/users', async (c) => {
  const db = drizzle(c.env.DB)
  const result = await db.select().from(users).all()
  return c.json(result)
})

export default app
```

**条件查询**

```ts
import { eq } from 'drizzle-orm'

const user = await db.select().from(users)
  .where(eq(users.id, id))
  .get()
// user 是 User | undefined
```

**插入**

```ts
const result = await db.insert(users)
  .values({ name, email })
  .returning()
  .get()
// result 是 User，字段带类型
// 如果 name 拼成了 nmae，TypeScript 直接报错
```

**更新**

```ts
await db.update(users)
  .set({ name })
  .where(eq(users.id, id))
  .run()
```

**删除**

```ts
await db.delete(users)
  .where(eq(users.id, id))
  .run()
```

**关联查询**

```ts
import * as schema from './db/schema'

const db = drizzle(c.env.DB, { schema })

// 查询用户及其所有文章
const usersWithPosts = await db.query.users.findMany({
  with: {
    posts: true,
  },
})
// 类型自动推导：{ id: number; name: string; ...; posts: Post[] }[]

// 查询文章及其作者
const postsWithAuthor = await db.query.posts.findMany({
  with: {
    author: true,
  },
})
```



## 3. 迁移工作流（drizzle-kit）

Schema 定义好了，怎么同步到数据库？这就是 drizzle-kit 的工作。

**配置 drizzle-kit**

在根目录创建drizzle.config.ts

```ts
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  schema: './src/db/schema.ts',
  out: './drizzle/migrations',
  dialect: 'sqlite',
})
```
1.生成迁移文件

```bash
wrangler d1 migrations create my-db init  // my-db 是数据库名称
```

这会在项目根目录生成 migrations/0001_init.sql 文件。写入建表语句：

```sql
CREATE TABLE IF NOT EXISTS users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  email TEXT NOT NULL UNIQUE,
  created_at TEXT DEFAULT (datetime('now'))
);
```

2. 用 wrangler 把迁移 SQL 跑到 D1 上：

```bash
# 本地执行迁移
wrangler d1 migrations apply my-db --local

# 远程执行迁移（部署到线上）：
wrangler d1 migrations apply my-db --remote
```

3. 本地开发

```bash
wrangler dev
```


如果想清空本地数据库，直接删掉 .wrangler/state/ 目录重新跑迁移就行。

# 和 Prisma 怎么选？

| | Drizzle | Prisma |
|---|---|---|
| 查询风格 | SQL-like，接近原生 | 独立 DSL（`findMany` 等） |
| 类型推导 | Schema 直接推导 | 需要 `prisma generate` |
| 运行时体积 | 极小 | 相对较大 |
| 上手难度 | 会 SQL 即可 | 学习 Prisma 语法 |
| 适用场景 | Edge、Serverless、追求轻量 | 全栈 CRUD、快速原型 |

没有绝对优劣。团队 SQL 基础好、部署在 Edge / D1 上，Drizzle 往往更顺手；需要成熟生态和可视化工具（Prisma Studio），Prisma 仍是稳妥选择。
