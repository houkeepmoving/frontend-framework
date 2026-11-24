# Vue2 与 Vue3 底层原理详解

> 从响应式系统、模板编译、虚拟 DOM 与 Diff 算法、组件机制四个维度，分别拆解 Vue 2 和 Vue 3 的底层实现，最后给出两者的系统性对比和常见面试题。

---

# 第一部分：Vue 2 底层原理

## 1. Vue 2 整体架构与初始化流程

Vue 2 的核心是「**响应式系统 + 模板编译 + Virtual DOM**」三件套。一次组件初始化的完整流程：

```
new Vue(options)
  └→ _init(options)
       ├→ initLifecycle / initEvents / initRender
       ├→ callHook('beforeCreate')
       ├→ initInjections / initProvide
       ├→ initState            ← 响应式系统在这里启动
       │    ├→ initProps      （props 逐个 defineReactive）
       │    ├→ initMethods
       │    ├→ initData       （data 深度递归 observe）
       │    ├→ initComputed   （每个 computed 创建 lazy watcher）
       │    └→ initWatch      （每个 watch 创建 user watcher）
       ├→ initWatch...
       └→ $mount(el)          ← 编译 + 渲染从这里开始
            ├→ 编译模板 → render 函数（完整版）或直接用 render 选项（运行时版）
            └→ mountComponent
                 ├→ new Watcher(vm, render函数, ...)   ← 渲染 Watcher
                 │    首次执行：Dep.target = 渲染watcher
                 │      → vm._render() 生成 vnode（期间触发所有数据 getter，完成依赖收集）
                 │      → vm._update(vnode) patch 到真实 DOM
                 └→ callHook('mounted')
```

**关键理解：数据的 getter 收集了「渲染 Watcher」，数据变化时 setter 通知 Watcher 重新执行 render + patch。** 整个 Vue 2 数据驱动的闭环就建立在这条链上。

---

## 2. 响应式核心：Object.defineProperty

### 2.1 Observer 类：递归观测入口

```js
// 源码位置：core/observer/index.js（简化）
export class Observer {
  constructor(value) {
    this.value = value
    this.dep = new Dep()          // 数组用：通知数组自己的 dep
    this.vmCount = 0
    // 在对象上挂 __ob__ 标记，避免重复观测（不可枚举，防止被遍历）
    def(value, '__ob__', this)

    if (Array.isArray(value)) {
      // 数组走特殊路径：不改写属性，而是替换原型上的 7 个方法
      const augment = hasProto ? protoAugment : copyAugment
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)    // 递归观测数组元素
    } else {
      this.walk(value)            // 对象：逐个属性 defineReactive
    }
  }

  walk(obj) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }
}
```

### 2.2 defineReactive：getter/setter 劫持

```js
export function defineReactive(obj, key, val, shallow) {
  const dep = new Dep()                    // 每个属性自己的依赖收集器

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) return

  const getter = property && property.get
  const setter = property && property.set

  // 关键：递归观测子值。这就是「深度响应式」的来源，也是初始化开销的来源
  let childOb = !shallow && observe(val)

  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter() {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {                    // 当前有 Watcher 正在求值
        dep.depend()                       // 收集：把 Dep.target 加进这个属性的 dep
        if (childOb) {
          childOb.dep.depend()             // 子对象/数组也收集（为了 $set 和数组能触发）
          if (Array.isArray(value)) {
            dependArray(value)             // 数组内嵌套对象也要收集
          }
        }
      }
      return value
    },
    set: function reactiveSetter(newVal) {
      const value = getter ? getter.call(obj) : val
      if (newVal === value || (newVal !== newVal && value !== value)) return
      if (getter && !setter) return        // 只有 getter 的属性不可写

      if (setter) setter.call(obj, newVal)
      else val = newVal

      // 新值如果是对象，也要递归观测
      childOb = !shallow && observe(newVal)

      dep.notify()                         // 通知所有依赖这个属性的 Watcher
    }
  })
}
```

**核心要点：**

- **每个响应式属性对应一个 `Dep` 实例**——它是属性和 Watcher 之间的桥梁
- getter 里做**依赖收集**（谁在读我），setter 里做**派发更新**（我变了告诉谁）
- `childOb.dep.depend()` 容易被忽略：读取 `obj.someObj` 时，`someObj` 这个**子对象整体**也被收集了依赖，这正是 `this.someObj.newKey = 1` 能通过 `$set` 触发更新、以及数组方法能触发更新的基础

### 2.3 依赖收集的完整链路：Dep、Watcher、Dep.target

```js
// Dep：依赖收集器（简化）
export default class Dep {
  static target: ?Watcher      // 全局唯一的「当前正在求值的 Watcher」
  id: number
  subs: Array<Watcher>         // 订阅者列表

  addSub(sub) { this.subs.push(sub) }
  depend() {
    if (Dep.target) {
      Dep.target.addDep(this)  // 反向调用，去重由 Watcher 侧完成
    }
  }
  notify() {
    const subs = this.subs.slice()
    for (let i = 0; i < subs.length; i++) subs[i].update()
  }
}

// Watcher：订阅者（简化）
export default class Watcher {
  constructor(vm, expOrFn, cb, options) {
    this.vm = vm
    this.getter = expOrFn          // 渲染watcher：render 函数；用户watcher：取值表达式
    this.deps = []                 // 我订阅的所有 dep（用于去重和移除）
    this.newDeps = []              // 本轮求值新收集的 dep
    this.dirty = this.lazy         // computed 专用：是否需要重新计算
    this.value = this.lazy ? undefined : this.get()
  }

  get() {
    pushTarget(this)               // ★ Dep.target = this（入栈）
    let value
    try {
      value = this.getter.call(this.vm, this.vm)   // ★ 求值，期间触发所有 getter
    } finally {
      popTarget()                  // ★ Dep.target 恢复为上一个（出栈）
      this.cleanupDeps()           // 清理本轮没被收集的旧依赖（处理 v-if 切换）
    }
    return value
  }

  addDep(dep: Dep) {
    if (!this.newDepIds.has(dep.id)) {   // 同一次求值中同一个 dep 只收集一次
      this.newDepIds.add(dep.id)
      this.newDeps.push(dep)
      if (!this.depIds.has(dep.id)) {
        dep.addSub(this)                 // 真正建立双向关系
        this.depIds.add(dep.id)
      }
    }
  }

  update() { ... }  // 被 dep.notify() 调用
}
```

**为什么 Dep.target 要用栈（pushTarget/popTarget）？**

因为存在嵌套求值：渲染过程中可能触发 computed 求值，computed 求值期间 `Dep.target` 必须临时变成 computed watcher，否则 computed 内部的数据会把渲染 watcher 当成依赖（渲染 watcher 对 computed 内部细节产生多余依赖，且 computed 的缓存机制会失效）。求值结束 pop 恢复，保证依赖收集「谁在求值就归谁」。

**cleanupDeps 解决什么问题？**

```html
<div v-if="show">{{ message }}</div>
```

当 `show` 从 true 变 false，模板里不再渲染 `message`，渲染 watcher 就不该再依赖 `message`。cleanupDeps 每轮求值后对比 `deps`（旧）和 `newDeps`（新），把不再使用的 dep 里的自己移除——否则会出现「已经不显示的数据变化仍触发重渲染」的浪费。

### 2.4 Watcher 的四种类型

| 类型 | 创建时机 | getter | 特点 |
|---|---|---|---|
| **渲染 Watcher** | `$mount` 时，每个组件实例一个 | render 函数 | `vm._watcher`，update 时触发重新渲染 |
| **用户 Watcher** | `initWatch` / `vm.$watch` | 表达式或函数 | 支持 `deep`、`immediate`、`sync` 选项 |
| **计算 Watcher** | `initComputed` | computed 的 getter 函数 | `lazy: true`，初始不执行，靠 `dirty` 控制缓存 |
| **dep Watcher** | `$watch` 配合组件销毁 | — | 用于在组件销毁时清理（vm.$watch 内部细节） |

---

## 3. 派发更新：数据变了之后发生什么

