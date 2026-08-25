---
title: Cloudflare Workers 简介
tags:
  - Cloudflare
  - Serverless
  - Edge Computing
categories:
  - 前端技术
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

项目配置写在 wrangler.toml（或 wrangler.jsonc）里：
```bash
name = "my-worker"
main = "src/index.ts"
compatibility_date = "2024-01-01"

# 绑定 KV namespace
[[kv_namespaces]]
binding = "MY_KV"
id = "xxxxxxxxxxxxxxxxxxxx"

# 绑定 D1 数据库
[[d1_databases]]
binding = "DB"
database_name = "my-database"
database_id = "xxxxxxxxxxxxxxxxxxxx"
```

# 需要注意的限制

Workers 不是完整 Node.js 环境，以下差异需要留意：

- 不支持 Node.js 原生模块（如 `fs`、`child_process`），需使用 Web API 或兼容层
- 单次请求 CPU 时间有限制（免费版 10ms，付费可更高）
- 单 Worker 脚本体积有上限（压缩后约 1~10MB，视计划而定）

对于 I/O 密集型任务（API 代理、鉴权、轻量计算）非常合适；CPU 密集型任务应考虑 Durable Objects 或 offload 到后端服务。

# 小结

Cloudflare Workers 把 Serverless 推到了 CDN 边缘：写一段标准 Web 风格的 JS/TS，部署后全球可用。如果你已经在用 Cloudflare 做 DNS/CDN，或者需要极低延迟的轻量 API，Workers 是值得尝试的方案。
