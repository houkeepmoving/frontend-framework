# React 底层原理详解

> 从 React 15 的困境讲起，拆解 Fiber 架构核心：FiberNode、双缓冲、Reconciliation、Scheduler 调度、Commit 提交、Hooks 实现机制。

---

## 一、为什么需要 Fiber：React 15 的架构困境

React 15 之前使用 **Stack Reconciler（栈协调器）**，它是**同步、不可中断**的：

```
setState
  └→ 递归遍历整棵组件树（stack 递归）
       ├→ componentReceiveProps
       ├→ render
       ├→ diff children
       └→ 同步更新 DOM
```

问题在于这个递归调用栈一旦开始就无法暂停——它是 JS 主线程上的同步执行。如果组件树很大（几千个组件），一次完整更新可能占用主线程 **50ms、100ms 甚至更久**，导致：

- 用户输入（键盘、点击）的响应被推迟，掉帧、卡顿
- 动画掉帧（一帧只有 16.6ms 预算）
- 无法实现「渲染中被打断，先响应更高优先级的交互」

递归的本质限制是：**依赖函数调用栈保存进度，栈一旦压进去就不能中途让出**。

Fiber 的解法思路：**把递归改成链表遍历 + 手动管理的循环**。用一个 JS 对象（FiberNode）保存每个组件的处理状态，遍历到哪、处理到什么阶段，都记在对象里，于是可以随时暂停、随时从记录点恢复——这就是**可中断的渲染**。

React 团队对这个架构的重写酝酿了两年（2016–2017），React 16 (2017) 正式落地。

---

## 二、FiberNode：架构的核心数据结构

Fiber 有两层含义：

1. **架构概念**：一种「可中断、可恢复」的执行单元，可以理解为「一个虚拟调用栈帧」
2. **数据结构**：`FiberNode` 对象，React 内部一切工作的载体

每个 React 元素（每个组件、每个 DOM 节点）在内部都对应一个 FiberNode。

### 2.1 FiberNode 的关键字段

```ts
interface FiberNode {
  // ─── 节点身份 ───
  tag: WorkTag,          // 节点类型标记：FunctionComponent / HostComponent(DOM) / ClassComponent...
  key: string | null,    // JSX 里的 key
  elementType: any,      // <App /> 里的 App 函数本身 / 'div' 字符串
  type: any,             // 同 elementType，语义上用于 diff
  stateNode: any,        // 真实 DOM 实例（HostComponent 时）或 class 实例；FunctionComponent 为 null

  // ─── 链表结构（Fiber 可以中断的关键）───
  return: Fiber | null,  // 父 Fiber（故意不叫 parent，因为这是链表不是树遍历）
  child: Fiber | null,   // 第一个子节点
  sibling: Fiber | null, // 下一个兄弟节点
  index: number,         // 在兄弟中的序号，diff 时用于位置对比

  // ─── 双缓冲 ───
  alternate: Fiber | null, // 指向另一棵树上「自己」的镜像节点

  // ─── 工作状态 ───
  pendingProps: any,     // 新的 props（待处理）
  memoizedProps: any,    // 上一次渲染已提交的 props
  memoizedState: any,    // 上一次渲染已提交的 state（对函数组件：hooks 链表头）
  updateQueue: UpdateQueue<any> | null, // class 组件的更新队列
  dependencies: Dependencies | null,    // 记录了 useContext 订阅的 context

  // ─── 副作用收集 ───
  flags: Flags,          // 位标记：Placement(插入) / Update(更新) / Deletion(删除) / Snapshot / Passive...
  subtreeFlags: Flags,   // 子树中所有副作用的聚合（React 17 优化引入）
  deletions: Array<Fiber> | null, // 要删除的子节点

  // ─── 优先级 ───
  lanes: Lanes,          // 本节点待处理的更新优先级（31 位二进制位图）
  childLanes: Lanes,     // 子树中待处理的优先级聚合

  // ─── 其他 ───
  mode: TypeOfMode,      // StrictMode / ConcurrentMode / ProfileMode...
  ref: any,
}
```

**重点理解链表结构**。组件树逻辑上是一棵树，但 Fiber 内部把它「压扁」成单向链表结构：