```js
// Watcher.update 的核心逻辑（简化）
update() {
  if (this.lazy) {
    this.dirty = true          // computed：只标记脏，等下次读取再算（懒执行）
  } else if (this.sync) {
    this.run()                 // sync: true 的用户 watcher，同步执行
  } else {
    queueWatcher(this)         // ★ 常规路径：进队列，异步批量执行
  }
}

// queueWatcher：批量去重 + nextTick 调度
export function queueWatcher(watcher: Watcher) {
  if (!flushing && !watcher.id && ...) { ... }
  const id = watcher.id
  if (has[id] == null) {       // 按 id 去重：同一个 watcher 一轮只进队列一次
    has[id] = true
    queue.push(watcher)
    if (!waiting) {
      waiting = true
      nextTick(flushSchedulerQueue)   // 异步刷新
    }
  }
  // flushing 期间可能插入新的 watcher（保证父组件在子组件之前更新）
  if (!has[id] && watcher.id > queue[len-1].id ...) { ... }
}

function flushSchedulerQueue() {
  queue.sort((a, b) => a.id - b.id)  // ★ 按 id 升序：创建顺序 = 父先于子
  for (const watcher of queue) {
    watcher.run()                    // 执行 getter：渲染watcher → 重新render+patch
  }
  resetSchedulerState()
}
```

**两个高频设计点：**

1. **按 id 排序的原因**：watcher 创建顺序就是组件创建顺序（父先创建）。父组件更新可能改变子组件（如 v-if），先更新父组件，子组件 watcher 若已失效可在 `run` 时被跳过（`if (watcher === this.vm._watcher && watcher.vm._isDestroyed)`），避免无效更新和更新已销毁组件。
2. **去重的原因**：一次同步代码里改同一个数据 10 次，渲染 watcher 只会进一次队列、只重渲染一次。

---

## 4. Vue 2 响应式的两大先天缺陷及 $set/$delete

**缺陷 1：无法侦测「新增/删除属性」**

`Object.defineProperty` 只能劫持**已存在**的属性。Vue 初始化时递归 walk 了所有 key，运行时新加的 key 没有 getter/setter，不具备响应式。

```js
this.obj.newKey = 1        // ❌ 不触发更新（没有 setter）
delete this.obj.existing   // ❌ 不触发更新（setter 不会因 delete 被调用）
```

所以需要 `Vue.set` / `this.$set`：

```js
export function set(target, key, val) {
  // 数组：splice 是被拦截过的方法，天然触发更新
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  // 已有属性：直接赋值（已有响应式）
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  const ob = target.__ob__
  if (!ob) { target[key] = val; return val }   // 非响应式对象，直接赋值
  defineReactive(ob.value, key, val)           // ★ 新属性手动 defineReactive
  ob.dep.notify()                              // ★ 通知「子对象整体」的 dep
  return val
}
```

注意 `ob.dep.notify()` 通知的是**父对象整体的 dep**——这就是 2.2 节里 `childOb.dep.depend()` 收集依赖的意义闭环。

**缺陷 2：数组无法通过下标和 length 触发更新**

给数组每个索引做 defineReactive 成本高（大数组初始化慢），Vue 2 选择了**方法拦截**方案：

```js
// 数组方法拦截
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)
const methodsToPatch = [
  'push', 'pop', 'shift', 'unshift',
  'splice', 'sort', 'reverse'          // 恰好是能改变数组自身的 7 个方法
]

methodsToPatch.forEach(function (method) {
  const original = arrayProto[method]
  def(arrayMethods, method, function (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push': case 'unshift': inserted = args; break
      case 'splice': inserted = args.slice(2); break
    }
    if (inserted) observeArray(inserted)   // 新插入的元素也要观测
    ob.dep.notify()                        // 触发更新
    return result
  })
})

// arr.push(1) 能触发更新的原因：
// arr.__proto__ 被指向 arrayMethods（protoAugment）
// arr.push 先沿原型链找到 arrayMethods 上的拦截方法
```

所以：`arr[0] = x` 和 `arr.length = 0` 不触发更新，需要用 `Vue.set(arr, 0, x)` 或 `arr.splice()`。

**其他局限：**
- 初始化时递归遍历整个 data 树，**大对象初始化开销大**
- 属性必须在 data 中预声明
- 无法监听 Map/Set，无法整体替换的对象内部结构等

---

## 5. Vue 2 的 Virtual DOM 与 Diff 算法（双端 Diff）

### 5.1 VNode

每次 render 产生一棵 VNode 树。Vue 2 的 VNode 是 class，包含 `tag / data / children / text / elm(真实DOM引用) / key / componentOptions` 等。有两点特殊：

- **组件 VNode**：`componentOptions` 保存组件信息，patch 时遇到组件 vnode 走 `createComponent/updateChildComponent` 路径
- **注释节点、文本节点、空节点**都是特殊 VNode，用于占位

### 5.2 patch 与 updateChildren：双端 Diff

新旧 vnode 对比入口 `patch(oldVnode, vnode)`：

```
sameVnode(oldVnode, vnode) ？
  ├→ true  → patchVnode：复用 elm，递归对比
  └→ false → createElm 新建，替换旧的（销毁整棵旧子树）
```

`sameVnode` 判断条件：`key` 相同 && `tag` 相同 && `isComment` 相同 && data 里的 `input type` 相同等。

**`updateChildren` 是 Vue 2 diff 的核心，四个指针 + 四种命中：**

```
oldStartIdx → [旧列表]  ← oldEndIdx
newStartIdx → [新列表]  ← newEndIdx
```

循环比较，每轮按顺序尝试四种匹配：

1. `oldStartVnode vs newStartVnode`（旧头-新头）→ 命中：patch，两指针后移
2. `oldEndVnode vs newEndVnode`（旧尾-新尾）→ 命中：patch，两指针前移
3. `oldStartVnode vs newEndVnode`（旧头-新尾）→ 命中：patch，把旧头节点**移动到旧尾之后**（对应真实 DOM `insertBefore`），指针收敛
4. `oldEndVnode vs newStartVnode`（旧尾-新头）→ 命中：patch，把旧尾节点**移动到旧头之前**，指针收敛

四种都不命中 → **暴力查找**：用新 startVnode 的 key 去 `oldVnode` 里 map 查找（有 key 用 key map，无 key 遍历对比）：

- 找到 → patchVnode，并把这个节点移动到最前（`canMove && nodeOps.insertBefore(...)`）
- 找不到 → `createElm` 新建插入

循环结束的收尾：

- `oldStartIdx > oldEndIdx`（旧列表先遍历完）→ 新列表剩余的**批量插入**
- `newStartIdx > newEndIdx`（新列表先遍历完）→ 旧列表剩余的**批量删除**

**双端 diff 的复杂度**：最好 O(n)（头尾对齐的追加/收缩场景），最坏 O(n²)（完全乱序）。Vue 2 没有做最长递增子序列优化，DOM 移动次数不是最优。

### 5.3 patchVnode 的处理顺序

```
patchVnode(oldVnode, vnode)
  ├→ 都是静态节点且 key 相同 → 直接复用，跳过（编译优化：staticRoot）
  ├→ 有 text 且 text 不变 → 跳过
  ├→ 都是文本 → 更新 text
  └→ 都有 children：
       ├→ updateChildren(oldCh, ch)   ← 核心
       └→ 同时更新 props/data（触发组件的 beforeUpdate）
```

---

## 6. Vue 2 模板编译器

编译三阶段（`vue-template-compiler` / 完整版 Vue 里的 `compileToFunctions`）：

```
template 字符串
  ├→ parse        ：用正则逐段解析，产出 AST（抽象语法树）
  │                 处理：标签、属性、指令、插值表达式、v-if/v-for 等结构化指令
  ├→ optimize     ：★ 标记静态节点
  │                 markStatic / markStaticRoots
  │                 给 AST 节点打上 static: true / staticRoot: true
  └→ generate     ：AST → render 函数字符串（with(this) 形式）
```

**optimize 的意义**：静态子树在 patchVnode 里被直接跳过（见 5.3），不参与 diff。这是 Vue 2 编译时优化的主要手段，但粒度粗——只要子树里有动态节点，整棵子树都要进入 diff。

**with(this) 的代价**：render 函数里 `msg` 实际是 `this.msg`，无法提前分析依赖，只能在运行时求值时靠 getter 收集依赖。这是 Vue 2 无法做「精准更新」的另一个原因。

---

## 7. nextTick 原理

```js
const callbacks = []         // 本轮所有回调队列
let pending = false

export function nextTick(cb, ctx) {
  callbacks.push(() => cb.call(ctx))
  if (!pending) {
    pending = true
    timerFunc()              // 异步执行 flushCallbacks
  }
}

function flushCallbacks() {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (const cb of copies) cb()
}

// timerFunc：按能力降级选择异步方案
if (typeof Promise !== 'undefined') {
  timerFunc = () => p.then(flushCallbacks)          // 优先微任务
  // 若环境不支持 Promise 且需要降级：MutationObserver（也是微任务）→ setTimeout
}
```

