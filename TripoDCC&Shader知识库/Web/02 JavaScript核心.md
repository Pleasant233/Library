# 02 JavaScript 核心

> 配套 [[面试题库]] 的「JavaScript」部分。这几个题都是「懂原理才能答好」的经典题。

## 一、闭包与内存

**闭包**：函数记住并能访问它「出生时」所在作用域的变量，即使外层函数已执行完。

**为什么会内存泄漏**：闭包引用的变量不会被垃圾回收（GC），只要闭包还活着。

```js
function createHandlers() {
  const bigData = new Array(10000).fill('*'.repeat(10000)) // 约 100MB
  return {
    onClick() { console.log('clicked') },
    getData() { return bigData },
  }
}
const { onClick } = createHandlers()
// 之后只用 onClick，从不调用 getData
```

- **入门**：`bigData` 会被回收吗？→ **不一定**。因为返回的对象里 `getData` 闭包引用了 `bigData`，只要这个对象活着，`bigData` 就活着。而 `onClick` 被解构出来长期持有，可能间接让整个对象/闭包环境存活。
- **进阶**：如果删掉 `getData`，只返回 `onClick`？→ 结果**可能不同**。V8 会做优化：一个函数如果**没有任何闭包引用** `bigData`，编译器分析后可以不把它放进闭包环境，`bigData` 就能被回收。但只要同作用域有任一函数用到它，V8 通常会保留整个环境。
- **深入（SPA 实战）**：常见泄漏源——**全局事件监听器 / 定时器没解绑**。
  - 组件销毁时忘了 `removeEventListener`、`clearInterval`，回调闭包一直持有组件数据。
  - **排查**：Chrome DevTools → Memory → 拍两次堆快照（Heap Snapshot）对比，看哪些对象只增不减（Detached DOM、闭包）。
  - **对策**：Vue `onUnmounted` / React `useEffect` 的 cleanup 函数里解绑。

---

## 二、Proxy 与响应式

**背景**：Vue 3 用 `Proxy` 替代 Vue 2 的 `Object.defineProperty`。

**Vue 2 的痛点（入门必答）**：
- **新增/删除属性检测不到**：`defineProperty` 只能给「已存在的属性」加 getter/setter，新增属性得用 `Vue.set`。
- **数组变异检测不到**：`arr[0] = x`、`arr.length = 0` 无法触发更新，Vue 2 只能靠**重写数组方法**（push/pop/splice…）来 hack。

**Proxy 能拦截而 defineProperty 不能的操作**：属性新增、删除（`deleteProperty`）、`in` 判断（`has`）、数组索引赋值、`length` 变化——因为 Proxy 代理的是**整个对象**，不是逐个属性。

