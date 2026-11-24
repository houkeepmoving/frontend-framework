# 前端框架原理笔记

整理前端框架（React / Vue 等）底层原理与源码级实现机制的学习笔记。

## 目录结构

```
.
├── README.md
├── react/    # React 原理
│   └── React底层原理详解.md
└── vue/      # Vue 原理（计划中）
```

## 内容索引

### React

| 文档 | 内容 |
|---|---|
| [React底层原理详解](./react/React底层原理详解.md) | Fiber 架构起源与动机、FiberNode 数据结构、双缓冲（current / workInProgress）、Update 与 Lanes 优先级模型、Scheduler 调度与时间切片、Render 阶段（Reconciliation / Diff 算法）、Commit 阶段三段式（Mutation / Layout / Passive）、并发特性原理、Hooks 实现机制（链表结构、useState、useEffect）、Context 实现、常见设计问题 FAQ |

### Vue（计划中）

- 响应式系统（Object.defineProperty → Proxy）
- 编译器与模板优化（静态提升、PatchFlag、Block Tree）
- 组件更新与 Diff 算法（双端对比 / 最长递增子序列）

## 维护说明

- 新增一篇笔记时，放到对应框架目录下，并在上面的「内容索引」中补充一行
- 文档内部使用相对路径链接，目录调整时注意同步更新
