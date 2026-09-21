---
title: LangChain.js 核心概念速通
tags:
  - LangChain
  - AI
  - LangGraph
  - LangSmith
---

# LangChain、LangGraph、LangSmith

LangChain 是框架，LangGraph 是工作流编排引擎，LangSmith 是观测平台。

# LangChain：高层框架

LangChain 是你直接打交道的那个框架。

你用它的 createAgent 创建 Agent，用它的 tool() 定义工具。LangChain 定位是“快速构建”，让你用最少的代码搭出一个能用的 Agent。

```ts
import { createAgent } from "langchain";

const agent = createAgent({
  model: "openai:gpt-4o",
  tools: [getWeather],
  systemPrompt: "你是一个天气助手。",
});
```

大多数场景下，你只需要用 LangChain 就够了。

# LangGraph：底层运行引擎
LangChain 的 createAgent 底层就是跑在 LangGraph 上的。

当 createAgent 的抽象不够用的时候。比如：

- 你需要自定义执行流程——不是简单的 ReAct 循环，而是有分支、有条件跳转、有并行执行的复杂工作流
- 你需要持久化状态——Agent 执行到一半挂了，重启后能从上次断的地方继续
- 你需要多 Agent 编排——多个 Agent 组成协作网络，消息在它们之间传递
- 你需要精细化的状态管理——在 Agent 执行过程中维护和修改自定义状态

LangGraph 用“状态图”（StateGraph）来建模这些问题。

你定义节点（Node）和边（Edge），节点是处理逻辑，边是流转方向，整个 Agent 就是一张有向图。

```js
import { StateGraph, END } from "@langchain/langgraph";

// 定义状态
const graph = new StateGraph({
  messages: { value: (x, y) => x.concat(y), default: () => [] },
});

// 添加节点
graph.addNode("planner", plannerNode);   // 规划节点
graph.addNode("executor", executorNode); // 执行节点
graph.addNode("reviewer", reviewerNode);  // 审查节点

// 添加边——定义流转逻辑
graph.addEdge("planner", "executor");
graph.addConditionalEdges("executor", (state) => {
  if (state.needsRevision) return "planner";  // 需要修改，回到规划
  return "reviewer";                           // 不需要修改，进入审查
});
graph.addEdge("reviewer", END);

const app = graph.compile();
```

# LangSmith：可观测性平台

Agent 跟普通程序最大的区别在于——它是一个黑盒。

你告诉它要做什么，它自己决定怎么做，中间走了几步、调了什么工具、花了多少 token，你很难直接看到。

LangSmith 是一个独立的 SaaS 平台（也有自托管版本），提供 Agent 的全链路可观测性。

它解决的核心问题包括：

- Trace 追踪：Agent 每一步做了什么——调了几次模型、每次返回什么、工具调用的参数和结果——全部可视化
- Token 和成本统计：每次运行花了多少 token，折合多少钱
- 延迟分析：哪个环节慢了，瓶颈在哪里
- 评估测试：用测试集评估 Agent 表现，支持回归测试
- Prompt 管理：版本化管理 Prompt，在线编辑和 A/B 对比

而且它的接入十分方便，上篇我们只是设置了几个环境变量，没有改任何业务代码：

```js
LANGSMITH_TRACING=true
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
LANGSMITH_API_KEY=xxxxxxxxxxxxxxxxx
LANGSMITH_PROJECT=xxxxxxxxxxxxxxxxx
```

# Deep Agents：高级 Agent 套件

除了这三个，LangChain Inc. 最近还推出了第四个产品：Deep Agents。

当我们使用 createAgent 时，它返回的是一个光杆 Agent——只会 ReAct 循环。

如果你需要更复杂的能力，比如读写文件、拆分任务交给子代理、跨会话记忆、任务规划……这些都需要你自己去实现。

而 Deep Agents 是一个功能强大的 Agent——基于 LangChain 和 LangGraph 构建，但内置了一整套开箱即用的能力，比如虚拟文件系统、子代理、长期记忆、技能系统、上下文管理等功能。

简单来说：createAgent 是毛坯房，createDeepAgent 是精装修。

写简单 Agent 可以用前者，写多步骤复杂任务建议用后者。

注意：Deep Agents 是 LangChain.js 生态中最新的 Agent Harness，目前仍在快速迭代中。对于生产项目，评估其稳定性后再决定是否采用。

# LangChain 的包结构
## langchain-core：基础抽象层