```
        App
       /   \
     Div    Span
     / \
   P    A

存储结构：
App.child → Div, Div.sibling → Span
Div.child → P, P.sibling → A
每个节点的 return 都指回父节点
```

为什么不用真正的树 + 递归遍历？因为**递归无法暂停**，而链表可以用一个指针记录「遍历到哪了」：

```js
// React 内部工作循环的简化形态
let current = rootFiber;
while (current !== null) {
  if (shouldYield()) {  // 时间片用完，检查是否该让出主线程
    // 保存 current，直接 return，让出线程
    // 下次恢复时从 current 继续遍历
    return;
  }
  current = performUnitOfWork(current); // 处理一个节点，返回下一个要处理的节点
}
```

`performUnitOfWork` 返回下一个节点的顺序：**优先 child → 没有 child 就 sibling → 都没有就 return 回父级**（completeUnitOfWork 阶段）。这就是整个 Reconciliation 的遍历方式。

### 2.2 WorkTag：节点类型

每个 FiberNode 有 tag 标识自己是什么：

| Tag | 对应 |
|---|---|
| `FunctionComponent` (0) | 函数组件 |
| `ClassComponent` (1) | 类组件 |
| `HostRoot` (3) | FiberRoot 下的根 |
| `HostComponent` (5) | 原生 DOM 元素 |
| `HostText` (6) | 文本节点 |
| `Fragment` (7) | Fragment |
| `MemoComponent` (14) | React.memo |
| `ForwardRef` (15) | forwardRef |

---

## 三、双缓冲（Double Buffering）：current 树与 workInProgress 树

React 内部**同时维护两棵 Fiber 树**：

- **current 树**：当前已提交、与屏幕上真实 DOM 一致的树
- **workInProgress 树**：正在后台计算的「未来状态」的树

每棵树的节点通过 `alternate` 字段互相指向对方：

```
current 树:  App₁ ←→ workInProgress 树: App₂
             (App₁.alternate = App₂, App₂.alternate = App₁)
```

### 3.1 工作流程

1. 初始渲染：只有 current 树从 HostRoot 逐步构建
2. 更新开始：React 从 current 树出发，**复用 alternate 对象**，在 workInProgress 树上创建/更新节点
3. workInProgress 树构建完成、提交后：它变成新的 current 树（`root.current = finishedWork`），原 current 树「退休」
4. 下一次更新又反过来：在退休树上重建，然后交换

**关键：这个交换是「瞬时指针切换」**。commit 阶段把 DOM 改完后，`root.current = workInProgressRoot`，一句话完成。这就是为什么 React 能做到「要么不显示新状态，要么完整显示新状态」——不存在半新半旧的中间态泄漏给用户（Suspense 的 fallback 就是利用这一点）。

### 3.2 为什么需要两棵树

- **并发可中断渲染**：后台渲染到一半被丢弃（比如被更高优先级更新打断），current 树上的 DOM 完全不受影响
- **复用对象，减少 GC**：alternate 节点会尽量复用（`WorkLoop` 里 `if (workInProgress.alternate !== null)` 就地复用），而不是每次新建

---

## 四、Update 与触发更新的机制

### 4.1 从 setState 到调度

以函数组件 `setCount(1)` 为例，完整链路：

```
1. dispatchSetState(fiber, hook, action)
   ├→ 创建 Update 对象 { payload: 1, lane: 请求的优先级 }
   ├→ 挂到 hook.queue.pending 环形链表上
   ├→ fiber.lanes |= lane          （标记这个 fiber 有工作）
   └→ ensureRootIsScheduled(root)  （从叶子冒泡到根，触发调度）

2. ensureRootIsScheduled → scheduleCallback(优先级, performConcurrentWorkOnRoot)
   （进入 Scheduler 的任务队列）

3. performConcurrentWorkOnRoot(root)
   ├→ renderRootSync / renderRootConcurrent（进入 WorkLoop）
   └→ commitRoot（渲染完成后统一提交）
```

**注意一个核心设计：更新从叶子冒泡到根**。`setState` 先在 fiber 上打 lane 标记，然后沿 `return` 链一路上溯，把父级的 `childLanes` 也标记上。这样调度器只需要看根节点的 lanes，就知道整棵树哪里有待处理的工作。