**核心理解：**

- `nextTick` 把回调放进一个队列，用**微任务**（Promise.then）在当前同步代码执行完后统一 flush
- 数据更新 → `queueWatcher` → `nextTick(flushSchedulerQueue)` 也走同一个微任务队列，所以「改数据后立刻读 DOM」拿到的是旧 DOM，`await nextTick()` 后拿到新 DOM
- 一次同步修改 N 个数据 → 只产生一个微任务 → **批量更新**

---

## 8. Vue 2 的 computed 与 watch 原理

### 8.1 computed：懒执行 + 缓存

```js
function initComputed(vm, computed) {
  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    watchers[key] = new Watcher(vm, getter, noop, { lazy: true })   // ★ lazy

    defineComputed(vm, key, userDef)
  }
}

function createComputedGetter(key) {
  return function computedGetter() {
    const watcher = this._computedWatchers[key]
    if (watcher.dirty) {          // ★ dirty 为 true 才真正求值
      watcher.evaluate()          // 内部 pushTarget(this) → 求值 → popTarget
    }
    if (Dep.target) {
      watcher.depend()            // ★ 把 computed 依赖的 dep「桥接」给外层 watcher
    }
    return watcher.value
  }
}
```

**关键机制「依赖桥接」**：

computed watcher 求值时，其依赖的数据收集的是 computed watcher；但渲染 watcher 读 computed 时需要「数据变了 → 触发渲染」。`watcher.depend()` 把 computed watcher 的 deps **追加到当前 Dep.target（渲染watcher）的依赖里**，实现：数据 → (computed dep) → 渲染 watcher 的传递。

**dirty 什么时候变 true？** 数据变化 notify → computed watcher.update()（lazy 分支）→ `this.dirty = true`，但**不立即求值**；下次模板读取 computed 时才发现 dirty 并重新计算，然后置 false。这就是「缓存」。

Vue 2 的已知细节：computed watcher 被激活时（`evaluate` 后）如果又有依赖变化，`watcher.dirty = true` 的同时因为桥接，渲染 watcher 也在 dep 列表里，会被 notify——两者的联动有时产生「computed 不脏但渲染被触发」的额外开销，Vue 3 针对此做了重写。

### 8.2 watch：立即/深度/异步

- `immediate`：创建 watcher 后立刻执行一次 cb
- `deep`：`traverse(value)`——递归遍历对象所有属性触发 getter，把深层数据全部收集为依赖
- 默认异步：进 scheduler 队列；`sync: true` 同步执行

---

## 9. Vue 2 组件化与生命周期

- **组件创建**：Vue.extend → 子类构造器；`createComponent` 在 patch 中创建占位 vnode，内部 `init` hook 实例化组件并 `$mount` 到占位元素
- **组件也是 Watcher**：每个组件实例对应一个渲染 watcher → 组件级别的更新隔离（父组件重渲染时，若子组件 props 没变，子组件通过 `shouldComponentUpdate` 类似的 props 浅对比跳过）
- **生命周期与 patch 钩子**：`beforeCreate/created` 在 `_init`；`beforeMount/mounted` 在 `$mount`；`beforeUpdate/updated` 在 patch 前后；`beforeDestroy/destroyed` 在 `$destroy`（拆 watcher、解绑事件、触发子组件递归销毁）

---

# 第二部分：Vue 3 底层原理

## 10. 响应式核心：Proxy + effect

### 10.1 为什么从 defineProperty 换成 Proxy

| 维度 | Object.defineProperty（Vue2） | Proxy（Vue3） |
|---|---|---|
| 新增/删除属性 | ❌ 需要 $set/$delete | ✅ 拦截 set/deleteProperty |
| 数组索引和 length | ❌ 方法拦截绕过 | ✅ 统一拦截 |
| 嵌套对象 | 初始化时递归，大对象开销大 | **惰性**：访问到子对象时才代理 |
| Map/Set | ❌ 不支持 | ✅ 拦截 get/set/has/delete |
| 整体兼容性 | IE9+ | 不支持 IE（Proxy 无法 polyfill） |

### 10.2 reactive 的实现结构

```js
// reactive.js（简化）
const proxyMap = new WeakMap()   // 缓存：原始对象 → 代理对象，避免重复代理

export function reactive(target) {
  if (target && (target[ReactiveFlags.IS_REACTIVE] || !isObservableType(target))) {
    return target                 // 已是代理 或 类型不可代理，直接返回
  }
  const existing = proxyMap.get(target)
  if (existing) return existing
  const proxy = new Proxy(target, mutableHandlers)
  proxyMap.set(target, proxy)
  return proxy
}

export const mutableHandlers = {
  get: createGetter(),
  set: createSetter(),
  deleteProperty, has, ownKeys     // ★ 拦截 delete / in / Object.keys
}
```

**get 拦截器（track 所在）**：

```js
function createGetter(isReadonly = false, shallow = false) {
  return function get(target, key, receiver) {
    if (key === ReactiveFlags.IS_REACTIVE) return !isReadonly
    if (key === ReactiveFlags.RAW) return target    // toRaw 用
    // ...

    const res = Reflect.get(target, key, receiver)

    track(target, key)                              // ★ 依赖收集

    if (isObject(res)) {
      // ★ 惰性代理：访问到子对象时才 reactive(res)，不做初始化递归
      return isReadonly ? readonly(res) : reactive(res)
    }
    return res
  }
}
```

**set 拦截器（trigger 所在）**：

```js
function createSetter() {
  return function set(target, key, value, receiver) {
    const oldValue = target[key]
    const result = Reflect.set(target, key, value, receiver)  // ★ 先改数据
    if (target === toRaw(receiver)) {
      if (hasChanged(value, oldValue)) {
        trigger(target, key, value)                           // ★ 再触发更新
      }
    }
    return result
  }
}
```

### 10.3 track / trigger 与依赖存储结构

```js
// 全局依赖存储：三层嵌套结构
const targetMap = new WeakMap()
// 结构：WeakMap<原始对象, Map<属性key, Set<effect>>>
//          obj        →   key    →  deps

function track(target, key) {
  if (!activeEffect) return                  // 没有正在运行的 effect
  let depsMap = targetMap.get(target)
  if (!depsMap) targetMap.set(target, (depsMap = new Map()))
  let dep = depsMap.get(key)
  if (!dep) depsMap.set(key, (dep = new Set()))
  trackEffects(dep)
  dep.add(activeEffect)
  activeEffect.deps.push(dep)                // 反向记录，cleanup 用
}

function trigger(target, key) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return
  const deps = depsMap.get(key)
  triggerEffects(deps)
}
```

**WeakMap 的意义**：key 是原始对象（弱引用），对象被销毁后依赖表自动被 GC，无需手动清理。

### 10.4 effect：Vue 3 的「Watcher」

```js
export function effect(fn, options = {}) {
  const _effect = new ReactiveEffect(fn)    // 包装成 ReactiveEffect 实例
  if (!options.lazy) _effect.run()          // 默认立即执行一次
  const runner = _effect.run.bind(_effect)
  runner.effect = _effect
  return runner
}

class ReactiveEffect {
  fn            // 副作用函数：渲染函数 / watch 回调 / computed getter
  active = true
  deps = []     // 我依赖的所有 dep（Set）
  scheduler     // ★ 调度器：触发更新时如果定义了 scheduler 就调用它而不是 fn
  onStop / stop()

  run() {
    if (!this.active) return this.fn()   // 停止后直接执行不收集
    // 与 Vue2 类似的 target 栈机制：
    if (effectStack.indexOf(this) === -1) {   // 防止递归 self-trigger
      cleanupEffect(this)                     // ★ 先清空旧依赖
      activeEffect = this
      try {
        this.fn()                             // 执行：期间 track 收集
      } finally {
        activeEffect = effectStack.pop()
      }
    }
  }

  trigger() {
    if (this.scheduler) this.scheduler(this)  // 有调度器走调度
    else this.run()
  }
}
```

**cleanup（依赖清理）与 Vue 2 的 cleanupDeps 是同一问题的不同解法**：分支切换（v-if）后旧的依赖不再需要，Vue 2 用 diff 两轮列表，Vue 3 干脆每次 run 前把 effect 从所有 dep 里移除，重新收集——实现简单但有额外开销，于是 Vue 3.4+ 优化为「**双层 dep**（track 期间只记录链表头尾，无变化时跳过 cleanup）」。

### 10.5 渲染 effect 与调度（对应 Vue 2 的渲染 Watcher）

每个组件的 `setup` 返回 render 时，Vue 3 创建一个渲染 effect：