**手写最简 `reactive()` 思路**（说清 trap + 数据结构即可）：
```js
const targetMap = new WeakMap() // target -> Map(key -> Set(effect))
let activeEffect = null

function reactive(obj) {
  return new Proxy(obj, {
    get(target, key, receiver) {
      track(target, key)                      // 读取时：收集依赖
      return Reflect.get(target, key, receiver)
    },
    set(target, key, value, receiver) {
      const r = Reflect.set(target, key, value, receiver)
      trigger(target, key)                    // 修改时：触发更新
      return r
    },
  })
}
```
- 关键 trap：`get` 收集依赖、`set` 触发通知。
- 为什么用 **WeakMap** 存依赖？→ key 是被代理对象，弱引用，对象销毁时依赖自动被 GC，不会内存泄漏（呼应 [[#四、弱引用 WeakRef]]）。
- **进阶**：嵌套对象**懒代理**——只在 `get` 到对象类型时才递归 `reactive`，避免一次性深度代理整棵树。用 WeakMap 缓存已代理对象，**避免重复代理**。

---

## 三、事件循环的微妙之处

```js
Promise.resolve()
  .then(() => { console.log(1); return Promise.resolve(2) })
  .then((v) => console.log(v))

Promise.resolve()
  .then(() => console.log(3))
  .then(() => console.log(4))
  .then(() => console.log(5))
```

**输出：`1, 3, 4, 5, 2`**（不是 `1,3,2,4,5`）。

- **入门**：两条链都是微任务，交替执行。第一条打印 1 后 `return Promise.resolve(2)`，这里 **2 被延后了**。
- **进阶：为什么 `return Promise.resolve(2)` 多花两个 tick？**
  - 当 `.then` 回调返回的是一个 thenable（Promise），规范要求走 `PromiseResolveThenableJob`，需要额外排一个微任务去「拆包」这个 Promise，再排一个微任务把值 `2` 传给下一个 `.then`。所以 2 落后了两个 tick，正好让 3、4、5 先跑完。
- **深入：`queueMicrotask` vs `Promise.then` vs `MutationObserver`？**
  - 三者都进**微任务队列**。`queueMicrotask` 是「我只想排个微任务，不需要 Promise 包装」时的直接手段，语义更清晰、开销更小。
  - `MutationObserver` 也是微任务，但用于监听 DOM 变化。
  - 场景：批量收集操作后在「本轮同步代码结束、渲染前」统一处理，用 `queueMicrotask`。

> 宏任务（macrotask）：`setTimeout`、`setInterval`、I/O。微任务（microtask）：`Promise.then`、`queueMicrotask`、`MutationObserver`。
> **执行规则**：每执行完一个宏任务，就清空**所有**微任务，然后才可能渲染。详见 [[04 浏览器原理]]。

---

## 四、弱引用 WeakRef

**「弱」的含义**：不阻止 GC 回收对象。

**`WeakMap` vs `WeakRef` 的区别（入门）**：
- **`WeakMap`**：它的 **key** 是弱引用。key 对象没别人引用时，这个键值对整体被回收。
- **`WeakRef`**：**它本身**弱引用一个对象。通过 `ref.deref()` 拿对象，可能拿到 `undefined`（已被回收）。

**WeakRef 有用的场景（进阶）**：缓存。
```js
const cache = new Map()
function getThumb(id) {
  const ref = cache.get(id)
  const cached = ref?.deref()
  if (cached) return cached          // 还在，直接用
  const img = loadThumb(id)          // 被回收了，重新加载
  cache.set(id, new WeakRef(img))
  return img
}
```
内存紧张时，缓存的大对象可以被 GC 回收，避免缓存把内存撑爆。

**`FinalizationRegistry` 时机为什么不确定（深入）**：它的清理回调由 GC 触发，而 **GC 何时运行是引擎决定的**，不保证及时、甚至不保证一定执行。
- **影响**：绝不能把它用于关键业务逻辑（如「对象销毁必须关连接」）。它只适合「回收后顺手清理一下」这种**尽力而为**的辅助逻辑。

---

## 五、模块系统与 Tree Shaking

**Tree Shaking**：打包时**摇掉没被用到的代码**，减小体积。

```js
// utils/index.js 导出 4 个函数
// 使用方只 import { formatDate }
```
- **入门**：其他三个会被打进去吗？→ **取决于是否 ESM + 是否有副作用**。ESM 的 `import/export` 是**静态**的，打包器能在编译期分析出「谁没被用」。CommonJS（`require`）是动态的，分析不了，**无法 tree shake**。
- **进阶：什么情况失效？**
  1. 用了 **CommonJS** 模块。
  2. 代码有**副作用**：顶层就执行的语句、`Object.assign` 给原型打 polyfill、顶层 IIFE——打包器不敢删，怕删了影响运行。
- **深入**：
  - **`package.json` 的 `sideEffects`**：告诉打包器「我这个包哪些文件没有副作用，可以放心摇」。设 `"sideEffects": false` = 全部可摇。**设错**（比如漏标了带副作用的 CSS 引入 `import './style.css'`）→ 样式被摇掉、页面丢样式。
  - **`/*#__PURE__*/` 注释**：标记某次函数调用「无副作用」，帮打包器判断可删。
  - **barrel file 陷阱**：`index.ts` re-export 一堆模块，可能导致引入一个就把整个 barrel 拉进来，影响构建速度和摇树效果。

---

## 面试速答卡

- **闭包泄漏怎么查**？→ Heap Snapshot 对比 + cleanup 解绑。
- **Vue3 为什么用 Proxy**？→ 解决新增属性 / 数组变异检测；拦截整个对象。
- **那道 Promise 输出**？→ `1,3,4,5,2`，thenable 拆包多两个微任务 tick。
- **Tree Shaking 前提**？→ ESM 静态分析 + 无副作用；`sideEffects` 配错会删样式。