### 4.2 Update 的结构

```ts
interface Update<S> {
  lane: Lane,           // 这个更新自己的优先级
  payload: S | ((prev: S) => S),  // 值或 updater 函数
  next: Update | null,  // 环形链表指针
}
```

hook 的 `queue.pending` 是一个**环形单向链表**（pending 指向链表尾，`pending.next` 是头），这样可以 O(1) 同时完成「插入尾部」和「拿到整个链表」。

### 4.3 优先级系统：Lanes 模型

React 用一个 **31 位二进制位图** 表示优先级：

```ts
const NoLanes = 0b0000000000000000000000000000000;
const SyncLane = 0b0000000000000000000000000000001;    // 最高，同步
const InputContinuousLane = 0b...00010;               // 连续输入（拖拽、hover）
const DefaultLane = 0b...00100;                       // 常规更新
const TransitionLane = 0b...10000...;                 // startTransition 包裹的更新
const IdleLane = 0b1000...0;                          // 最低，空闲时才做
```

**位图设计的好处**：一个组件可能同时有多个不同优先级的更新（一个 sync 的 + 一个 transition 的），用位图可以 O(1) 合并（`|`）、判断（`&`）、取出最高优先级。

### 4.4 双优先级模型

React 内部有两套优先级，要注意区分：

| 系统 | 用途 | 取值 |
|---|---|---|
| **Lane（React 17+）** | Reconciler 内部表示更新的优先级 | 31 位位图 |
| **SchedulerPriority** | 交给 Scheduler 排序/过期用的数值 | ImmediatePriority(-1) ~ IdlePriority(5) |
| **eventPriority** | 事件处理函数里更新的初始优先级 | 离散事件→Sync，连续输入→InputContinuous，其他→Default |

事件监听器（比如 onClick 里 setState）之所以「看起来同步更新」，就是因为离散事件（click、keydown）创建的更新被标为 `SyncLane`，同步渲染。

---

## 五、Scheduler：调度器与时间切片

Fiber 架构把「做什么」(Reconciler) 和「什么时候做」(Scheduler) 分离了。

### 5.1 MessageChannel 与宏任务

Scheduler 不用 `setTimeout` 做切片（它有 4ms 的最小延迟且嵌套超过 5 层会被 clamp 到 4ms 以上），而是用 **`MessageChannel`**：

```js
const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;

function scheduleCallback(priority, callback) {
  // 把任务塞进按过期时间排序的小顶堆，然后 port.postMessage(null)
  // 浏览器在本次宏任务结束后立刻触发 onmessage，延迟接近 0
}
```

`MessageChannel` 的 onmessage 是一个**宏任务**，浏览器会先完成当前宏任务、微任务、渲染（paint），再执行它——所以 React 让出主线程后，浏览器有机会处理输入事件和绘制。

### 5.2 任务队列：小顶堆

Scheduler 维护两个按 `expirationTime（过期时间 = startTime + timeout）` 排序的小顶堆：

- **taskQueue**：普通任务，按过期时间排序
- **timerQueue**：延迟任务（`scheduleCallback` 可以传 delay）

每个优先级对应一个 timeout：

```ts
ImmediatePriority: -1ms  // 永不过期，最紧急
UserBlockingPriority: 250ms
NormalPriority: 5000ms
LowPriority: 10000ms
IdlePriority: 1073741823ms  // 几乎永不过期
```

「过期」的任务必须立刻同步执行，不再切片——这就是**饥饿避免**：低优先级任务虽然总是被高优先级插队，但一旦拖到过期，就会强制全量执行。

### 5.3 shouldYield：时间切片判断

```js
function shouldYield() {
  const timeElapsed = getCurrentTime() - eventStartTime; // 当前帧已工作时间
  if (timeElapsed < frameInterval /* 默认 5ms */) {
    return false; // 时间片没用完，继续干活
  }
  // 用完了，除非第一个任务已过期，否则让出
  return true;
}
```

**frameInterval 默认是 5ms**，不是 16.6ms。因为主线程让出再恢复有成本，5ms 的切片在实测中吞吐最高，同时一帧内也能切多次。

### 5.4 一帧的完整生命周期

