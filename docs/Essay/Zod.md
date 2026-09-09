---
title: Zod + zod-validator介绍
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

```js
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
// { error: ZodError, success: false }

```

# 推断类型

```js
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

PingRequestSchema:

```js
export const PingRequestSchema = z.object({
  page: z.coerce.number().int().positive().default(1), // 查询参数从 URL 解析出来本来都是字符串，这里用 z.coerce.number() 的意思是“先接收字符串，再自动帮你转成 number”。
  limit: z.coerce.number().int().positive().default(10),
})
```

zValidator 的第一个参数指定校验哪里的数据，
第二个参数指定校验Schema

```js
import { zValidator } from '@hono/zod-validator'

app.post('/rpc/system/ping', zValidator('json', PingRequestSchema), (c) => {
  const payload = c.req.valid('json') // 把通过检验的数据取出来 取出来的数据就是 PingRequestSchema里面的变量
  console.log(payload.page)
})
```

zValidator 的第三个参数是一个 hook 函数，让你自定义错误响应
```js
app.post(
  '/users',
  zValidator('json', createUserSchema, (result, c) => {
    if (!result.success) {
      return c.json(
        {
          code: 'VALIDATION_ERROR',
          errors: result.error.flatten().fieldErrors,
        },
        400
      )
    }
  }),
  (c) => {
    const data = c.req.valid('json')
    return c.json({ id: 1, ...data }, 201)
  }
)
```

**组合多个校验器**

```js
const paramSchema = z.object({
  id: z.string().regex(/^\d+$/),
})

const bodySchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
})

const headerSchema = z.object({
  'x-api-key': z.string().min(1),
})

app.put(
  '/users/:id',
  zValidator('param', paramSchema),
  zValidator('json', bodySchema),
  zValidator('header', headerSchema),
  (c) => {
    const { id } = c.req.valid('param')
    const body = c.req.valid('json')
    // 三个位置的数据都校验通过，类型都自动推导
    return c.json({ id, ...body })
  }
)
```
