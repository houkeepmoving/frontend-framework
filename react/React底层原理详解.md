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

---

# 第五部分：常见面试题

## A. Fiber 与调度类

### Q1：什么是 Fiber？React 为什么要重写为 Fiber 架构？

**答**：Fiber 有两层含义：一是架构概念——一种「可中断、可恢复」的执行单元（可以理解为可保存进度的虚拟调用栈帧）；二是数据结构——`FiberNode`，React 内部一切工作的载体，每个组件/DOM 节点对应一个。

重写的原因：React 15 的 Stack Reconciler 是**同步递归**的，依赖函数调用栈保存进度，一旦开始无法暂停。组件树大时一次更新可能占用主线程 50–100ms，导致输入响应延迟、动画掉帧，也无法实现「渲染中让位给更高优先级交互」。Fiber 把递归改成**链表遍历 + 手动循环**，进度记在 FiberNode 里，可随时暂停/恢复/丢弃，从而支撑时间切片和并发特性。React 16 (2017) 正式落地，重写酝酿了两年。

### Q2：Render 阶段为什么可中断，Commit 阶段为什么不可中断？

**答**：Render 阶段的工作是「构建 workInProgress 树」：执行组件函数、diff、打 flags——全程**不碰真实 DOM、不执行用户可见副作用**，所有进度存在 FiberNode 和变量里，所以被高优先级打断时可以整树丢弃、零成本重跑。Commit 阶段要做 DOM 增删改和生命周期执行，这个阶段如果中断，用户就会看到「改了一半的 DOM」，违反一致性，所以**同步、不可中断**。这也是「render 里禁止副作用」的根本原因：render 可能被多次执行甚至执行了但不提交。

### Q3：React 的时间切片是怎么实现的？为什么不用 requestIdleCallback / setTimeout？

**答**：Scheduler 用 **MessageChannel** 创建宏任务：任务按过期时间放进小顶堆，每个宏任务里执行工作循环，`shouldYield()` 判断已工作超过 **frameInterval（默认 5ms）** 就让出主线程，下次 MessageChannel 回调从保存的 `workInProgress` 指针继续。

- 不用 `requestIdleCallback`：触发频率太低（空闲才有），且 `timeRemaining` 不可控，Safari 曾不支持
- 不用 `setTimeout`：有 4ms 最小延迟，且嵌套超过 5 层会被 clamp 到 4ms 以上，切片太粗糙；MessageChannel 的 onmessage 在本次宏任务结束后立刻触发，延迟接近 0

**饥饿避免**：每个任务有超时时间（按优先级 250ms~10s+），过期的任务强制同步全量执行，低优先级任务不会被高优先级无限插队。

### Q4：React 的优先级模型是什么？说下 Lane 和 SchedulerPriority 的关系。

**答**：React 内部是多套优先级配合：

- **Lane**（React 17+）：31 位二进制位图，Reconciler 内部表示更新的优先级（SyncLane / InputContinuousLane / DefaultLane / TransitionLane / IdleLane…）。位图的好处是一个组件可同时持有多优先级更新，合并/判断/取最高都是 O(1) 位运算
- **SchedulerPriority**：交给 Scheduler 排序用数值（ImmediatePriority -1 到 IdlePriority 5）
- **事件优先级**：离散事件（click/keydown）创建 SyncLane 更新（表现为「同步更新」），连续输入（拖拽/hover）用 InputContinuousLane，`startTransition` 包裹的用 TransitionLane

更新从 fiber 冒泡到根（fiber.lanes 和祖先 childLanes 打标），调度器看根的 lanes 就知道哪里有工作。

### Q5：setState 之后到界面更新，中间发生了什么？

**答**：以函数组件为例：

