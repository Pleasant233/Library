# 06 Vue 与 React

> 配套 [[面试题库]] 的「框架类」部分。这里聚焦响应式和逻辑复用的对比。

## 一、Vue 响应式基础：依赖收集

**依赖收集**：响应式系统在「读取」时记下「谁用了我」，在「修改」时通知这些人更新。（底层 Proxy 见 [[02 JavaScript核心]]）

**`watch` / `computed` 怎么收集依赖**：
- Vue 内部有个全局的「当前正在运行的 effect」。
- 当 `computed` 的 getter 或 `watch` 的回调**执行时**，它读取的每个响应式属性会通过 `get` trap 把「当前 effect」记进该属性的依赖集合。
- 属性变化时，`set` trap 遍历依赖集合，通知这些 effect 重新执行。

```js
const count = ref(0)
const double = computed(() => count.value * 2)
// 执行 getter 时读了 count.value → count 记住了 double 这个依赖
count.value++   // 触发 count 的依赖 → double 重新计算
```

**为什么能「自动回收」，什么样的 watch 不会自动回收**：
- **组件内**创建的 `watch` / `computed` / `watchEffect`，绑定到当前组件实例的**作用域（effectScope）**上。组件卸载时，Vue 自动 stop 这些 effect，解除依赖 → **自动回收**。
- **不会自动回收的情况**：在组件作用域**之外**创建的（比如在一个模块顶层、或在异步回调里 `setTimeout` 之后才创建的 watch，此时已脱离 setup 同步执行上下文）。这些不归属任何组件，需要**手动**保存返回的 `stop` 函数并调用它，否则泄漏。
  ```js
  const stop = watch(source, cb)
  // 不再需要时手动停止
  stop()
  ```

---

## 二、Composable（Vue）vs Hook（React）

**相同之处**：
- 都是**逻辑复用**手段——把「状态 + 逻辑」抽成可复用函数（`useXxx`）。
- 都替代了旧的复用方式（Vue 的 mixin、React 的 HOC/render props）。
- 命名约定都是 `use` 前缀。

**不同之处**（关键）：
| | Vue Composable | React Hook |
| --- | --- | --- |
| 执行次数 | `setup` 里**只执行一次** | **每次渲染都重新执行** |
| 响应式来源 | `ref`/`reactive`，靠依赖收集 | `useState`，靠重新执行 + 闭包 |
| 调用限制 | 可以写在条件/循环里（相对宽松） | **不能**放条件/循环（依赖调用顺序） |
| 依赖数组 | 不需要（自动追踪） | `useEffect`/`useMemo` 要手写依赖数组 |

**一句话**：Vue 靠「响应式对象 + 只运行一次」，React 靠「每次重跑函数 + Hooks 顺序」。

**思考题：composable 里创建了 `computed`，又被用在节点的点击事件里会怎样？**
- 如果这个 composable 在组件 `setup` 中调用，`computed` 归属组件作用域，随组件卸载被回收——**正常**。
- 但如果它在**点击事件回调里**才被调用（即脱离了 setup 的同步执行上下文），那么创建的 `computed` **不再自动绑定到组件作用域**，组件卸载时不会自动 stop → 每点一次就多一个不被回收的 effect → **内存泄漏 / 重复副作用**。
- **正确做法**：响应式声明放在 setup 顶层同步执行；确实要动态创建，用 `effectScope` 手动管理并在合适时机 `stop`。

---

## 三、React 补充（面试常问）

题库里 react 一节留白，补几个高频点：
- **为什么需要 key**：帮助 diff 算法识别列表项身份，避免错误复用 DOM。别用 index 当 key（顺序变会出 bug）。
- **useEffect 依赖数组**：空数组=挂载时跑一次；有依赖=依赖变才跑；不写=每次渲染都跑。清理函数处理解绑（呼应 [[02 JavaScript核心]] 内存泄漏）。
- **useMemo / useCallback**：缓存计算结果 / 函数引用，减少不必要的重渲染。别滥用，本身也有成本。
- **受控 vs 非受控组件**：表单值由 state 管（受控）还是由 DOM 管（非受控，用 ref）。

---

## 面试速答卡

- **依赖收集原理**？→ 读取时记录当前 effect，修改时通知。
- **哪种 watch 不自动回收**？→ 组件作用域外创建的，需手动 stop。
- **Composable vs Hook 最大区别**？→ Vue 只运行一次靠响应式；React 每次渲染重跑靠 Hooks 顺序。
- **React key 为什么重要**？→ diff 识别身份，别用 index。