```js
// componentRenderUtils.ts（简化）
const effect = new ReactiveEffect(componentUpdateFn)
effect.scheduler = () => queueJob(update)   // ★ 调度器把更新推入队列而不是立即执行

const update = effect.run.bind(effect)
update.id = instance.uid
```

`queueJob` 是 Vue 3 的批量调度队列（对应 Vue 2 的 `queueWatcher`）：

```js
const queue = []
let flushIndex = 0
let isFlushing = false

export function queueJob(job) {
  // ★ 去重条件：id 相同 或 相同 job 或 递归检测 allowRecurse
  if (!queue.includes(job) &&
      (job === currentFlushJob || !job.allowRecurse)) {
    queue.push(job)
    queueFlush()       // 微任务 flushJobs（Promise.resolve().then）
  }
}

function flushJobs() {
  isFlushing = true
  queue.sort(comparator)   // ★ 按 id 排序：父组件在子组件之前；pre 队列在渲染前
  for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
    queue[flushIndex]()    // 执行每个 job：渲染 / pre watch / pre 生命周期
  }
  queue.length = 0
  flushPostFlushCbs()      // post 队列（如 updated 钩子）
  isFlushing = false
}
```

**与 Vue 2 的主要差别**：Vue 3 的调度队列是**统一队列**，watch（pre 时机）、渲染、生命周期钩子都作为 job 进同一个队列，按 `id + pre/post` 排序执行——比 Vue 2 的双队列（watcher 队列 + user watcher 队列）更可控。

### 10.6 数组与集合类型的处理

Proxy 天然拦截索引和 length，但 Vue 3 对数组/集合做了**专门的性能优化**：

```js
// 大数组的索引访问不 track（只 track length 和迭代类方法）
if (isIntegerKey(key) && key !== 'length' && ...) {
  // 仅在 specific 情况下 track
}

// 方法拦截（get 里识别数组方法名）
['includes', 'indexOf', 'lastIndexOf'].forEach(key => {
  // 避免代理对象找不到的问题：先在代理里找，找不到去原始值里找
})
['push', 'pop', 'shift', 'unshift', 'splice'].forEach(key => {
  // 避免 track/trigger 深层递归：这些方法会读/写 length，需要暂停收集
})
```

**集合（Map/Set）**单独一套 handlers：拦截 `get/has/delete/size/clear/forEach/entries/values/keys`，`add/set/delete` 走 trigger。

### 10.7 ref 与 toRef

```js
// ref：给原始值包一层对象，因为 Proxy 只能代理对象
export function ref(value) {
  return createRef(value)
}

function createRef(rawValue, shallow = false) {
  if (isRef(rawValue)) return rawValue           // 已是 ref 直接返回
  const ref = {}
  def(ref, RefFlag, true)                        // __v_isRef 标记
  def(ref, ReactiveFlags.RAW, rawValue)          // toRaw 用
  ref._value = shallow ? rawValue : toReactive(rawValue)  // ref(对象) 内部自动 reactive
  Object.defineProperty(ref, 'value', {
    get() {
      trackRefValue(ref)                         // 依赖收集在 .value 上
      return ref._value
    },
    set(newVal) {
      const oldValue = ref._value
      ref._value = shallow ? newVal : toReactive(newVal)
      if (hasChanged(newVal, oldValue)) {
        triggerRefValue(ref)                     // 触发
      }
    }
  })
  return ref
}
```

**为什么需要 ref**：Proxy 不能代理原始值（number/string 等），所以包一层对象，通过 `.value` 存取。这也是「模板里 ref 自动解包」的实现位置（编译时或 runtime 处理 `__v_isRef`）。

---

## 11. Vue 3 编译优化：静态提升、PatchFlag、Block Tree

Vue 3 在**编译时**做大量优化，让运行时 diff 更少工作。这是 Vue 3 性能提升的核心来源之一。

### 11.1 编译产物对比

Vue 2 的 render（with(this)）：

```js
with (this) {
  return _c('div', [
    _c('span', [_v('静态文本')]),          // 静态的也进 vnode 树，参与 diff
    _c('span', [_v(String(message))])     // 动态的
  ])
}
```

Vue 3 的 render（codegen 后）：

```js
const _hoisted_1 = /*#__PURE__*/ _createElementVNode("span", null, "静态文本", -1 /* HOISTED */)

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("div", null, [
    _hoisted_1,                                  // ★ 静态提升：模块级变量，永远不重建
    _createElementVNode("span", null, _toDisplayString(_ctx.message), 1 /* TEXT */)
  ]))
}
```

### 11.2 PatchFlag：让 diff 只看标记位

`_createElementVNode` 的第四个参数是 PatchFlag，用**位图**表示这个 vnode 哪些部分是动态的：

```js
export const enum PatchFlags {
  TEXT = 1,            // 动态文本内容
  CLASS = 1 << 1,      // 动态 class
  STYLE = 1 << 2,      // 动态 style
  PROPS = 1 << 3,      // 动态非 class/style 的 props
  FULL_PROPS = 1 << 4, // 带动态 key 的 props（key 数量本身可能变）
  HYDRATE_EVENTS = 1 << 5,
  STABLE_FRAGMENT = 1 << 6,
  KEYED_FRAGMENT = 1 << 7,
  UNKEYED_FRAGMENT = 1 << 8,
  NEED_PATCH = 1 << 9,
  DYNAMIC_SLOTS = 1 << 10,
  HOISTED = -1,        // 静态提升，永不 diff
  BAIL = -2            // 放弃优化
}
```

**运行时的价值**：`patchElement` 拿到 `patchFlag` 后：

- `flag & TEXT` → 只对比 text
- `flag & CLASS` → 只对比 class
- `flag & STYLE` → 只对比 style
- `flag & PROPS` → 只对比 `dynamicProps` 数组里列出的那些 key

**不再遍历所有 props**，diff 从「盲扫全部属性」变成「按标记精准打击」。

### 11.3 Block Tree 与 dynamicChildren

这是 Vue 3 最核心的运行时优化。

```js
export function render() {
  return (_openBlock(), _createElementBlock("div", null, [
    // ... 内部所有**有动态内容的节点**（不管多深）都会被收集到 block 的 dynamicChildren 里
  ]))
}
```

**Block 的收集机制**：

- `openBlock()` 开启一个数组容器（存储在当前组件实例作用域栈上）
- `_createElementVNode` / `_createElementBlock` 执行时，若自身有动态性（patchFlag ≠ -1），把自己 push 进当前 block 的 `dynamicChildren`
- **嵌套 block**：`v-if`/`v-for`/`component`/`slot` 都会创建子 block。子 block 一旦创建，其内部动态节点收集到子 block，父 block 只收集这个子 block 本身——形成「树状分片」

**diff 时的革命性变化**：

```js
// Vue 2：diff 整棵新旧 vnode 树（含所有静态节点）
// Vue 3：有 dynamicChildren 时，**只遍历 dynamicChildren 数组**，逐个 patch
patchBlockChildren(n1.dynamicChildren, n2.dynamicChildren, ...)
```

结构化指令（v-if/v-for）引发的**结构变化**通过子 block 边界隔离：整块替换子 block，而不需要 diff 内部每个节点。

**对比总结**：

| | Vue 2 | Vue 3 |
|---|---|---|
| diff 范围 | 全树 | 只 diff dynamicChildren |
| 静态节点 | optimize 阶段标记，子树级跳过 | 编译期提升，运行时完全不碰 |
| 动态属性 | 全量遍历对比 | patchFlag 按需对比 |
| 结构变化 | 双端 diff 移动 | block 边界 + key，整块替换 |

### 11.4 cacheHandler 与事件缓存

```js
// 编译产物
_createElementVNode("button", {
  onClick: _cache[0] || (_cache[0] = (...args) => _ctx.onClick(...args))
})
```

内联函数被缓存到 `_cache`，props 对比时引用不变 → 不触发子组件的无效更新。这是 Vue 3 解决「内联箭头函数导致子组件重渲染」的编译时方案（等价于手动 useCallback/useMemo）。

### 11.5 其他编译优化

- **Tree-shaking**：全局 API 改为命名导出（`createApp`、`nextTick` 等），未使用的内置组件/指令（如 `<Transition>`、`v-model` 的 transform）不进产物
- **Fragment 支持**：编译器允许多根节点
- **SSR 优化**：编译产物含 SSR 专用的字符串拼接函数（`ssrRender`），跳过 vnode 创建
- **源码 location / sourcemap**：dev 模式给模板报错精确到行列

---

## 12. Vue 3 渲染器（Runtime）