1. `dispatchSetState` 创建带 lane 的 Update，挂到 hook.queue 的环形链表；先做一次 **eagerState 优化**（队列为空且新值与当前值 `Object.is` 相同则直接 return，不调度）
2. `scheduleUpdateOnFiber`：沿 return 链冒泡标记 lanes，`ensureRootIsScheduled` 触发 Scheduler 调度
3. Scheduler 按优先级安排 `performConcurrentWorkOnRoot`
4. **Render 阶段**（可中断）：workLoop 循环执行 beginWork（bailout 判断 → 执行组件函数 → reconcileChildren diff）和 completeUnitOfWork（创建 DOM 实例、flags 冒泡到 subtreeFlags）
5. **Commit 阶段**（同步）：Mutation（DOM 增删改）→ Layout（useLayoutEffect/componentDidMount，paint 前）→ 双缓冲切换 `root.current = finishedWork` → Passive（useEffect 通过 Scheduler 异步调度，paint 后）

### Q6：React 18 里 setState 是同步还是异步？（automatic batching）

**答**：React 18 之前就有批处理，但只在 React 事件处理函数和生命周期里生效（React 17 在 setTimeout/Promise 原生回调里 setState 是同步逐个渲染的，这是「有时同步有时异步」混乱的来源）。React 18 引入 **automatic batching**：无论在什么上下文（setTimeout、Promise、原生事件），多次 setState 默认合并为一次渲染。想要「改完立刻拿到新 DOM」用 `flushSync(fn)` 强制同步刷新。所以面试标准答法：**「同步/异步」的表述不准确，准确说法是「是否批处理」——批处理是优化，不是异步 API**。

### Q7：并发模式（concurrent rendering）下 render 可能重复执行，这对写代码有什么要求？

**答**：transition 渲染进行中若来了 sync 更新，React 会**丢弃当前 workInProgress 树、从根重新渲染**，而不是从断点继续。因此组件函数在同一个提交周期内可能被调用多次（执行了但没 commit）。要求：

- render/组件函数必须是**纯函数**：不能在里面发请求、改外部变量、操作 DOM
- 副作用只放 useEffect / 事件回调里
- StrictMode 在开发环境故意双调用组件函数来提前暴露违反纯函数的代码

## B. Hooks 原理类

### Q8：Hooks 的实现原理是什么？

**答**：函数组件没有实例，Hooks 状态存在 **FiberNode 上**。fiber 的 `memoizedState` 字段是 **hook 链表的头指针**，每个 Hook 节点含 `memoizedState`（当前值）、`queue`（更新队列，环形链表）、`next`。首次渲染（mount 阶段）按调用顺序逐个创建 hook 节点 append 到链表；更新渲染时按 `currentHook = currentHook.next` **顺序匹配**读取。React 内部用 `ReactCurrentDispatcher` 在 mount 和 update 之间切换两套 hooks 实现。setState 的 dispatch 闭包持有对应 fiber 和 hook 引用，所以每次渲染拿到的是同一个 setter。

### Q9：为什么 Hooks 不能写在条件语句/循环里？

**答**：因为 hook 是**按调用顺序**与链表节点一一匹配的。如果两次渲染调用顺序不同（条件分支里少调了一个 hook），链表位置对不上，状态会串到别的 hook 上。React 直接报错 `Rendered more hooks than during the previous render`。这不是风格约定而是机制约束——链表匹配没有「按名字查找」的能力（这也是早期讨论过的 alternative：按 key 索引，最终为了简单选择了顺序约定）。eslint-plugin-react-hooks 就是为静态检查这个约束。

### Q10：setState 传入和当前相同的值，会重新渲染吗？

**答**：分两层：

1. **eagerState 优化**（dispatchSetState 内）：如果当前 fiber 没有其他待处理更新，React 会**立刻 eagerly 计算** `action(currentState)`，与当前值 `Object.is` 相同则直接 return，**连调度都不发生**
2. 如果该 fiber 上还有其他更新（无法 eager 比较），会走完整调度，render 执行后 reducer 算出的 state 相同，**bailout 机制**可能让组件跳过重渲染——但注意 bailout 有条件：父组件本身在渲染中、或上下文变化时，子组件即使 state 没变仍可能重渲染（`eagerState` 只影响调度层）

