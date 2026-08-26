---
title: Cloudflare Workers 简介
tags:
  - Cloudflare
  - Serverless
  - Edge Computing
categories:
  - 服务端
description: 简单介绍 Cloudflare Workers 是什么、能做什么，以及一个最小可运行示例
---

# Cloudflare Workers 是什么？

Cloudflare Workers 是 Cloudflare 提供的 **Serverless 边缘计算平台**。你可以在全球 300+ 个边缘节点上运行 JavaScript / TypeScript / WebAssembly 代码，无需自己管理服务器。

和传统 Node.js 服务不同，Workers 运行在 Cloudflare 自研的 **V8 隔离环境**（不是完整 Node.js 运行时），启动极快、冷启动几乎可忽略，请求会在离用户最近的节点处理。

# 核心特点

- **全球边缘部署** — 代码自动分发到全球 PoP，降低延迟
- **零冷启动感知** — V8 Isolate 模型，毫秒级启动
- **按请求计费** — 免费额度慷慨，小规模项目几乎零成本
- **Web 标准 API** — 基于 Fetch API、Request/Response，前端开发者上手成本低
- **丰富生态** — 可搭配 KV、D1、R2、Queues、Durable Objects 等存储与状态方案

# 配套服务

光有计算没有存储，什么也干不了。Cloudflare 围绕 Workers 构建了一整套配套服务：

KV（Key-Value Store） 全球分布式键值存储，最终一致性。适合配置项、缓存数据、session 信息。读取极快，写入有短暂延迟传播。

D1（SQL Database） 基于 SQLite 的 Cloudflare 托管数据库，支持标准 SQL。它适合结构化数据存储，但你不要把它简单理解成"和 KV 一样的全球分布式缓存"。D1、KV、Durable Objects 解决的是三类不同问题，后续章节会专门展开。

R2（Object Storage） 兼容 S3 API 的对象存储。存图片、文件、大块数据。没有出口流量费用——这一点比 AWS S3 便宜很多。

Queues（Message Queue） 消息队列，用于异步任务。比如用户上传文件后触发后台处理，不阻塞主请求。

Durable Objects 有状态的边缘计算。每个 Object 有自己的存储和单线程执行环境，适合需要强一致性的场景：实时协作、WebSocket 连接管理、计数器。

这些服务统一通过 Workers 的 Bindings 机制访问，代码里直接调用，不需要管连接字符串和认证。

# 最小示例

通过 Wrangler CLI 创建项目后，一个 Worker 的核心结构如下：

```js
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url)

    if (url.pathname === '/api/hello') {
      return Response.json({ message: 'Hello from the Edge!' })
    }

    return new Response('Not Found', { status: 404 })
  },
}
```

每个 Worker 导出一个 `fetch` 处理器，接收标准 Web `Request`，返回 `Response`。就这么简单——没有 Express、没有 `app.listen()`。

# 常见使用场景

| 场景 | 说明 |
| --- | --- |
| API 网关 / BFF | 在边缘聚合多个后端接口，做鉴权、限流、字段裁剪 |
| 静态站点增强 | 配合 Pages，在边缘做 A/B 测试、重定向、Header 注入 |
| 反向代理 | 修改请求/响应头、缓存策略、Geo 路由 |
| 轻量后端 | 结合 D1（SQLite）或 KV 构建 CRUD API |
| Webhook 处理 | 接收第三方回调，快速响应，异步丢进 Queue |

# 与 Cloudflare Pages 的关系

- **Pages** — 托管静态前端（HTML/CSS/JS），类似 Vercel / Netlify
- **Workers** — 运行服务端逻辑

两者可以组合：Pages 负责静态资源，同项目的 `functions/` 目录或独立 Worker 负责 API。对于全栈项目，也可以直接用 **Workers + Assets** 一体化部署。

# 开发工具：Wrangler

Wrangler 是官方 CLI，负责本地开发、部署和绑定资源：

```bash
# 安装
npm install -g wrangler

# 登录 Cloudflare 账号
wrangler login

# 创建新项目
npm create cloudflare@latest my-worker

# 本地开发（启动本地模拟环境）
wrangler dev

# 部署到 Cloudflare
wrangler deploy
```

wrangler dev 会在本地启动一个模拟 Workers 运行时的开发服务器，支持热更新。本地开发时也能访问 KV、D1、R2 等服务的本地模拟版本。

项目配置写在 wrangler.toml（或 wrangler.jsonc）里：【这里只使用了KV】

```json
{
	"$schema": "node_modules/wrangler/config-schema.json",
	"name": "<ENTER_WORKER_NAME>",
	"main": "src/index.ts",
	"compatibility_date": "2025-02-04",
	"observability": {
		"enabled": true
	},

	"kv_namespaces": [
		{
			"binding": "KV",
			"id": "<YOUR_BINDING_ID>"
		}
	]
}
```

使用:
```ts
export interface Env {
  USERS_NOTIFICATION_CONFIG: KVNamespace;
}

export default {
  async fetch(request, env, ctx): Promise<Response> {
    try {
      await env.USERS_NOTIFICATION_CONFIG.put("user_2", "disabled");
      const value = await env.USERS_NOTIFICATION_CONFIG.get("user_2");
      if (value === null) {
        return new Response("Value not found", { status: 404 });
      }
      return new Response(value);
    } catch (err) {
      console.error(`KV returned error:`, err);
      const errorMessage =
        err instanceof Error
          ? err.message
          : "An unknown error occurred when accessing KV storage";
      return new Response(errorMessage, {
        status: 500,
        headers: { "Content-Type": "text/plain" },
      });
    }
  },
} satisfies ExportedHandler<Env>;
```

