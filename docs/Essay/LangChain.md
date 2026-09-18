---
title: LangChain介绍
tags:
  - LangChain
  - AI
---

# LangChain是什么
LangChain 是一个用于构建 LLM 应用和 AI Agent 的开源框架，2022 年 10 月由 Harrison Chase 创建，最初只有 Python 版本，后来推出了 JavaScript 版本。

LangChain 的核心定位用一句话概括就是：Agent = Model + Harness（模型 + 框架）。

模型负责推理，框架负责模型周围的一切——提示词管理、工具调用、对话记忆、执行循环、可观测性。

LangChain 把这些东西抽象成可组合的模块，让你像搭积木一样组装出自己的 Agent。

# 环境准备

创建项目目录
```bash
mkdir langchain-quickstart && cd langchain-quickstart
```

用 Bun 初始化 TypeScript 项目：
```bash
bun init
# 模板选择 Blank
```

安装依赖
```bash
bun add langchain @langchain/core @langchain/deepseek zod
```

注意：LangChain.js 要求 Node.js 22+

简单说明一下这些包：
langchain：核心包，提供 createAgent、tool、initChatModel 等高层 API
@langchain/core：基础类型和工具类
@langchain/deepseek：DeepSeek AI 模型集成（如果用其他模型，换成对应的包即可，比如 @langchain/openai）
zod：Schema 验证库，定义工具输入参数

**注意：之所以会有 langchain 和 @langchain/core 两个包，简而言之，就是当年 langchain 包越来越大，难以维持，于是做了拆分。**

# 第一次调用 LLM
在根目录创建.env文件

```js
DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxx
```
我们先不碰 Agent、不碰工具，就用 LangChain.js 调一次大模型。

修改index.ts

```ts
import { ChatDeepSeek } from "@langchain/deepseek";

const llm = new ChatDeepSeek({
  model: "deepseek-v4-flash",
  temperature: 0,
  // other params...
})

const aiMsg = await llm.invoke([ // 和llm交互
  [
    "system",
    "你是一个翻译助手，帮我把英文翻译成中文。",
  ],
  ["human", "Hello World!"],
])

console.log(aiMsg)
```

# 构建第一个 Agent

构建一个简单的天气查询 Agent：用户问天气，Agent 自动调用天气工具，返回结果。

创建agent.ts

```ts
import { createAgent, tool } from "langchain";
import { z } from "zod";

// 1. 定义工具
const getWeather = tool(
  async (input) => {
    const { city } = input;
    const response = await fetch(`https://wttr.in/${encodeURIComponent(city)}?format=3`);
    const weather = await response.text();
    return { weather };
  },
  {
    name: "get_weather",
    description: "获取指定城市的天气信息",
    schema: z.object({
      city: z.string().describe("城市名称，如北京、上海、广州"),
    }),
  }
);

// 2. 创建 Agent
const agent = createAgent({
  model: "deepseek:deepseek-v4-flash", // 使用的llm
  tools: [getWeather], // 调用tool
  systemPrompt: "你是一个天气助手，可以帮助用户查询城市天气。回答时简洁明了。"
});

// 3. 运行 Agent
const result = await agent.invoke( // 和llm交互
  { messages: [{ role: "user", content: "杭州今天天气怎么样？" }] },
);

// 4. 输出结果
const lastMessage = result.messages[result.messages.length - 1];
console.log(lastMessage.content);
```

# tool定义工具

tool 函数来自 langchain 包。它接收两个参数：一个函数（工具的实际逻辑）和一个配置对象（工具的元信息）。

配置对象中 3 个字段最重要：

name：工具名称，模型通过它来识别和调用工具
description：工具描述，这是模型决定是否使用该工具的关键依据——写清楚工具能做什么、什么时候该用
schema：用 Zod 定义函数的输入参数，模型会根据它来生成正确的调用参数

# invoke：运行 Agent

agent.invoke() 接收两个参数：输入消息和配置。当然这里我们只传入了用户消息，并没有进行其他的配置【比如后面可以配置thread_id】。

通过 agent.invoke 我们实现了大模型的调用和获取返回数据。

# 添加短期记忆功能

LangChain 通过 MemorySaver 解决这个问题。它是一个检查点机制，每次 Agent 调用结束后自动保存对话状态，下次调用时自动恢复。

先安装依赖：
```bash
bun add @langchain/langgraph
```

在创建agent的时候给agent加上记忆

```ts
import { MemorySaver } from "@langchain/langgraph";

const agent = createAgent({
  model: "deepseek:deepseek-v4-flash",
  tools: [getWeather],
  systemPrompt: "你是一个天气助手，可以帮助用户查询城市天气。回答时简洁明了。",
  checkpointer: new MemorySaver(),  // 加上这行
});
```

checkpointer 是 LangGraph 提供的记忆组件。把它传给 createAgent，Agent 就具备了自动保存和恢复对话的能力。

要使用记忆，还需要在调用时传入 thread_id：

```ts
// 3. 运行 Agent
const result = await agent.invoke(
  { messages: [{ role: "user", content: "杭州今天天气怎么样？" }] },
  { configurable: { thread_id: "weather-1" } } // 配置thread_id 
);

// 4. 输出结果
const lastMessage = result.messages[result.messages.length - 1];
console.log(lastMessage.content);

const result2 = await agent.invoke(
  { messages: [{ role: "user", content: "我刚才问了什么？" }] },
  { configurable: { thread_id: "weather-1" } }
);

console.log(result2.messages[result2.messages.length - 1].content);
```

注意：MemorySaver 是内存存储，进程重启就没了。生产环境可以换成 LangGraph 支持的持久化存储（比如 PostgreSQL），API 完全一样。

# LangSmith

开发 Agent 时，一个很自然的需求是：Agent 内部到底发生了什么？ 它调用了几次工具？每次传了什么参数？模型返回了什么？

LangChain 官方提供了 LangSmith 来解决这个问题。它是一个可视化追踪平台，记录 Agent 运行过程中的每一次 LLM 调用、工具调用和中间结果。接入方式很简单。https://smith.langchain.com/

我们去 LangSmith 平台免费注册一个账号，支持 Gmail、GitHub 账号登录。

创建一个项目后，然后生成一个 API Key：

将环境变量赋值到 .env 文件中：

```js
LANGSMITH_TRACING=true
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
LANGSMITH_API_KEY=xxxxxxxxxxxxxxxxx
LANGSMITH_PROJECT=xxxxxxxxxxxxxxxxx
```

设置后重新运行脚本，打开 LangSmith 控制台就能看到完整的调用链路——从用户输入到最终输出，每一步都清晰可见。

调试和理解 Agent 行为时非常有用。

本篇我们用不到 50 行代码，完成了 LangChain.js 的快速入门：

1. 用 ChatDeepSeek 调用大模型，完成第一次 LLM 对话
2. 用 tool 定义工具，让 Agent 能够查询外部数据
3. 用 createAgent 组装 Agent，把模型、工具、提示词打包在一起
4. 用 MemorySaver + thread_id 实现多轮对话记忆
5. 用 LangSmith 追踪 Agent 内部运行过程