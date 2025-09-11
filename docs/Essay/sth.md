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