结论：「相同值不渲染」大体成立但**不是 100% 保证**，依赖不可变数据 + memo 才是可靠的。

### Q11：useEffect 和 useLayoutEffect 的区别？执行时机分别是什么？

**答**：

- **useLayoutEffect**：在 Commit 的 **Layout 阶段**同步执行——DOM 已修改、浏览器**尚未绘制**。适合需要在绘制前测量/修改 DOM 的场景（如测量元素尺寸后定位 tooltip）。注意：里面「改了再读布局」会触发**强制同步 reflow**
- **useEffect**：不在 commit 调用栈里，commit 结束后由 Scheduler 安排一个 **NormalPriority 的异步任务**（paint 之后、另一个宏任务里）执行。适合订阅、请求、日志等副作用

区别的表现：useLayoutEffect 里改 DOM 用户看不到中间帧（不闪烁）；useEffect 改依赖视觉状态的 DOM 可能闪一帧。React 官方建议 90% 场景用 useEffect，SSR 时 useLayoutEffect 会有告警（服务端没有 paint 概念）。

### Q12：useEffect 的 cleanup 什么时候执行？销毁组件和依赖变化的执行顺序？

**答**：cleanup 与 effect 成对：commit 阶段 diff deps（`Object.is` 逐项浅比较），有变化时 **先 destroy（上一次的 cleanup）再 create（新的 effect）**；组件卸载时在 Mutation 阶段把 cleanup 入队执行。注意两点：cleanup 执行时机也是异步的（passive 队列），同一个 fiber 上多个 effect 按声明顺序成对执行；React 18 StrictMode 下开发环境 mount → cleanup → mount 会跑两遍以暴露副作用不幂等的问题。

### Q13：useMemo / useCallback 的原理是什么？滥用的代价？

**答**：`useMemo(fn, deps)` 在 hook 链表节点上缓存 `memoizedState`（计算结果）和 `deps`，更新渲染时浅比较 deps，相同则直接返回缓存值**不执行 fn**。`useCallback(fn, deps)` 就是 `useMemo(() => fn, deps)` 的别名，缓存的是函数引用。

滥用代价：每个 useMemo/useCallback 本身有开销（比较 deps、维护链表节点），比直接计算更贵；并且**只优化本组件，不阻止子组件重渲染**——除非子组件是 memo 包裹的且 props 引用稳定才生效。正确用法：① 传给 memo 子组件的 props 引用稳定化；② 真正昂贵的计算；③ 作为其他 Hook 的依赖。无脑全包反而更慢。

### Q14：useRef 的原理？和 createRef 的区别？

**答**：`useRef(initialValue)` 首次渲染时创建一个 `{ current: initialValue }` 对象，挂在 hook 链表的 `memoizedState` 上；之后每次渲染**返回同一个对象引用**（updateRef 分支直接返回 currentHook.memoizedState）。所以改 `ref.current` 不会触发渲染（它不是响应式的），且跨渲染保持值——适合存 DOM 引用、定时器 id、最新值镜像（避免 useEffect 闭包陈旧值）。

createRef 用于 class 组件（每次 render 都会创建新对象，不能用在函数组件，否则每次渲染都是新 ref）；useRef 在 fiber 上持久化。另外 `forwardRef` 是把父组件传入的 ref 转发到子组件内部节点/自定义暴露（配合 `useImperativeHandle` 控制暴露内容）。

### Q15：自定义 Hook 和 mixin/HOC/render props 相比解决什么问题？

**答**：逻辑复用的三代方案：

- **mixin**（React 早期）：属性合并，来源不透明、命名冲突——已废弃
- **HOC**：组件包装，props 命名冲突（嵌套地狱）、ref 透传麻烦、调试时组件层级爆炸
- **render props**：显式传递但嵌套缩进深，只能复用「渲染」不能复用「逻辑+状态」

**自定义 Hook**：复用的是**有状态的逻辑本身**（每次调用是独立的 hook 链表片段，状态天然隔离），无嵌套层级、无命名冲突（返回值可重命名）、类型推断友好。本质是「函数 + hooks 顺序约定」的组合，与 Composition API 的 composable 是同一设计思路的两种语言实现。