### 12.1 shapeFlags：vnode 类型位图

```js
export const enum ShapeFlags {
  ELEMENT = 1,
  FUNCTIONAL_COMPONENT = 1 << 1,
  STATEFUL_COMPONENT = 1 << 2,
  TEXT_CHILDREN = 1 << 3,
  ARRAY_CHILDREN = 1 << 4,
  SLOTS_CHILDREN = 1 << 5,
  TELEPORT = 1 << 6,
  SUSPENSE = 1 << 7,
  COMPONENT_SHOULD_KEEP_ALIVE = 1 << 8,
  COMPONENT_KEPT_ALIVE = 1 << 9,
  COMPONENT = FUNCTIONAL_COMPONENT | STATEFUL_COMPONENT,
}
```

`patch` 的分派逻辑：

```js
const patch = (n1, n2, container, anchor, ...) => {
  if (n1 === n2) return                       // 引用相同，直接返回
  if (n1 && !isSameVNodeType(n1, n2)) {
    unmount(n1)                               // key/type 不同：卸载旧的，锚点重置
    n1 = null
  }
  if (n2.patchFlag === PatchFlags.BAILOUT) { /* full diff */ }
  const { type, ref, shapeFlag } = n2
  switch (type) {
    case Text:      processText(n1, n2, container, anchor); break
    case Comment:   processCommentNode(...); break
    case Static:    /* 静态节点整块复用 */; break
    case Fragment:  processFragment(...); break
    default:
      if (shapeFlag & ShapeFlags.ELEMENT) processElement(n1, n2, container, anchor, ...)
      else if (shapeFlag & ShapeFlags.TELEPORT) processTeleport(...)
      else if (shapeFlag & ShapeFlags.SUSPENSE) processSuspense(...)
      else if (shapeFlag & ShapeFlags.COMPONENT) processComponent(n1, n2, ...)
  }
}
```

### 12.2 processElement 的关键路径

- **mountElement**：`hostCreateElement` → 逐个 set 属性 → `hostInsert`
- **patchElement**：处理 full diff 或按 patchFlag 精准 diff
- **patchChildren** 分四种组合：

```
旧文本 + 新文本 → 直接 setText
旧文本 + 新数组 → 清空文本，mount 新数组
旧数组 + 新文本 → unmount 旧数组，set 文本
旧数组 + 新数组 → ★ 核心路径 diffArrayChildren
```

---

## 13. Vue 3 的 Diff：快速 Diff（预处理 + 最长递增子序列）

Vue 3 在「有 key 的列表 diff」中使用了一套预处理 + LIS 的方案（参考 ivi / inferno 的思路），**核心是先处理两端相同的部分，把真正需要移动的情况收窄，再用最长递增子序列算出最少的移动次数**。

### 13.1 流程分解

```js
const patchKeyedChildren = (c1, c2, container, ...) => {
  let i = 0, l2 = c2.length
  let e1 = c1.length - 1, e2 = l2 - 1

  // ① 左端预处理：从头开始，相同 type+key 的直接 patch
  while (i <= e1 && i <= e2) {
    if (isSameVNodeType(c1[i], c2[i])) {
      patch(c1[i], c2[i], ...)
    } else break
    i++
  }

  // ② 右端预处理：从尾开始，相同的直接 patch
  while (i <= e1 && i <= e2) {
    if (isSameVNodeType(c1[e1], c2[e2])) {
      patch(c1[e1], c2[e2], ...)
    } else break
    e1--; e2--
  }

  // ③ 情况1：旧的遍历完，新的还有 → 批量新增
  if (i > e1) {
    while (i <= e2) {
      patch(null, c2[i], container, anchor)   // 挂到正确位置
      i++
    }
  }
  // ④ 情况2：新的遍历完，旧的还有 → 批量卸载
  else if (i > e2) {
    while (i <= e1) {
      unmount(c1[i])
      i++
    }
  }
  // ⑤ 乱序情况：核心算法
  else {
    const s1 = i, s2 = i
    const keyToNewIndexMap = new Map()
    // ⑤a 新列表的 [s2, e2] 建 key → index 映射
    for (let j = s2; j <= e2; j++) keyToNewIndexMap.set(c2[j].key, j)

    let patched = 0
    const toBePatched = e2 - s2 + 1
    let moved = false
    let maxNewIndexSoFar = 0
    const newIndexToOldIndexMap = new Array(toBePatched).fill(0)
    // 0 表示「旧列表里没有」→ 需要新建

    // ⑤b 遍历旧列表的 [s1, e1]
    for (let j = s1; j <= e1; j++) {
      const prevChild = c1[j]
      if (patched >= toBePatched) { unmount(prevChild); continue }
      let newIndex = keyToNewIndexMap.get(prevChild.key)
      if (newIndex === undefined) {
        unmount(prevChild)                    // 新列表里没有 → 删除
      } else {
        newIndexToOldIndexMap[newIndex - s2] = j + 1   // 记录旧索引（+1 区分 0）
        if (newIndex >= maxNewIndexSoFar) {
          maxNewIndexSoFar = newIndex         // 新索引递增，说明没移动
        } else {
          moved = true                        // 新索引变小了 → 有移动
        }
        patch(prevChild, c2[newIndex], ...)   // 就地复用
        patched++
      }
    }

    // ⑤c 计算最长递增子序列（LIS）
    const increasingNewIndexSequence = moved
      ? getSequence(newIndexToOldIndexMap)   // 返回「新索引序列里递增的子序列」的下标数组
      : EMPTY_ARR

    let j = increasingNewIndexSequence.length - 1
    // ⑤d 从后往前遍历新列表的 [s2, e2]，插入/移动/新建
    for (let j2 = toBePatched - 1; j2 >= 0; j2--) {
      const nextIndex = s2 + j2
      const nextChild = c2[nextIndex]
      const anchor = nextIndex + 1 < l2 ? c2[nextIndex + 1].el : parentAnchor
      if (newIndexToOldIndexMap[j2] === 0) {
        patch(null, nextChild, container, anchor, ...)   // 新建
      } else if (moved) {
        if (j < 0 || j2 !== increasingNewIndexSequence[j]) {
          move(nextChild, container, anchor, MoveType.REORDER)  // 移动
        } else {
          j--                                 // 在 LIS 里 → 不动
        }
      }
    }
  }
}
```

### 13.2 为什么用 LIS

- LIS 的结果代表「**这些节点在两个序列中的相对顺序都没变，不需要移动**」
- 让不在 LIS 里的节点移动，移动次数最少
- 例：`旧 [1,2,3,4,5]` → `新 [3,1,4,5,2]`，`newIndexToOldIndexMap = [2,0,3,4,1]`（1-based），LIS 为 `[3,4]`（对应旧索引 3、4），这两个节点不动，其余 3 个移动

**复杂度**：预处理 O(n)，建 Map O(n)，LIS O(n log n)。**总 O(n log n)**，优于 Vue 2 双端 diff 的最坏 O(n²)。

### 13.3 getSequence：贪心 + 二分 + 回溯

```js
function getSequence(arr) {
  const p = arr.slice()                  // 前驱索引表（回溯用）
  const result = [0]                     // 当前最长子序列的索引栈
  let resultLen = 1
  for (let i = 0; i < arr.length; i++) {
    const u = arr[i]
    if (u === 0) continue                // 0 表示新建，跳过
    const j = result[resultLen - 1]
    if (arr[j] < u) {
      p[i] = j
      result.push(i)                     // 直接递增，入栈
      resultLen++
    } else {
      // ★ 二分查找：找第一个 >= u 的位置，替换之
      let l = 0, r = resultLen - 1, mid
      while (l < r) {
        mid = (l + r) >> 1
        if (arr[result[mid]] < u) l = mid + 1
        else r = mid
      }
      if (u < arr[result[l]]) {
        if (l > 0) p[i] = result[l - 1]
        result[l] = i                    // 替换（数组不是真 LIS，只保证长度正确）
      }
    }
  }
  // 回溯：从栈顶往前，用 p 恢复真实的 LIS
  let len = resultLen
  const res = new Array(len)
  while (len-- > 0) {
    res[len] = result[len]
    result[len] = p[result[len]]
  }
  return res
}
```

**注意**：这个算法返回的是「**长度**正确的 LIS 的**索引**」（通过回溯 p 数组恢复），不是贪心栈本身（栈内容可能不是真正的 LIS）。

---

## 14. Vue 3 组件机制

### 14.1 setup 的执行时机与原理

