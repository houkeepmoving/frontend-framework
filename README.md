# 前端框架原理笔记

整理前端框架（React / Vue 等）底层原理与源码级实现机制的学习笔记。

## 目录结构

```
.
├── README.md
├── react/    # React 原理
│   └── React底层原理详解.md
└── vue/      # Vue 原理
    └── Vue2与Vue3底层原理详解.md
```

## 内容索引

### React

| 文档 | 内容 |
|---|---|
| [React底层原理详解](./react/React底层原理详解.md) | Fiber 架构起源与动机、FiberNode 数据结构、双缓冲（current / workInProgress）、Update 与 Lanes 优先级模型、Scheduler 调度与时间切片、Render 阶段（Reconciliation / Diff 算法）、Commit 阶段三段式（Mutation / Layout / Passive）、并发特性原理、Hooks 实现机制（链表结构、useState、useEffect）、Context 实现、常见设计问题 FAQ；附 28 道常见面试题（按 Fiber 调度 / Hooks / Diff 渲染 / 性能工程 / 手写题分类） |

### Vue

| 文档 | 内容 |
|---|---|
| [Vue2与Vue3底层原理详解](./vue/Vue2与Vue3底层原理详解.md) | Vue2：defineProperty 响应式（Observer/Dep/Watcher）、依赖收集与派发更新、$set 与数组方法拦截、双端 Diff 算法、编译器三阶段、nextTick；Vue3：Proxy + effect 响应式、track/trigger 依赖结构、ref/computed 实现、编译优化（PatchFlag/静态提升/Block Tree）、快速 Diff 与最长递增子序列、渲染器 shapeFlags、Composition API 动机、Vue2 vs Vue3 对比总结；附 28 道常见面试题（按响应式/Diff/computed/nextTick/组件设计分类） |

## 维护说明

- 新增一篇笔记时，放到对应框架目录下，并在上面的「内容索引」中补充一行
- 文档内部使用相对路径链接，目录调整时注意同步更新