这是整个框架的地基，定义了所有核心接口：

- Runnable：所有组件的统一协议。不管是 Prompt、Model 还是 OutputParser，都实现了 invoke()、batch()、stream() 等方法
- ChatModel / LLM：模型调用的抽象接口
- PromptTemplate：提示词模板
- OutputParser：输出解析器
- Message：消息类型（SystemMessage、HumanMessage、AIMessage、ToolMessage）
- Tool：工具定义接口

**Runnable 是 LangChain 里最重要的抽象。它不是某个具体的类，而是一个接口协议。**

其中最核心的是这 4 个方法：
- invoke(input) — 异步调用，输入 → 处理 → 输出。最常用的就是这个。
- batch(inputs) — 批量调用，对效率敏感的批处理场景有用。
- stream(input) — 流式输出，逐 token 返回结果，适合需要实时展示的 UI。
- pipe(next) — 将当前 Runnable 的输出传给下一个 Runnable。被重载为 | 运算符。

看一个例子：

```js
import { ChatOpenAI } from "@langchain/openai";
import { ChatPromptTemplate } from "@langchain/core/prompts";
import { StringOutputParser } from "@langchain/core/output_parsers";

const prompt = ChatPromptTemplate.fromMessages([
  ["system", "你是一个简洁的助手，用一句话回答。"],
  ["human", "{question}"],
]);

const model = new ChatOpenAI({ model: "gpt-4o-mini", temperature: 0 });
const parser = new StringOutputParser();

const chain = prompt.pipe(model).pipe(parser);

const answer = await chain.invoke({ question: "什么是 RAG？" });
console.log(answer);
```

它们之所以能用一个 .pipe() 串起来，正是因为它们都收敛到了同一个接口。这个接口就是 Runnable。

在 LangChain.js 中，Prompt、Model、OutputParser、Tool、Agent——所有这些都实现了 Runnable 协议。

这也是为什么会有“搭积木”的说法。每个积木都是 Runnable，积木之间用 pipe 连接。

# langchain：核心框架包

这是你最常使用的包，提供了核心的 createAgent 函数：

```bash
npm install langchain
```

# @langchain/langgraph：图编排引擎
LangGraph 是 LangChain 的下层运行时。LangChain 的 createAgent 底层就是跑在 LangGraph 上的。

当你需要比 createAgent 更精细的控制——比如自定义状态图、复杂的多 Agent 协作流程、持久化检查点、人工审批——你就直接用 LangGraph。

```bash
npm install @langchain/langgraph
```

# @langchain/community：社区集成
第三方集成包，已经废弃了

# PromptTemplate

PromptTemplate 让你把提示词变成可复用的模板，通过变量注入动态生成。

```js
import { ChatPromptTemplate } from "@langchain/core/prompts";

const prompt = ChatPromptTemplate.fromTemplate(
  "你是一个{role}，请用{tone}的语气回答以下问题：
{question}"
);

const formatted = await prompt.invoke({
  role: "历史老师",
  tone: "幽默",
  question: "秦始皇为什么修长城？",
});

console.log(formatted);
// 你是一个历史老师，请用幽默的语气回答以下问题：
// 秦始皇为什么修长城？
```

在实际 Agent 开发中，PromptTemplate 更多是配合 LCEL 管道使用的，后面会讲到。

# Message（消息）

LangChain 中有 4 种消息类型：
- 系统消息（SystemMessage）：告诉模型如何运行，为交互提供上下文
- 人类消息（HumanMessage）：代表用户输入以及与模型的交互
- AI消息（AIMessage）：模型生成的响应，包括文本内容、工具调用和元数据
- 工具消息（ToolMessage）：表示工具调用的输出

LangChain 用消息对象来表示对话中的每一条消息：

```js
import {
  SystemMessage,
  HumanMessage,
  AIMessage,
  ToolMessage,
} from "@langchain/core/messages";

const messages = [
  new SystemMessage("你是一个天气助手"),
  new HumanMessage("北京今天天气怎么样？"),
  new AIMessage({
    content: "",
    tool_calls: [{ name: "get_weather", args: { city: "北京" } }],
  }),
  new ToolMessage({
    content: "晴天，25°C",
    tool_call_id: "call_xxx",
  }),
];

```
这看着挺抽象，其实就跟我们之前手写的 messages 数组是一样的概念，只是 LangChain 用类包装了一下，加上了类型信息：