## C. Diff 与渲染类

### Q16：React 的 Diff 有哪三个前提假设？

**答**：1）**不同类型的元素产生不同的树**：`<div>` 变 `<span>` 直接销毁整棵子树重建，不尝试复用；2）**同层级比较**：不跨层移动节点，跨层等于删了重建；3）**key 提示同身份**：同层级内用 key 判断是不是同一个元素。这三个假设把树 diff 从 O(n³) 降到 O(n)。

**与 Vue 的实现差异**：React 的列表 diff 是**单向从左到右**遍历（靠 key 匹配 + lastPlacedIndex 判断移动），不移动匹配节点、只移动「顺序不对」的节点；Vue 2 是**双端 diff**（四指针头尾对比），Vue 3 是**预处理 + 最长递增子序列**。三者共同点都是同层比较 + key 复用，差异在移动策略的优化程度。

### Q17：为什么列表要用稳定的 key？index 做 key 什么时候会出事？

**答**：key 决定「复用还是重建」。用 index 的问题：插入/删除/排序后，同位置的 index 相同但**元素身份变了**，diff 认为「key 0 还是 key 0」，就地复用错误的 fiber——内部 state 串位（input 输入值错乱）、动画丢失、组件生命周期不触发。出事条件：**列表会插入/删除/重排，且子项有内部状态**。安全的场景：列表纯静态展示、只在尾部追加且不删除。此外 key 要「稳定且兄弟间唯一」，不要用 `Math.random()`（每次渲染都变，等于全部重建）。

### Q18：React 怎么决定一个组件要不要重渲染？bailout 机制是什么？

**答**：beginWork 时判断：

1. 当前 fiber 的 `lanes` 里有没有本次 render 的优先级更新（`fiber.lanes & renderLanes`）
2. props 是否浅比较相同（fiber.pendingProps vs memoizedProps，memo 组件就是在这里加 `shallowEqual` 的自定义比较）
3. context 有没有变化（dependencies 里记录的 context 版本）

三者都「没变化」→ **bailout**：直接克隆 current 节点，`alternate` 复用，整个子树跳过。`React.memo`、`shouldComponentUpdate`、`PureComponent` 都作用于第 2 步；`useMemo/useCallback` 通过稳定 props 引用为第 2 步创造条件。注意 context 变化会**穿透 memo**，强制订阅者重渲染。

### Q19：父组件重渲染时，子组件默认会发生什么？怎么阻止？

**答**：默认**子组件整个重渲染**（哪怕 props 没变）：父 render 产生新的子 React 元素（新对象引用），React 对非 memo 组件不比较 props 直接重新执行子组件函数。阻止手段按优先级：

1. **React.memo** 包子组件（props 浅比较；复杂结构配合自定义比较函数）
2. 稳定 props 引用：函数 props 用 `useCallback`，对象 props 用 `useMemo`，否则 memo 形同虚设
3. **状态下沉**：把频繁变化的状态拆到更小的子组件里，缩小重渲染范围（结构性优化，比 memo 更根本）
4. `useMemo` 包 JSX（Children memoization）
5. 内容提升：把 children 作为 props 传入（JSX 在父组件创建，但如果父不重渲染它就不变）

## D. 性能与工程类

### Q20：React 渲染性能优化的完整手段清单？

**答**：

- **减少重渲染范围**：memo + useCallback/useMemo、状态下沉、children 透传、context 拆分（把频繁变化的 context 拆出去，避免大范围订阅者重渲染）
- **减少渲染成本**：虚拟列表（react-window/virtuoso）处理长列表、避免在 render 里做昂贵计算（useMemo）、懒加载（React.lazy + Suspense 代码分割）
- **优化更新调度**：非紧急更新用 `startTransition`/`useDeferredValue`（如搜索输入时列表过滤），让输入保持响应
- **避免副作用层的问题**：useEffect 依赖写对（避免无限循环）、数据请求竞态处理（AbortController / cleanup 标志位）
- **不可变更新**：正确触发浅比较（useState 里直接改引用内容会让 memo 失效）