```
createComponent(vnode)
  └→ createComponentInstance(instance)
  └→ setupComponent(instance)
       ├→ initProps       （props 也是响应式的，浅层 reactive）
       ├→ initSlots
       └→ setupStatefulComponent
            └→ setup(instance.props, setupContext)  ← ★ 用户 setup 在这里被调用
  └→ setupRenderEffect(instance, vnode, container)
       └→ 创建渲染 effect → 绑定 update job → queueJob/update()
```

**setup 的返回值处理**：

- 返回函数 → 当作 render 函数（Options API 兼容）
- 返回对象 → 用 `proxyRefs` 包一层（访问时自动 `.value` 解包）

**为什么 ref 在模板里不用 .value**：渲染函数读 `xxx` 时通过 instance proxy 的 get 拦截，检测到 `__v_isRef` 就返回 `.value`。

### 14.2 生命周期

| Vue 2 | Vue 3 (setup 内) |
|---|---|
| beforeCreate | 不需要（setup 本身） |
| created | 不需要（setup 本身） |
| beforeMount | onBeforeMount |
| mounted | onMounted |
| beforeUpdate | onBeforeUpdate |
| updated | onUpdated |
| beforeUnmount | onBeforeUnmount |
| unmounted | onUnmounted |
| errorCaptured | errorCaptured |
| activated / deactivated | onActivated / onDeactivated（KeepAlive） |
| — | onRenderTracked / onRenderTriggered（调试用） |

注册原理：`injectHook` 把回调包装后推入 `instance[type]` 数组，patch 时机按 `pre/post` 分组（如 `onBeforeUpdate` 是 pre，`onUpdated` 是 post，通过 queuePostFlushCb 推迟到 flush 后）。

### 14.3 Teleport / Suspense / KeepAlive

- **Teleport**：vnode 带 `shapeFlag.TELEPORT`，`processTeleport` 直接把子树挂到 `to` 指定的目标容器，更新/卸载时操作目标容器。内部通过 targetAnchor 占位
- **Suspense**：子树渲染 throw promise → Suspense 捕获，显示 fallback；resolve 后重新渲染真实内容。支持嵌套和 `onFallback` 过渡
- **KeepAlive**：不是真正的组件，是内部组件 `KeepAliveImpl`。通过 `shapeFlag.COMPONENT_SHOULD_KEEP_ALIVE` 标记，卸载时**不真正销毁**，而是移到一个 `Map<key, vnode>` 缓存池，DOM 移到隐藏容器；激活时从缓存取 vnode 重新挂载。配合 activated/deactivated 钩子

---

## 15. Composition API 的设计动机（从原理角度）

- **Options API 的逻辑组织问题**：同一功能的代码分散在 data/methods/computed/watch 里，复杂组件阅读时反复跳转；mixin 存在命名冲突和数据来源不透明问题
- **Composition API**：逻辑按功能聚合到一个函数（composable），依赖通过参数显式传递，类型推断更好
- **原理层面**：setup 在组件实例创建时调用一次（不是响应式作用域），返回的内容绑定到实例；`ref/reactive/computed/watch` 是独立于组件的响应式原语（effect + track/trigger），这让「逻辑复用」可以脱离组件生命周期进行

---

# 第三部分：Vue 2 vs Vue 3 系统性对比

## 16. 总对比表

| 维度 | Vue 2 | Vue 3 |
|---|---|---|
| **响应式实现** | Object.defineProperty 递归遍历 | Proxy + Reflect，惰性代理 |
| **新增/删除属性** | 需要 $set / $delete | 原生支持（set/deleteProperty 拦截） |
| **数组监听** | 重写 7 个方法 + 6 个数组操作陷阱 | 原生支持索引/length |
| **初始化开销** | 递归深度遍历全部 data | 首次访问才代理子层 |
| **依赖收集** | Dep + Watcher 双向列表，Dep.target 单例 + 栈 | targetMap（WeakMap→Map→Set）+ activeEffect |
| **依赖清理** | cleanupDeps（新旧 diff） | cleanup（每次 run 清空重收，3.4 优化为链表增量） |
| **渲染单位** | 渲染 Watcher | 渲染 effect + scheduler |
| **更新队列** | queueWatcher（watcher 专用队列） | queueJob（渲染/watch/生命周期统一队列） |
| **编译产物** | with(this) 的 render 函数 | 无 with，模块级静态提升 |
| **静态内容** | optimize 标记，子树级跳过 | hoistStatic 提升 + PatchFlag -1 |
| **动态属性 diff** | 全量遍历 props | patchFlag 位标记精准 diff |
| **diff 算法** | 双端 diff（四指针） | 预处理 + 快速 diff（LIS）O(n log n) |
| **结构变化** | diff 过程中处理 | Block Tree 分片，整块替换 |
| **事件处理** | 每次渲染重建内联函数 | cacheHandler 缓存 |
| **组件定义** | Options API | Composition API + Options API 兼容 |
| **根节点** | 单根 | Fragment 多根 |
| **全局 API** | Vue.use / Vue.mixin（全局污染） | createApp().use()（实例隔离） |
| **逻辑复用** | mixin（来源不透明） | composables（显式依赖） |
| **Tree-shaking** | 差（Vue 构造器整体引入） | 好（命名导出，未用特性可摇掉） |
| **TypeScript** | 支持，类型弱 | 源码 TS 编写，类型完整 |
| **性能模型** | 运行时依赖收集（任何路径读数据都会收集） | 编译时标记 + 运行时收集结合，收集范围更小 |

## 17. 性能提升的本质

Vue 3 的性能提升不是单一因素，而是**三层叠加**：

1. **编译时**：静态提升让静态内容零开销；PatchFlag 让属性 diff 精准化；Block Tree 让结构 diff 只看动态部分
2. **运行时**：Proxy 惰性代理降低初始化成本；LIS diff 降低移动成本；统一调度降低批量更新开销
3. **构建时**：Tree-shaking 让产物体积最小化

**一个重要的心智模型转变**：Vue 2 是「运行时尽量智能」，Vue 3 是「编译时能算出的绝不留给运行时」——这是整个前端框架演化的大方向（Svelte/Solid 走得更远）。

---

# 第四部分：常见面试题

## A. 响应式原理类

### Q1：Vue 2 响应式的核心原理是什么？

**答**：基于 `Object.defineProperty` 劫持属性的 getter/setter。初始化时递归遍历 data（`observe` + `defineReactive`），每个属性创建一个 Dep。组件渲染时创建渲染 Watcher，求值过程中触发数据 getter，将 Watcher 收集进对应属性的 Dep（依赖收集）；数据修改时触发 setter，调用 `dep.notify()` 通知所有订阅的 Watcher（派发更新），渲染 Watcher 收到通知后不立即渲染，而是通过 `queueWatcher` 去重后进入队列，`nextTick` 异步批量刷新。

### Q2：为什么 Vue 3 用 Proxy 替代 Object.defineProperty？

**答**：四个原因：

1. defineProperty 只能劫持**已存在的属性**，无法侦测新增/删除（Vue 2 需要 `$set`/`$delete`），Proxy 拦截的是对象整体操作（`set`/`deleteProperty`/`ownKeys`），天然支持
2. 数组索引赋值和 `length` 修改无法被 defineProperty 监听，Vue 2 靠重写 7 个数组方法绕过；Proxy 原生支持
3. Vue 2 初始化时要**递归遍历**整个 data 树，大对象开销大；Vue 3 的 Proxy 是**惰性**的，访问到嵌套对象时才递归代理
4. Proxy 还能代理 Map/Set、拦截 `in` 操作符等，能力更完整。代价是不支持 IE11（Proxy 无法 polyfill）

### Q3：Vue 2 数组是怎么实现响应式的？为什么 `arr[0] = 1` 不触发更新？

**答**：Vue 2 没有给数组每个索引做 defineReactive（成本高），而是**重写数组的原型**：让数组实例的 `__proto__` 指向一个以 `Array.prototype` 为原型的新对象 `arrayMethods`，在其上拦截 `push/pop/shift/unshift/splice/sort/reverse` 这 7 个会改变数组的方法，方法内部先执行原生逻辑，再对新增元素进行观测、并调用 `ob.dep.notify()` 触发更新。`arr[0] = 1` 是普通属性赋值，绕过了方法拦截也没有索引的 setter，所以不触发；要用 `Vue.set(arr, 0, 1)` 或 `arr.splice(0, 1, 1)`。Vue 3 中 Proxy 直接拦截索引和 length，不再有这个问题。

### Q4：Vue 2 的 Dep.target 为什么是全局单例？为什么要用栈？

