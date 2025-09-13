---
title: 一些杂记
categories:
  - 前端技术
---



# Babel 是如何将新的 JavaScript 语法转换为向后兼容的代码，以及它如何处理 Polyfill？

 源代码 → 解析(Parse) → 转换(Transform) → 生成(Generate) → 目标代码
 ```
 // 1. 解析阶段 (Parse)
// 源代码 → AST (抽象语法树)

// 输入代码
const code = `
  const arrow = (a, b) => a + b;
  class MyClass {
    method() {
      return this.value;
    }
  }
`;

// 2. 词法分析 (Lexical Analysis)
// 将代码分解为 tokens
[
  { type: 'Keyword', value: 'const' },
  { type: 'Identifier', value: 'arrow' },
  { type: 'Punctuator', value: '=' },
  // ... 更多 tokens
]

// 3. 语法分析 (Syntactic Analysis) 
// tokens → AST
{
  type: 'Program',
  body: [
    {
      type: 'VariableDeclaration',
      kind: 'const',
      declarations: [
        {
          type: 'VariableDeclarator',
          id: { type: 'Identifier', name: 'arrow' },
          init: {
            type: 'ArrowFunctionExpression',
            params: [
              { type: 'Identifier', name: 'a' },
              { type: 'Identifier', name: 'b' }
            ],
            body: {
              type: 'BinaryExpression',
              operator: '+',
              left: { type: 'Identifier', name: 'a' },
              right: { type: 'Identifier', name: 'b' }
            }
          }
        }
      ]
    }
    // ... class 的 AST 节点
  ]
}
 ```

语法转换机制：
去读取type，然后去调用相应的插件
```
// babel.config.js
module.exports = {
  plugins: [
    // 转换箭头函数 将箭头函数转换成普通函数
    '@babel/plugin-transform-arrow-functions',
    // 转换 class 识别 ClassDeclaration 节点，将class转换为构造函数 + 原型链
    '@babel/plugin-transform-classes',
    // 转换 const/let
    '@babel/plugin-transform-block-scoping'
  ]
};
```

# rel='preconnect'的用处是什么？
rel="preconnect" 是一个 HTML 链接关系属性，它的主要用处是预连接到外部域名，以提升网页加载性能。

主要用处：

1. 提前建立网络连接

在浏览器实际需要从外部域名加载资源之前，提前建立 DNS 解析、TCP 握手和 TLS 协商
减少后续资源加载时的网络延迟

2. 常见使用场景
```
<!-- 预连接到 Google Fonts -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

<!-- 预连接到 CDN -->
<link rel="preconnect" href="https://cdn.jsdelivr.net">

<!-- 预连接到图片服务器 -->
<link rel="preconnect" href="https://img.example.com">
```
3. 性能优势

DNS 解析时间节省：提前解析域名到 IP 地址

连接建立时间节省：提前完成 TCP 三次握手

SSL 握手时间节省：对于 HTTPS 站点，提前完成 TLS 协商

总体加载时间减少：可以节省 100-500ms 的加载时间

4. 使用注意事项

只对确定会用到的外部域名使用，避免浪费资源

浏览器通常限制同时预连接的数量（一般 6 个左右）

适用于页面加载过程中很快就会用到的资源

5. 与其他预加载指令的区别

preconnect：只建立连接，不下载资源

dns-prefetch：只进行 DNS 解析

preload：建立连接并下载特定资源

prefetch：低优先级预加载资源


# pageData.frontmatter.head ??= [] 这个写法是什么意思？

```
x ??= y
// 等价于
x = x ?? y
// 也等价于
if (x === null || x === undefined) {
  x = y
}
```

 具体含义
 ```
 pageData.frontmatter.head ??= []
 ```
 这行代码的意思是：
 
如果 pageData.frontmatter.head 是 null 或 undefined

那么 将其赋值为空数组 []

否则 保持原值不变

与其他运算符的区别

```
// ||= (逻辑或赋值) - 所有假值都会被替换
x ||= y  // 替换 false, 0, "", null, undefined, NaN

// ??= (空值合并赋值) - 只有 null 和 undefined 会被替换
x ??= y  // 只替换 null, undefined

// 示例对比
let a = ""
let b = ""

a ||= "default"  // a = "default" (空字符串被替换)
b ??= "default"  // b = "" (空字符串保持不变)
```

浏览器支持

现代浏览器：Chrome 85+, Firefox 79+, Safari 14+

Node.js：14.0+

TypeScript：4.0+