```
一次事件循环：
  宏任务（可能是 Scheduler 的 MessageChannel 回调，跑 5ms 渲染工作）
    └→ 微任务队列清空
  ─── 浏览器接管 ───
  requestAnimationFrame 回调
  Style 计算 → Layout → Paint → Composite
  ─── 回到事件循环 ───
  下一个宏任务...
```

React 渲染工作就寄生在「宏任务」的空隙里，每次最多占 5ms，剩下的时间还给浏览器。

---

## 六、Render 阶段：Reconciliation（可中断）

Render 阶段 = 从根开始，构建 workInProgress 树。这是**唯一可中断的阶段**，且**必须保持纯函数语义**（可能被丢弃重跑）。

### 6.1 WorkLoop 与 performUnitOfWork

```js
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(fiber) {
  // ① beginWork：处理「进入」这个节点
  const next = beginWork(current, fiber, renderLanes);
  fiber.memoizedProps = fiber.pendingProps; // 处理完就固化 props

  if (next === null) {
    // 没有子节点，进入 completeUnitOfWork（「完成」阶段，回溯）
    completeUnitOfWork(fiber);
    return null; // completeUnitOfWork 内部自己推进指针
  }
  return next; // 返回 child，继续深入
}
```

遍历方向：

```
beginWork(App) → beginWork(Div) → beginWork(P)
→ P 无 child → completeUnitOfWork(P) → completeUnitOfWork(Div)
→ beginWork(A) → completeUnitOfWork(A) → completeUnitOfWork(App)
```

像一个「下潜上浮」的过程，每个节点恰好被 begin 一次、complete 一次。

### 6.2 beginWork：复用还是重渲染

beginWork 的核心决策是**「这个节点能不能跳过重渲染」**：

```
当前 fiber 有本次优先级的更新吗？ (fiber.lanes & renderLanes)
  ├→ 没有，且 props 浅比较没变（context 也没变）
  │    └→ bailout（短路）：直接克隆 current 节点，子树全部跳过
  └→ 有更新
       └→ 按类型执行 render：
            FunctionComponent → 调用函数本体，拿到新的 children（React元素）
            ClassComponent    → 实例化/调用 render()
            HostComponent     → 不调用 render，diff props
```

**Bailout（跳出）机制是 React 性能优化的根基**，`React.memo`、`shouldComponentUpdate`、`useMemo` 都是在这个分支上生效。跳出的前提是这个子树上没有比当前 renderLanes 更高的待处理更新。

### 6.3 Diff 算法：reconcileChildren

beginWork 拿到新的 children（React 元素数组）后，要和旧的子 Fiber 对比，决定复用谁、删除谁、移动谁。这就是 **Reconciliation / Diff**。

三个前提假设让算法从 O(n³) 降到 O(n)：

1. **不同类型的元素产生不同的树**：`<div>` 变 `<span>` → 直接销毁整棵子树重建，不尝试复用
2. **同层级比较**：不跨层移动节点（跨层等于删了重建）
3. **key 提示同身份**：同层级内用 key 判断「是不是同一个东西」

核心函数 `reconcileChildFibers`，按新 child 的形态分派：

```
单元素 → reconcileSingleElement
多元素 → reconcileChildrenArray / reconcileChildrenIterable
```

**多元素 diff（最难的部分）分两轮**：

**第一轮**：从左到右逐一对比，只要 `key` 和 `type` 都相同就复用（打上 Update flag），遇到第一个不匹配的节点就停止。

**第二轮**（处理剩余部分）：

- 如果新 children 遍历完了 → 剩下的旧节点全部打 `Deletion`
- 如果旧 children 遍历完了 → 剩下的新节点创建，打 `Placement`
- 两边都有剩余 → 进入 **移动判断**：把剩余的旧节点按 key 存入一个 Map，遍历剩余新节点，从 Map 里找。用一个 `lastPlacedIndex` 变量追踪「最右一个不需要移动的节点位置」：
  - 新节点在 Map 中能找到且 index ≥ lastPlacedIndex → 复用，不移动，更新 lastPlacedIndex
  - 能找到但 index < lastPlacedIndex → 复用，但打 `Placement`（要移动）
  - 找不到 → 新建，打 `Placement`