**答**：依赖收集需要知道「当前正在求值的 Watcher 是谁」，JS 没有执行上下文的显式访问手段，所以用模块级变量 `Dep.target` 标记。用栈是因为存在**嵌套求值**：渲染 Watcher 执行 render 时可能触发 computed watcher 的求值，computed 求值期间 target 要临时切换成 computed watcher，求值完成出栈恢复为渲染 watcher，保证依赖归对「正在求值的人」。

### Q5：Vue 2 的依赖是怎么清理的？Vue 3 呢？

**答**：都为了处理「分支切换后旧依赖不再需要」的问题。Vue 2 在 Watcher 上维护 `deps`（旧）和 `newDeps`（本轮），每轮求值结束后 diff 两轮收集结果，把不再出现的 dep 从中移除自己。Vue 3 更简单粗暴：effect 每次 run 前先把自己从所有已记录的 dep（`effect.deps`）里移除（cleanup），再重新收集；Vue 3.4 之后优化为基于链表的增量版本，依赖无变化时跳过 cleanup，减少开销。

### Q6：ref 和 reactive 的区别？为什么需要 ref？

**答**：`reactive` 基于 Proxy 代理**对象**，直接访问属性；`ref` 是一个带 `value` 属性的对象，通过 defineProperty 劫持 `.value` 的 get/set 实现响应式。需要 ref 是因为 **Proxy 无法代理原始值**（number/string/boolean），要响应式化只能包一层对象；此外 ref 内部如果包了对象会自动调用 reactive。使用区别：reactive 有解构丢失响应式的问题（解构后变成普通值），ref 没有；模板里 ref 自动解包，script 里需要 `.value`。

### Q7：Vue 3 的响应式数据解构后为什么丢失响应式？怎么解决？

**答**：`reactive` 返回的是 Proxy 对象，解构 `const { a } = obj` 相当于执行了一次 `obj.a` 的 get，拿到的是**原始值**，后续读取不再经过 Proxy，track/trigger 都失效。解法：`toRefs` 把对象每个属性都转成 ref（保持 getter/setter 的引用），或 Vue 3.5 的 `destructuring` 响应式 Props 解构（编译器处理）。

## B. 编译与 Diff 类

### Q8：Vue 3 编译做了哪些优化？

**答**：五个核心优化：

1. **静态提升（hoistStatic）**：静态节点提升到 render 函数外成为模块级常量，每次渲染不重新创建 vnode
2. **PatchFlag**：编译时给动态 vnode 打上位标记（TEXT/CLASS/STYLE/PROPS 等），运行时 diff 只按标记对比动态部分，不再全量遍历 props
3. **Block Tree / dynamicChildren**：以 block 为单位收集所有动态后代节点，diff 时只遍历 `dynamicChildren` 数组，静态节点完全不参与；v-if/v-for/component/slot 创建子 block，结构变化整块替换
4. **cacheHandler**：内联事件函数缓存到 `_cache`，props 引用稳定，避免子组件无效重渲染
5. **Tree-shaking**：全局 API 命名导出，未使用的内置组件/指令不进产物

### Q9：Vue 2 和 Vue 3 的 Diff 算法分别是什么？

**答**：

- **Vue 2：双端 diff**。维护新旧列表各两个指针（头/尾），每轮从四种组合（旧头-新头、旧尾-新尾、旧头-新尾、旧尾-新头）依次尝试匹配，命中就 patch 并收敛指针；四种都不命中时用 key 查 map 找可复用节点（找到则移动到头部），否则新建。最后处理剩余节点的批量插入/删除。复杂度最好 O(n)，最坏 O(n²)
- **Vue 3：快速 diff（预处理 + LIS）**。先从**头到尾**和**从尾到头**两轮预处理，直接 patch 相同的节点，把问题收窄到中间乱序部分；乱序部分建立 `key → newIndex` 的 Map，遍历旧列表标记复用/删除，用 `newIndexToOldIndexMap` 计算新索引序列的**最长递增子序列（LIS）**——LIS 内的节点相对顺序未变不需要移动，其余节点移动。从后向前遍历完成新建/移动。复杂度 O(n log n)

**为什么 LIS 能让移动最少**：LIS 找出的是「在新旧两个序列中相对顺序都保持不变的最长节点序列」，这些节点不动，剩下的节点一次性移动，总移动次数最少。

### Q10：key 的作用是什么？用 index 做 key 有什么问题？

**答**：key 是 vnode 的身份标识，diff 时用 `isSameVNodeType(oldVnode, newVnode)`（key 和 type 都相同）判断「是不是同一个节点」，从而决定复用还是销毁重建。用 index 做 key 的问题：当列表发生**插入/删除/排序**时，同位置的 index 相同但内容不同，diff 会错误复用——例如删除第一项后，原来第二项的 key(1) 变成第一项的 key(0)，React/Vue 会认为「key 0 还在」，只是内容变了，于是就地更新而不是移动，导致：内部状态（input 输入值、动画、组件实例状态）串位；且可能引发大量不必要的 patch。只有在**列表只追加且无状态组件**的场景下 index key 才勉强可用。

### Q11：v-if 和 v-show 的区别与原理？

**答**：`v-if` 是**真正的条件渲染**，编译成三元表达式（Vue 3 编译为 block 边界），切换时销毁/重建组件和 DOM，有更高的切换开销；`v-show` 编译成 `vShow` 指令，初始渲染无论如何都渲染 DOM，通过 `display: none` 控制显隐，切换只是改样式，有更高的初始渲染开销。所以频繁切换用 v-show，条件很少变用 v-if。

### Q12：v-for 中 key 变化时 DOM 是复用的还是重建的？v-if 和 v-for 谁优先级高？

**答**：key 相同则复用（patch），不同则销毁重建。Vue 2 中 v-for 优先级高于 v-if（同一节点上 v-for 先执行，可能对每项都执行 v-if 判断，不推荐同时用）；Vue 3 中 v-if 优先级更高（v-if 先执行，此时可能访问不到 v-for 的变量会报错）。**正确写法**是在外层 template 上用 v-if，或者用 computed 过滤列表。

## C. computed / watch 类

### Q13：computed 的原理和缓存机制？

**答**：

- **Vue 2**：每个 computed 创建一个 `lazy: true` 的 Watcher，创建时不求值；模板读取时通过 `computedGetter`，先判断 `watcher.dirty`，为 true 才执行 getter 求值并更新缓存值、置 dirty = false；依赖数据变化时 computed watcher 的 update 只是置 `dirty = true`（不立即求值），同时通过 `watcher.depend()` 把 computed 的依赖桥接给渲染 watcher，保证渲染能感知
- **Vue 3**：computed 是一个特殊的 effect：`lazy: true` + `dirty` 标记 + `scheduler`。读取时若 dirty 则计算；Vue 3.4 优化后，computed 依赖的 dep 会做「版本/脏标记」传递，未变化的依赖不会触发上层 effect 重跑，进一步减少无效计算

**缓存的意义**：同一个依赖没变时多次读取 computed 不重复计算；且 computed 不会被多次求值（依赖收集时同一轮只收一次）。

### Q14：computed 和 watch 的区别？

**答**：computed 是**派生值**：由其他响应式数据计算而来，有缓存，依赖不变不重算，必须返回值，不支持异步；watch 是**副作用**：观察数据变化执行回调，无缓存（每次变化都触发），支持异步、`deep`、`immediate`、`flush` 时机（pre/post/sync），适合「数据变化后做某件事」（如请求接口、操作 DOM）。一个用于「得到值」，一个用于「响应变化做动作」。

### Q15：watch 和 watchEffect 的区别？

**答**：`watch` 需要显式指定监听源（ref/reactive/getter/数组），回调能拿到新旧值，默认懒执行（immediate 可选）；`watchEffect` 立即执行一次回调，**自动收集**回调中用到的所有响应式依赖（不需要声明依赖列表），拿不到旧值，适合「依赖多个数据、只关心结果」的场景。两者都返回 stop 函数可手动停止。

## D. nextTick / 更新机制类

### Q16：nextTick 的原理？为什么改了数据 DOM 没立刻变？

**答**：Vue 的 DOM 更新是**异步批量**的：数据变化触发 setter → 通知 watcher → watcher 进入更新队列（去重）→ 通过 `nextTick(flushSchedulerQueue)` 注册一个微任务（Promise.then）→ 当前同步代码执行完后统一 flush，此时才真正执行 render + patch。所以同步代码里改完数据立刻读 DOM 是旧值，需要 `await nextTick()` 后再读。nextTick 的实现：维护一个回调队列，用 Promise.then（降级 MutationObserver/setTimeout）异步 flush；它的价值还在于「在 DOM 更新后执行自定义回调」，和渲染更新在同一个微任务批次中按入队顺序执行。

