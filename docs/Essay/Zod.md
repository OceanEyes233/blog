---
title: Zod 介绍
tags:
  - Typescript
categories:
  - 前端技术
---

# Zod是什么？

Zod 是一个优先支持 TypeScript 的验证库。使用 Zod，你可以定义模式，借助这些模式可以验证从简单的string到复杂嵌套对象的各类数据。

```
import * as z from "zod";
 
const User = z.object({
  name: z.string(),
});
 
// some untrusted data...
const input = { /* stuff */ };
 
// the parsed result is validated and type safe!
const data = User.parse(input);
 
// so you can use it with confidence :)
console.log(data.name);
```

# 功能特性

- **零外部依赖** — 不引入任何第三方包
- **跨平台运行** — 支持 Node.js 及所有现代浏览器
- **体积小巧** — 核心包仅约 2kb（gzip 压缩后）
- **不可变 API** — 链式方法返回新实例，原 schema 不会被修改
- **简洁的接口** — 声明式语法，上手成本低
- **TypeScript 友好** — 同时支持 TypeScript 与原生 JavaScript，类型可自动推导
- **内置 JSON Schema 转换** — 可将 Zod schema 导出为 JSON Schema
- **完善的生态系统** — 社区插件丰富，与主流框架集成良好

# 定义模式

```
import * as z from "zod"; 
 
const Player = z.object({ 
  username: z.string(),
  xp: z.number()
});

let res  = Player.parse({ username: "billie", xp: 100 })
console.log(res) 
// { username: "billie", xp: 100 }

let err = Player.safeParse({ username: "billie", xp: '111' })
console.log(res) 
// { error: ZodError
success: false }
```

# 推断类型

```
const Player = z.object({ 
  username: z.string(),
  xp: z.number()
});
 
// extract the inferred type
type Player = z.infer<typeof Player>;
 
// use it in your code
const player: Player = { username: "billie", xp: 100 };
```

# @hono/zod-validator

hono/validator 是 Hono 内置的通用校验中间件，@hono/zod-validator 是 Hono 官方的 Zod 集成，把 Zod schema 变成路由中间件。

```
import { zValidator } from '@hono/zod-validator'

app.post('/rpc/system/ping', zValidator('json', PingRequestSchema), (c) => {
  const payload = c.req.valid('json') // 把通过检验的数据取出来 取出来的数据就是 PingRequestSchema里面的变量
  ...
})

PingRequestSchema:

```
export const PingRequestSchema = z.object({
  name: z.string().trim().min(1),
})