这就是为什么「把列表项从数组尾部挪到头部」会导致它后面的项全部被移动——`lastPlacedIndex` 的算法只保证**尽量少动，不保证最优解**。

### 6.4 completeUnitOfWork 与副作用列表

beginWork 只决定「子树长什么样」，complete 阶段做的是自底向上的收尾：

1. **HostComponent（DOM 节点）**：创建 DOM 实例（`document.createElement`），append 到父实例，**在此阶段做初始 DOM 属性赋值**（注意：更新路径上 DOM 修改不放这里，而是在 commit 阶段）
2. **副作用冒泡**：把自己的 `flags` 合入父级的 `subtreeFlags`

```
complete(P): P.flags |= Update (比如文本变了)
           → 父 Div.subtreeFlags |= Update
complete(Div): Div 的 subtreeFlags 继续冒泡给 App
```

**subtreeFlags 聚合的意义**：commit 阶段遍历 DOM 树时，如果某个父节点的 `subtreeFlags` 为 0，说明整个子树都没有任何副作用，可以**整棵子树跳过**——不用在每个节点上都判断。这是 React 17 用「聚合」替换了老的 `effectList 链表` 方案。

**flags 是位标记**：

```ts
const Placement = 0b00000000000010;      // 插入/移动
const Update = 0b00000000000100;         // 属性/文本更新
const Deletion = 0b00000000001000;       // 删除
const ChildDeletion = ...
const Snapshot = 0b...                   // getSnapshotBeforeUpdate
const Passive = 0b...                    // useEffect
const Layout = 0b...                     // useLayoutEffect
const Ref = ...
```

### 6.5 Render 阶段可以被丢弃重跑

整个 render 阶段不碰 DOM、不执行用户可见的副作用（这也是为什么 render 里不该有 side effect），所以被打断、被丢弃、被高优先级插队后重跑，都是安全的。

---

## 七、Commit 阶段：同步、不可中断

Render 完成后进入 commit。**commit 是同步且不可中断的**——因为用户不能看到「改了一半的 DOM」。

### 7.1 三个子阶段

**① Mutation（变更 DOM）**

从根开始遍历，只访问 `flags` 或 `subtreeFlags` 非 0 的节点：

- 执行 DOM 插入（`insertBefore`/`appendChild`）、删除（`removeChild`）、更新属性/文本
- 删除节点时同步触发 ref 的 detach 和相关 useEffect 的 cleanup 入队
- 删除前先把节点「临时藏起来」（display: none 等技巧，供 Suspense 恢复）

**② Layout（同步执行，paint 之前）**

- **DOM 变更完成后、浏览器绘制前**同步执行：
  - `useLayoutEffect` 的 effect 回调
  - `componentDidMount` / `componentDidUpdate`
  - 重新挂 ref
- 在这个阶段读 DOM 布局（`getBoundingClientRect` 等）是安全的：改 DOM 与读布局都发生在 paint 前，不会被中间状态污染，但**依然可能触发强制同步布局**（改了再读 → 浏览器被迫立刻 reflow），这是 layout 抖动的常见来源

**③ Passive（异步调度，paint 之后）**

- `useEffect` 不在 commit 里同步执行，而是通过 **Scheduler 安排一个 NormalPriority 任务**
- 这就是「useEffect 在浏览器绘制之后才跑」的原理——它根本不在 commit 调用栈里，而是另一个宏任务

### 7.2 flushPassiveEffects 与优先级

passive effects 队列会按优先级调度（比如数据变化引发的 effect 和 transition 渲染可以交错）。这也是为什么 useEffect 的清理和执行顺序不保证在同一个宏任务里。

---

## 八、并发特性如何基于这套架构

### 8.1 可中断渲染的时间线

```
用户 setState(startTransition 内)
  → TransitionLane 更新 → Scheduler LowPriority 调度
  → render 开始（5ms 切片）
     ⤷ 此时用户敲键盘 → 离散事件 → SyncLane 更新
        → React 丢弃当前 workInProgress 树（不 commit）
        → 先同步渲染 SyncLane 更新并 commit
        → 再从头重新渲染 transition 的部分
```

**丢弃的依据**：render 阶段所有进度都存在 FiberNode/WorkInProgress 变量里，不碰外部世界，所以扔掉是零成本的。