### Q17：Vue 的更新是同步还是异步？为什么设计成异步？

**答**：异步。原因：**批量去重**——同一个 tick 里修改 N 个数据，组件只重渲染一次，避免重复 render/patch；**保证顺序可预期**——同步渲染会在数据未改完时就触发视图更新，产生中间态。代价是需要 nextTick 来对齐「DOM 已更新」的时机。

### Q18：父子组件的更新顺序？为什么？

**答**：**更新**时父组件的渲染 watcher/job 先于子组件（按创建 id 排序，id 小的先创建），因为父组件更新可能改变子组件的结构（v-if/props），先更新父再更新子可以避免子组件无效更新；**销毁**时父组件先触发 beforeDestroy/beforeUnmount，子组件后销毁（递归 $destroy / unmount）；**挂载**时 created 顺序是父→子，mounted 是子→父（子组件挂载完成先于父组件 mounted 回调执行）。

## E. 组件通信与设计类

### Q19：Vue 组件通信的方式有哪些？

**答**：

- **props / $emit**（Vue 3：`defineProps` / `defineEmits`）：父子
- **v-model / defineModel**（Vue 3.4）：双向绑定语法糖
- **$refs / expose**（Vue 3：`defineExpose`）：父访问子
- **provide / inject**：跨层级（祖先→后代），Vue 3 中 provide 的值如果是响应式的，inject 侧也能保持响应式
- **EventBus / mitt**（Vue 3 移除了 $on/$off，需要外部库）：任意组件
- **Vuex / Pinia**：全局状态
- **$attrs / $listeners**（Vue 3 中 $listeners 合并进 $attrs）：透传
- **slot/scope slot**：内容分发

### Q20：Vue 2 的 mixin 有什么问题？Composition API 怎么解决的？

**答**：mixin 的问题：

1. **数据来源不透明**：组件里用到一个方法/属性，不知道来自哪个 mixin，需要翻所有 mixin
2. **命名冲突**：多个 mixin 或 mixin 与组件同名属性会覆盖，合并策略复杂
3. **隐式依赖**：mixin 可能依赖组件的 data/methods，耦合脆弱
4. **逻辑分散**：一个功能（如拖拽）的 data+mounted+destroyed 分散在不同选项

Composition API 用**函数组合**解决：composable 是普通函数，内部状态是函数作用域的 ref，依赖通过参数显式传入，返回值显式导出；同一功能代码聚合在一处；TypeScript 类型友好；可重命名避免冲突（`const { x: positionX } = useMouse()`）。

### Q21：Vue 3 为什么移除了 filters 和 $on/$off？

**答**：filters 功能（文本格式化）可以用 methods/computed 完全替代，且对 TypeScript 类型和 Tree-shaking 不友好，双竖线语法还与 JS 位运算冲突，增加编译器复杂度；$on/$off 是 EventBus 的实现，这种「隐式全局事件」模式容易导致事件监听泄漏和维护困难，社区用 mitt/tiny-emitter 这类独立库替代，框架保持核心精简（Tree-shaking 也要求 API 尽量少而精）。

### Q22：v-model 的原理？Vue 2 和 Vue 3 有什么区别？

**答**：v-model 是语法糖。

- **Vue 2**：在 input 上是 `:value + @input` 的糖；在组件上是 `:value + @input` 且默认用 `value` prop 和 `input` 事件（可用 model 选项改名）；`.sync` 修饰符是 `:prop + @update:prop` 的糖，用于双向绑定非表单场景
- **Vue 3**：组件的 v-model 统一为 `modelValue` prop + `update:modelValue` 事件；支持**多个 v-model**（`v-model:title="x"` 即 `:title + @update:title`）；移除了 `.sync`；内置修饰符 `.trim/.number/.lazy` 支持，自定义修饰符通过 `modelModifiers` prop 传给组件

### Q23：为什么 Vue 3 的根组件可以多根节点（Fragment）？

**答**：Vue 2 的 vnode diff 和组件实例绑定 DOM 的机制都假设单根（组件的 `$el` 是单一 DOM 节点，属性透传、transition 都依赖它），多根无法确定 `$el` 指向谁。Vue 3 引入 Fragment vnode 类型（`shapeFlag.FRAGMENT`），patch 时对 Fragment 的 children 逐个处理，支持多根渲染；代价是**多根组件不能自动继承 attrs**，需要手动 `v-bind="$attrs"`，否则告警。

### Q24：KeepAlive 的实现原理？

**答**：KeepAlive 是一个内置组件，本质是**缓存 vnode 而不销毁组件实例**。它的 render 里返回缓存的子 vnode；当子组件被切换掉时，patch 过程检测到 `COMPONENT_SHOULD_KEEP_ALIVE` 标记，不执行真正的 unmount，而是把 vnode 存入 `Map<key, vnode>` 缓存（key 由组件名或显式 key 生成），并把真实 DOM 移到隐藏容器中；再次激活时从缓存取 vnode，DOM 移回原位，跳过 mount 的重新实例化。配合 `include/exclude/max`（LRU 淘汰策略）控制缓存范围；activated/deactivated 钩子在激活/失活时触发。

## F. 综合与工程类

### Q25：Vue 3 性能比 Vue 2 好在哪？从哪几个层面说？

**答**：三个层面：

1. **编译时**：静态提升（静态节点零开销）、PatchFlag（属性 diff 精准化）、Block Tree（只 diff 动态部分）、cacheHandler（事件缓存）
2. **响应式运行时**：Proxy 惰性代理（初始化不递归）、大数组优化（整数索引不 track）、统一调度
3. **Diff**：预处理 + LIS，O(n log n)，移动次数最优
4. **体积**：Tree-shaking，核心约 10KB（Vue 2 全量约 30KB+）

### Q26：如果让你实现一个简易的 Vue 3 响应式，核心要写哪些东西？

**答**：四个部分：

1. `reactive(target)`：`new Proxy(target, handlers)`，get 里 track、set 里 trigger，缓存 proxyMap
2. `track(target, key)`：维护 `WeakMap<target, Map<key, Set<effect>>>`，把 `activeEffect` 加入对应 Set
3. `effect(fn)`：执行 fn 前 `activeEffect = this`，fn 中访问数据自动收集；提供 `scheduler` 支持调度
4. `trigger(target, key)`：从 targetMap 取出依赖 Set，遍历调用每个 effect 的 scheduler/run

加分项：依赖清理（v-if 分支切换）、computed 的 lazy/dirty、数组和集合类型的特殊处理、effect 嵌套栈。

### Q27：Vue 2 中 data 为什么必须是个函数（组件）？

**答**：组件是可复用的实例，如果 data 是对象，所有实例会**共享同一个对象的引用**（对象是引用类型），一个实例改数据会影响其他实例。写成函数则每次创建组件时调用 `data()` 返回一个**全新对象**，实例间数据隔离。根实例 `new Vue()` 不复用，所以对象也可以（但为了统一，函数是推荐写法）。

### Q28：Vue 3 的 effect 和 Vue 2 的 Watcher 对应关系？

**答**：概念上是同一个东西的演进：

| Vue 2 | Vue 3 |
|---|---|
| 渲染 Watcher | 渲染 effect（`componentUpdateFn` + scheduler 推入 queueJob） |
| 用户 Watcher | `watch`（基于 effect，getter 包裹 + scheduler 入队回调） |
| computed Watcher | computed effect（lazy + dirty + 缓存） |
| Dep.target 栈 | activeEffect + effectStack 栈 |
| Dep.subs（数组） | targetMap 里的 Set<effect> |

本质区别：Vue 2 的 Watcher 和 Dep **互相持有**（双向关联，需要双向清理）；Vue 3 的 effect 和 dep **单向持有**（effect.deps 记录自己依赖的 dep，cleanup 时反向移除），实现更简单。

---

## 附：面试答题技巧提示

- 问「原理」时按「**触发 → 收集 → 更新**」三段式回答：数据怎么触发拦截 → 依赖怎么收集 → 更新怎么调度执行
- 问「区别」时先说本质（实现机制），再说表象（API/行为），最后说影响（性能/场景）
- Vue 2 vs Vue 3 的对比题，优先从「**defineProperty vs Proxy**」「**双端 diff vs 快速 diff**」「**Options vs Composition**」三个最热的点切入
- 回答里带上一个具体代码例子（如 $set 的实现、LIS 的过程演示）会显著提升说服力