```bash
npm run kv:sync 同步远程数据到本地
npx wrangler dev 启动本地服务
npm run deploy 将 KV 部署到 Cloudflare 的全球网络
```

# 需要注意的限制

Workers 不是完整 Node.js 环境，以下差异需要留意：

- 不支持 Node.js 原生模块（如 `fs`、`child_process`），需使用 Web API 或兼容层
- 单次请求 CPU 时间有限制（免费版 10ms，付费可更高）
- 单 Worker 脚本体积有上限（压缩后约 1~10MB，视计划而定）

对于 I/O 密集型任务（API 代理、鉴权、轻量计算）非常合适；CPU 密集型任务应考虑 Durable Objects 或 offload 到后端服务。

# 小结

Cloudflare Workers 把 Serverless 推到了 CDN 边缘：写一段标准 Web 风格的 JS/TS，部署后全球可用。如果你已经在用 Cloudflare 做 DNS/CDN，或者需要极低延迟的轻量 API，Workers 是值得尝试的方案。

# 补充
## 关系型数据库 vs 非关系型数据库

KV是非关系型数据库，D1是关系型数据库

关系型数据库的核心能力就是这些：

结构化：每张表有固定的列，每条数据都遵循同样的结构
SQL 查询：可以按任意字段筛选、排序、聚合、分组
表关联：通过外键把多张表串起来，联合查询
约束：唯一性、非空、外键约束，数据库帮你守规矩
常见的关系型数据库有 MySQL、PostgreSQL、SQLite。

非关系型数据库
非关系型数据库（NoSQL）不按表格来组织数据，根据存储方式的不同又分好几类 KV 就是其中一种：

键值存储（Key-Value）：一个 key 对应一个 value，就像一个巨大的 Map。Cloudflare KV、Redis 都属于这类
文档数据库（Document）：存的是 JSON 文档，每条数据的结构可以不一样。MongoDB 是典型代表
列族存储、图数据库……还有其他类型，这里不展开
非关系型数据库的共同特点是：灵活，但查询能力有限。不能写 SQL，不能跨表联查，很多事情得靠应用层代码自己实现。

```
wrangler d1 create my-db
```

配置wrangler.jsonc

```json
{
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-db",
      "database_id": "xxxx-xxxx-xxxx-xxxx"
    }
  ]
}
```

binding = "DB" 表示在代码里通过 c.env.DB 访问这个数据库。

在 Hono 里使用 D1，需要声明 Bindings 类型：
```ts
import { Hono } from 'hono'

type Bindings = {
  DB: D1Database
}

const app = new Hono<{ Bindings: Bindings }>()
```

D1 用迁移文件管理表结构，不要手动建表。

创建迁移文件
```
wrangler d1 migrations create my-db init
```

# 语法

**查询所有记录**

```ts
const { results } = await c.env.DB
  .prepare('SELECT * FROM users')
  .all()
```

**查询单条记录**

```ts
const user = await c.env.DB
  .prepare('SELECT * FROM users WHERE id = ?')
  .bind(id)
  .first()
```

**插入记录**

```ts
await c.env.DB
  .prepare('INSERT INTO users (name, email) VALUES (?, ?)')
  .bind(name, email)
  .run()
```
**更新记录**

```ts
await c.env.DB
  .prepare('UPDATE users SET name = ?, email = ? WHERE id = ?')
  .bind(name, email, id)
  .run()
```
**删除记录**

```ts
await c.env.DB
  .prepare('DELETE FROM users WHERE id = ?')
  .bind(id)
  .run()
```

.prepare() 写 SQL，用 ? 做参数占位符
.bind() 绑定参数，防 SQL 注入
.all() 返回多条，.first() 返回一条，.run() 不返回数据

**批量执行（事务）**
```ts
const results = await c.env.DB.batch([
  c.env.DB.prepare('INSERT INTO users (name, email) VALUES (?, ?)').bind('Alice', 'alice@example.com'),
  c.env.DB.prepare('INSERT INTO users (name, email) VALUES (?, ?)').bind('Bob', 'bob@example.com'),
  c.env.DB.prepare('INSERT INTO users (name, email) VALUES (?, ?)').bind('Charlie', 'charlie@example.com'),
])
```
batch() 里的语句会在同一个**事务**中执行，要么全成功，要么全失败。

# D1的限制
用之前要知道这些限制：

**数据库大小**：免费版单个数据库最大 500MB，账号总量 5GB；付费版单库最大 10GB，账号总量 1TB
**写入性能**：写操作需要同步到主节点，延迟比读高。写密集场景不适合 D1
**不支持事务嵌套**：batch() 是一个事务，但不能在事务里再开事务
**SQLite 语法**：不是 MySQL 也不是 PostgreSQL，部分语法有差异（比如一些仅限于 MySQL/PostgreSQL 的特有函数和索引写法）
**单次查询限制**：单条 SQL 最多返回 5MB 数据

对于大部分 Web 应用来说，这些限制不是问题。D1 就是为轻量级、读多写少的场景设计的。