### Q21：不可变数据（immutable update）为什么是 React 的核心约定？

**答**：因为 React 的相等性判断全是**引用比较**（`Object.is`）：state 相等性、memo 的 props 浅比较、useMemo/useEffect 的 deps 比较。如果直接修改对象内部属性（`obj.x = 1`），引用不变，React 认为「没变」，跳过更新——这是最经典的「改了但界面不更新」bug 的根源。所以更新嵌套状态要用展开运算符/结构化复制创建新引用（`{...obj, x: 1}`、`arr.concat()`），大型状态树可引入 immer（Proxy 写时复制）或 Immutable.js。

### Q22：startTransition / useDeferredValue 是什么？解决什么问题？

**答**：都用于**优先级分层**：把「可以让界面等一下」的更新标为 TransitionLane（低优先级），让输入等紧急更新插队。

- `startTransition(() => setList(filtered))`：包裹更新本身。搜索框场景：输入（SyncLane）立即响应，过滤重列表（TransitionLane）可中断、可被丢弃重跑，不阻塞打字
- `useDeferredValue(value)`：包值，返回延迟版本——本质是「用旧值渲染一帧 + 调度用新值的渲染」，适合把「上一次的值先展示着」的场景

区别：startTransition 控制的是「哪段代码触发的更新」，useDeferredValue 控制的是「某个值的变化被延后感知」，后者无需改 setState 调用处，适合封装在子组件内部。

### Q23：Suspense 的工作原理？

**答**：子组件在 render 中 **throw 一个 promise**（legacy 数据获取方式）→ React 在 render 阶段捕获，把该子树标记为 Suspended → commit 时**整棵 workInProgress 树不替换 current 树**（DOM 保持旧状态），只显示最近的 Suspense 边界的 fallback。promise resolve 后重新调度渲染，成功才切换到新内容。这完全依赖双缓冲——「未来」和「现在」两棵树天然分离。React 18 之后配合 `React.lazy`（代码分包）、RSC（Server Component）数据流使用；`use()` + 框架级缓存是新的推荐用法。

## E. 手写与综合类

### Q24：手写一个简化版 useState。

```js
// 单 fiber 简化版：演示 hook 链表 + 状态持久化 + 触发重渲染
let hooks = [];
let hookIndex = 0;
let isMount = true;

function useState(initialValue) {
  const currentHookIndex = hookIndex;
  hooks[currentHookIndex] = hooks[currentHookIndex] ?? initialValue;

  function setState(newState) {
    hooks[currentHookIndex] =
      typeof newState === 'function' ? newState(hooks[currentHookIndex]) : newState;
    scheduleRender();  // 触发重渲染（真实 React 里是调度，这里是简化）
  }

  hookIndex++;
  return [hooks[currentHookIndex], setState];
}

function render() {
  hookIndex = 0;   // ★ 每次渲染重置游标，hooks 按调用顺序重新匹配
  isMount = false;
  App();           // 执行组件函数
}
```

要点：① 状态存在**外部闭包的链表**里（对应 fiber.memoizedState），组件函数本身无状态；② 每次 render 重置 `hookIndex`，按调用顺序匹配——这就是「不能写条件分支」的原因；③ 函数式更新 `setState(prev => ...)` 支持批量。

### Q25：手写防抖 / 节流 Hook。

```js
function useDebouncedValue(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);   // ★ cleanup 清掉上一次的 timer，实现防抖
  }, [value, delay]);
  return debounced;
}

function useThrottledValue(value, interval = 300) {
  const [throttled, setThrottled] = useState(value);
  const lastExecuted = useRef(0);
  useEffect(() => {
    const now = Date.now();
    const remaining = interval - (now - lastExecuted.current);
    const timer = setTimeout(
      () => { lastExecuted.current = Date.now(); setThrottled(value); },
      Math.max(remaining, 0)
    );
    return () => clearTimeout(timer);
  }, [value, interval]);
  return throttled;
}
```