```js
const messages = [
  { role: "system", content: "你是一个天气助手" },
  { role: "user", content: "北京今天天气怎么样？" },
  { role: "assistant", content: "" },
];

```
所以 SystemMessage、HumanMessage 看着挺复杂，其实就是生成了一个包含以下内容的对象：
- 角色：标识消息类型（例如system，user）
- 内容：指消息的实际内容（例如文本、图像、音频、文档等）。
- 元数据：可选字段，例如响应信息、消息 ID 和令牌使用情况

之所以使用 SystemMessage、HumanMessage 这种形式，就是为了提供了一种适用于所有模型提供程序的标准消息类型，确保无论调用哪个模型，行为都保持一致。

# OutputParser（输出解析器）

OutputParser 负责把 LLM 的输出解析成你想要的格式：

```js
import { StringOutputParser } from "@langchain/core/output_parsers";

const parser = new StringOutputParser();
// 把 AIMessage 的 content 提取为纯字符串
```

这段代码单看不知道起什么作用，我们把它放到完整的代码中：

```js
import { StringOutputParser } from "@langchain/core/output_parsers";
import { initChatModel } from "langchain";
import { ChatPromptTemplate } from "@langchain/core/prompts";

const model = await initChatModel("openai:gpt-4o");
const prompt = ChatPromptTemplate.fromMessages([
  ["human", "用一句话解释 {concept}"],
]);

const chain = prompt.pipe(model).pipe(new StringOutputParser());

const result = await chain.invoke({ concept: "Agent" });
console.log(typeof result); // "string"
```

初次接触这段代码的时候，你可能奇怪为什么需要提取为纯字符串，我直接读 response.text 不也行？

如果是非 stream 读取字符串，怎么实现都行，但如果要用 stream 的话，StringOutputParser 会在 .stream() 时把每个 chunk 都转成 string 增量，从而实现“打字机”的效果。

除了字符串，我们也可以让 LLM 按照 JSON 格式输出：

```js
import { JsonOutputParser } from "@langchain/core/output_parsers";

const parser = new JsonOutputParser();

const prompt = ChatPromptTemplate.fromMessages([
  ["system", "你是数据提取助手，以 JSON 输出，字段：name / age / skills。"],
  ["human", "{input}"],
]);

const chain = prompt.pipe(model).pipe(parser);

const result = await chain.invoke({
  input: "冴羽，21 岁，擅长前端开发",
});

console.log(result);
// { name: "冴羽", age: 21, skills: ["前端开发"] }
```

在 v1.0 后，如果你需要结构化输出，更推荐用 Zod schema + responseFormat 的方式：

```js
import { createAgent } from "langchain";
import { z } from "zod";

const WeatherReport = z.object({
  city: z.string(),
  temperature: z.number(),
  condition: z.string(),
  suggestion: z.string(),
});

const agent = createAgent({
  model: "openai:gpt-4o",
  tools: [getWeather],
  responseFormat: WeatherReport,
  systemPrompt: "你是一个天气助手。",
});
```

# LCEL：LangChain 表达式语言

LCEL 是学习 LangChain 必须掌握的内容，缩写看起来高大上，其实就是 LangChain Expression Language 的缩写，指的是 LangChain 的链式组合语法。

也就是我们讲 Runnable 时用到的 pipe 语法：

```js
const chain = prompt.pipe(model).pipe(parser);
```
在这样一个链中，我们把 prompt 传给 model，然后将返回的结果再传给 parser，从而形成了一条最经典的 LCEL 链：Prompt → Model → OutputParser

本质上就是一个数据从左往右流、每一站做一次加工的模式。

实现的底层逻辑是一样的，每个方法返回一个同类型的新对象，所以能一直 . 下去。 jQuery 返回 jQuery 实例，Runnable 返回 Runnable 实例。

# 总结

回顾一下本篇的内容：

产品矩阵：LangChain（框架）、LangGraph（引擎）、LangSmith（观测）、Deep Agents（套件），四层分工明确。用前端类比就是 Fiber → React → Next.js → DevTools.

包结构：langchain-core 定义 Runnable 协议，langchain 提供 createAgent 等高层 API，@langchain/langgraph 负责底层编排。

核心概念：LangChain.js 的 5 大核心抽象：Runnable 接口、Model I/O、Prompt Templates、Output Parsers、LCEL 表达式语言

概念虽然多，但都围绕一个核心思想：统一接口，搭积木。 理解这一点，后面的 Agent 开发就是具体代码的事了。