### 8.2 高优先级插队与「重复渲染」

一个细节：transition 渲染进行中，如果来了 sync 更新，React 会**从根重新开始渲染**（不是从断点继续混合渲染）。因此 transition 里的组件可能被渲染多次（render 不保证只执行一次），这正是「render 必须是纯函数」的并发原因。

### 8.3 Suspense 与「显示旧 UI」

Suspense 的原理：某个子组件 throw promise → React 在 render 阶段捕获 → 该子树标记 Suspended → commit 时**整棵 workInProgress 树不替换 current 树**，DOM 维持旧状态，只插入 fallback（通过 Offscreen Fiber 控制）。数据到达后重新渲染，才切换。这完全依赖双缓冲——「未来」和「现在」两棵树天然分离。

---

## 九、Hooks 的实现原理

函数组件没有实例，Hooks 的状态存在哪里？答案：**存在 FiberNode 上**。

### 9.1 Hook 链表

```ts
interface Hook {
  memoizedState: any,     // 当前 state（对 useState 就是值；对 useEffect 是 effect 对象）
  baseState: any,         // 上次「中断合并」后的基础值
  queue: UpdateQueue,     // 本 hook 的更新队列（环形链表）
  baseQueue: Update | null, // 被跳过（低优先级）未处理的更新
  next: Hook | null,      // 指向下一个 hook
}
```

FiberNode 的 `memoizedState` 字段，对函数组件而言就是 **hook 链表的头指针**。

### 9.2 为什么 Hooks 不能写在条件语句里

```js
function Component() {
  const [a, setA] = useState(0);   // hook[0]
  if (cond) {
    const [b, setB] = useState(0); // hook[1]
  }
  const [c, setC] = useState(0);   // hook[2] ← 危险！
}
```

React 按**调用顺序**匹配 hook：`mountState` 时逐个 append 到链表，`updateState` 时按 `currentHook = currentHook.next` 顺序读取。如果两次渲染调用顺序不同，链表位置对不上，直接报错 `Rendered more hooks than during the previous render`。这就是规则的根本原因——不是风格问题，是**链表匹配机制**。

（技术上 `useEffect`、`useMemo` 这类可以安全放在条件后面的场景，React 也依然不允许，为了 lint 可靠性和机制统一。）

### 9.3 useState 更新的完整过程

```js
const [count, setCount] = useState(0);

// 首次渲染（mountState）：
//   创建 hook 对象，memoizedState = 初始值，挂到 fiber.memoizedState 链表
//   返回 [state, dispatchSetState.bind(null, fiber, hook)]

// 更新（dispatchSetState）：
function dispatchSetState(fiber, queue, action) {
  const lane = requestUpdateLane(fiber);  // 依事件类型取优先级
  const update = { lane, payload: action, next: null };

  // 优化：如果 queue 为空且当前 fiber 没有其他待处理更新，
  // 且 action 与 current state 相同（Object.is）→ 提前 bail，不调度渲染
  if (fiber.lanes === NoLanes && ...) {
    const eagerState = typeof action === 'function' ? action(current) : action;
    if (Object.is(eagerState, currentState)) return; // 无变化，跳过
  }

  enqueueUpdate(fiber.queue, update);  // 入环形链表
  scheduleUpdateOnFiber(fiber, lane);  // 冒泡到根，触发调度
}

// 下一次 render 到这个 hook 时（updateState）：
//   遍历 queue.pending 里 lane ⊆ renderLanes 的更新，逐个算出新 state
//   lane 不满足的更新留给 baseQueue，下次低优先级渲染再处理
```

「高优先级更新插入时，低优先级更新被搁置在 baseQueue」就是**基于优先级的渐进渲染**——同一个 state，可以按不同优先级分批计算出不同的中间值。

### 9.4 useEffect 的三段结构

```ts
interface Effect {
  tag: Passive | Layout | HasEffect...,
  create: () => (() => void) | void,  // 你的回调
  destroy: (() => void) | null,       // 上一次的 cleanup
  deps: Array<any> | null,
  next: Effect,  // 环形链表（同一 fiber 上所有 effect 串起来）
}
```