要点：防抖/节流的「重置」全部靠 **useEffect cleanup** 实现，不需要手动管理 timer 生命周期。

### Q26：React 和 Vue 的响应式/更新模型有什么本质区别？

**答**：

| 维度 | React | Vue |
|---|---|---|
| 更新触发 | **推模式（渲染时机）**：setState 标记「组件需要重渲染」，从该组件**向下**重新执行整个子树 | **推模式（数据驱动）**：数据 setter/effect 精确知道谁依赖它，**只通知订阅者** |
| 粒度 | 组件级：默认子树全量重渲染，靠 memo/reducer bailout 缩小 | Vue2 组件级（渲染 Watcher），Vue3 编译优化后接近节点/属性级 |
| 依赖收集 | 无（运行时不知道 render 读了什么，只能全量重跑再 diff） | 运行时收集（getter 触发 track），依赖关系精确 |
| 不可变要求 | 强（引用比较判定变化） | 弱（Proxy setter 直接感知属性修改，`obj.x = 1` 就能触发） |
| 优化位置 | 运行时（memo、useMemo、虚拟 DOM diff） | 编译时（PatchFlag、Block Tree、静态提升） |

一句话总结：**React 的思路是「重新执行 + diff 兜底」，Vue 的思路是「精确追踪 + 最小更新」**——这决定了两边性能优化手段完全不同（React 优化「少执行」，Vue 优化「少 diff」）。

### Q27：React 18 的新特性有哪些？分别基于什么原理？

**答**：

- **automatic batching**：调度层改动，所有上下文的 setState 默认合并批处理
- **并发渲染 + startTransition/useDeferredValue**：基于 Fiber 可中断渲染 + Lane 优先级（TransitionLane）
- **Suspense 增强（流式 SSR）**：`renderToPipeableStream` 服务端流式注入 HTML + selective hydration（按优先级分批水合）
- **useId**：SSR/流式场景下生成稳定的唯一 id（同一次渲染内一致，避免水合不匹配）
- **useSyncExternalStore**：为外部 store（Redux/Zustand）提供「并发安全的订阅」——并发渲染下 store 快照可能撕裂（tearing），这个 API 保证读到的快照与当前渲染一致
- **新 Hooks（useDeferredValue、useInsertionEffect）**：分别对应优先级分层和 CSS-in-JS 注入时机

### Q28：为什么 React 需要 useSyncExternalStore？什么是 tearing（撕裂）？

**答**：并发渲染中，一次渲染可能被打断、恢复甚至重跑。如果外部 store（不在 React 状态体系内）在渲染中途被修改，被中断前后的渲染可能读到**同一个 store 的不同快照**，导致 UI 不同部分显示不一致——这就是 tearing。`useSyncExternalStore(subscribe, getSnapshot)` 的解决方式：渲染开始时锁定快照，如果渲染期间快照变了就**丢弃重渲染**，保证一次提交内的所有读取一致。这也是 Redux 等外部状态库适配 React 18 并发模式的必经改造。

---

## 附：面试答题技巧提示

- Fiber 题按「**为什么重写 → 数据结构 → 两棵树 → 两阶段 → 调度**」五步走，全程围绕「可中断」展开
- Hooks 题的钥匙是「**hook 链表 + 顺序匹配**」，几乎所有 Hooks 约束（不能条件调用、依赖数组、闭包陈旧值）都能从这一点推出来
- 性能优化题先说「重渲染是怎么发生的」（父 render 产生新元素引用），再说优化手段，显得有因果链而不是背清单
- 对比题（React vs Vue）落到「**重新执行+diff 兜底 vs 精确追踪+最小更新**」这个本质分歧，比罗列 API 差异高一档
- 手写题里主动说出「对应 React 内部的哪个机制」（如 hookIndex 对应链表游标、cleanup 对应 passive effect destroy），是加分点
