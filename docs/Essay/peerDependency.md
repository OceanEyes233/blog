---
title: Peer Dependency 详解
tags:
  - npm
  - 包管理
  - 前端工程化
categories:
  - 前端技术
description: 详细解释 peerDependency 和 dependency 的区别及使用场景
---

# peer dependency 是什么意思？跟dependency的区别是什么？
Peer Dependency（对等依赖） 是指你的包需要与宿主项目共享的依赖，而不是由你的包直接安装的依赖。
Dependency（常规依赖）
```
{
  "dependencies": {
    "lodash": "^4.17.21"
  }
}
```

特点：

✅ 安装你的包时，会自动安装这些依赖

✅ 每个包都有自己独立的版本

❌ 可能导致多个版本共存，增加 bundle 大小

❌ 可能出现版本冲突

Peer Dependency（对等依赖）
```
{
  "peerDependencies": {
    "vue": "^3.0.0"
  }
}
```

特点：

❌ 安装你的包时，不会自动安装这些依赖

✅ 要求宿主项目必须已经安装指定版本

✅ 确保全局只有一个版本

✅ 避免版本冲突和重复安装

实际场景举例

场景1：Vue 组件库

```
// 你开发的 Vue 组件库 package.json
{
  "name": "my-vue-components",
  "peerDependencies": {
    "vue": "^3.0.0"
  }
}
```
为什么用 peerDependency？

你的组件需要与用户项目使用同一个 Vue 实例

如果用 dependency，会安装两个 Vue，导致问题

用户项目必须已经安装了 Vue 3.x

场景2：React 插件
```
{
  "name": "react-awesome-plugin",
  "peerDependencies": {
    "react": ">=16.8.0",
    "react-dom": ">=16.8.0"
  }
}
```

安装行为差异
使用 Dependency
```
npm install my-package
# 结果：
node_modules/
├── my-package/
│   └── node_modules/
│       └── lodash/  # 包自带的 lodash
└── lodash/          # 项目的 lodash（可能不同版本）
```

使用 Peer Dependency

```
npm install my-vue-component
# 警告：需要安装 vue
# npm WARN my-vue-component@1.0.0 requires a peer of vue@^3.0.0

npm install vue@^3.0.0  # 需要手动安装
# 结果：
node_modules/
├── my-vue-component/   # 组件包
└── vue/               # 共享的 Vue（全局唯一）
```

开发时的最佳实践

1. 同时声明 peerDependencies 和 devDependencies
```
{
  "peerDependencies": {
    "vue": "^3.0.0"
  },
  "devDependencies": {
    "vue": "^3.0.0"  // 开发时需要，用于测试和构建
  }
}
```

什么时候使用 Peer Dependency？

✅ 应该使用的场景

框架插件：Vue/React 组件库、插件

工具扩展：Webpack 插件、Babel 插件

样式框架：需要特定 CSS 框架的组件

类型定义：TypeScript 类型包

❌ 不应该使用的场景

工具函数库：如 lodash、moment

独立功能：不依赖宿主环境的包

内部依赖：包内部使用的工具

简单记忆：
dependency：我需要这个包，帮我安装

peerDependency：我需要你已经有这个包，我们一起用