commit 阶段对比 `deps`：浅比较数组每一项（`Object.is`）。无变化 → 跳过。有变化或首次 → destroy 入队 + create 入队。

---

## 十、Context 的实现

`createContext` 创建一个 `$$typeof: REACT_CONTEXT_TYPE` 的对象，含 `_currentValue`。

- **Provider**：在 render 阶段把 value 写到 `context._currentValue`，同时用**栈**管理嵌套 Provider（beginWork push，completeWork pop）
- **Consumer / useContext**：在函数执行时直接读 `_currentValue`，同时把自己的 fiber 加入 `dependencies` 链表
- **传播更新**：Provider 的 value 变化时，React 从该 fiber 向下做**深度优先查找**（`propagateContextChange`），找到所有 `dependencies` 里含该 context 的 fiber，给它们打上更新标记——所以 context 的更新会绕过 memo，强制下层订阅者重渲染

---

## 十一、全局流程总览

把所有环节串起来：

```
setState / 事件处理
   │
   ▼
创建 Update（带 lane）→ 挂 hook.queue / updateQueue
   │
   ▼
markUpdateLaneFromFiberToRoot：fiber.lanes 和祖先 childLanes 打标
   │
   ▼
ensureRootIsScheduled → Scheduler.scheduleCallback(优先级 → workLoop)
   │
   ▼
┌─────────────── Render 阶段（可中断、纯）──────────────┐
│ workLoopConcurrent:                                    │
│   while (workInProgress && !shouldYield()):            │
│     beginWork:                                         │
│       ├ bailout 判断（memo / props浅比较 / lanes）     │
│       ├ 执行组件函数 / render()                        │
│       └ reconcileChildren（diff，打 Placement/Update） │
│     completeUnitOfWork:                                │
│       ├ 创建 DOM 实例（首次）、收集 flags              │
│       └ subtreeFlags 冒泡                             │
│   （被打断：保存 workInProgress 指针，让出 5ms）       │
│   （被高优先级打断：丢弃重来）                         │
└────────────────────────────────────────────────────────┘
   │
   ▼  render 完成，同步进入
┌─────────────── Commit 阶段（同步、不可中断）──────────┐
│ ① Mutation: DOM 增删改、ref detach、effect 清理入队   │
│ ② Layout:   useLayoutEffect、componentDidMount/Update │
│             重新挂 ref、同步、在 paint 前              │
│ ③ root.current = finishedWork（双缓冲交换）           │
│ ④ Passive:  useEffect 通过 Scheduler 异步调度         │
└────────────────────────────────────────────────────────┘
   │
   ▼
浏览器 paint → 之后某次宏任务执行 passive effects
```

---

## 十二、几个常被问到的设计细节

**Q：为什么 render 阶段禁止副作用？**
因为并发模式下 render 可能被执行多次、甚至执行了但不 commit（被打断丢弃）。副作用放 render 里会导致重复执行或「渲染了但 UI 没变」的不一致。

**Q：React.memo 和 useMemo 分别作用在哪一层？**
`memo` 作用于 beginWork 的 bailout 分支（props 浅比较通过则跳过整个子树）；`useMemo` 只缓存 hook 链表里的值（复用 memoizedState），不阻止子组件重渲染。

**Q：为什么 key 用 index 会出问题？**
index 相同不代表是同一个元素。删除数组头部元素后，所有节点的 index 都前移，diff 会认为「第 0 个还是第 0 个」，复用了错误的 fiber（内部 state 串位），只更新内容——表现为 input 状态错乱、动画丢失。

**Q：为什么 useLayoutEffect 里读 DOM 不会闪烁而 useEffect 会？**
Layout 在 paint 前同步执行，改了 DOM 之后浏览器才绘制（只画最终状态）；Passive 在 paint 之后执行，用户会先看到一帧没有 effect 结果的画面（闪烁）。

**Q：Fiber 树和「组件树 / DOM 树」是什么关系？**
三棵树一一对应但不是同一个东西：JSX 描述的组件树（逻辑结构）→ Fiber 树（内部工作结构，带状态和调度信息）→ DOM 树（最终渲染结果）。Fiber 树上非 Host 的节点（函数组件等）在 DOM 里不存在对应节点。
