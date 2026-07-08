# 03 TypeScript

> 配套 [[面试题库]] 的「TypeScript」部分。重点是「类型安全思维」，不是背语法。

## 一、any vs unknown vs never

| 类型 | 含义 | 一句话 |
| --- | --- | --- |
| `any` | 关闭类型检查 | 什么都能做，**放弃**了类型保护 |
| `unknown` | 类型安全的「任意值」 | 什么都能存，但**用之前必须收窄** |
| `never` | 永不存在的值 | 「不可能到达」的类型 |

**`any` vs `unknown` 关键区别**：
```ts
let a: any = getData()
a.foo.bar()        // ✅ 编译通过，运行时可能崩

let u: unknown = getData()
u.foo              // ❌ 报错：必须先收窄
if (typeof u === 'string') u.toUpperCase()  // ✅ 收窄后才能用
```
> 用 `unknown` 逼你写类型守卫，安全；`any` 是「我不管了」。

**`never` 自然出现的两个场景（不是你手写的）**：
1. **函数永不返回**：抛异常或死循环。
   ```ts
   function throwError(msg: string): never { throw new Error(msg) }
   ```
2. **穷尽所有分支后**：联合类型被收窄到空。
   ```ts
   function process(value: string | number) {
     if (typeof value === 'string') return value.toUpperCase()
     if (typeof value === 'number') return value.toFixed(2)
     return throwError('unexpected')   // 这里 value 的类型是 never
   }
   // 上面 process 的返回类型 = string
   ```
   **进阶用法**：`const _exhaustive: never = value` 可做「穷尽性检查」，将来 union 加了新成员而忘记处理，这里会编译报错。

- **深入**：`never` 在条件类型里有「过滤」作用，是 `Exclude` 的原理——分发到某个成员时返回 `never`，就等于从联合里剔除它。

---

## 二、类型推断 vs 显式注解

```ts
// 风格 A：到处注解
const name: string = 'hello'
// 风格 B：让 TS 推断
const name = 'hello'
```

**核心区别**：
- 风格 A：`name` 类型是 `string`（宽类型）。
- 风格 B：`const` + 字面量 → 推断为**字面量类型** `'hello'`（窄类型）。

这个区别**很重要**：
```ts
const dir = 'left'                 // 类型是 'left'，能传给 '左右' union
const dir2: string = 'left'        // 类型被拓宽成 string，传不进 union 了
type Dir = 'left' | 'right'
function move(d: Dir) {}
move(dir)   // ✅
move(dir2)  // ❌ string 不能赋给 Dir
```
> 所以「到处写 `: string`」反而可能**把精确的字面量类型退化成宽类型**，弄巧成拙。

**什么时候必须显式注解**：
- **函数参数**：TS 推断不出，必须写。
- **公共 API / 模块边界的函数返回值**：显式写，锁定契约、加速编译、防止实现改动悄悄改变返回类型。
- 内部临时变量：交给推断，更简洁。

**结论**：能推断的让 TS 推断，**边界处**（参数、导出 API）显式注解。

---

## 三、泛型的边界

```ts
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
function setProperty<T, K extends keyof T>(obj: T, key: K, value: T[K]): void {
  obj[key] = value
}
const user = { name: 'Alice', age: 25 } as const
setProperty(user, 'name', 'Bob')   // 会怎样？
```
- **入门**：`as const` 让所有属性变成 **`readonly`** 且是字面量类型。`name` 类型是 `'Alice'`（只读）。给它赋 `'Bob'` → **报错**（既是只读、`'Bob'` 也不是 `'Alice'`）。
- **进阶**：去掉 `as const` 有安全漏洞吗？→ 当 `T` 被推断成**联合类型**时，`keyof` / 索引访问会出现协变/逆变问题，`setProperty` 可能允许写入本不该允许的值。
- **深入：泛型什么时候是过度设计？**
  - 这种「万能 getter/setter」封装，收益是类型安全，成本是可读性下降、报错信息难懂。
  - **YAGNI 原则**：只有一两处用、类型固定时，直接写具体类型更清晰。泛型是给「真正需要复用 + 类型随调用方变化」的场景用的。

---

## 四、声明文件与类型扩展

**场景**：引入没类型的第三方库，或给 `window` / Vue 实例挂自定义属性。

- **`.d.ts` 声明文件**：只写类型、不含实现。给无类型库补声明。
- **`declare global`**：扩展全局类型，比如给 `Window` 加属性。
  ```ts
  // global.d.ts
  declare global {
    interface Window { myApp: { version: string } }
  }
  export {}   // 让文件成为模块，declare global 才生效
  ```
- **`declare module`（module augmentation）**：给已有模块「补充/扩展」类型，比如给 Vue 的 `ComponentCustomProperties` 加全局属性 `$http`。

**`declare module '*.vue'` 的作用**：告诉 TS「`.vue` 文件是一个 Vue 组件模块」。删掉它，`import Foo from './Foo.vue'` 会报「找不到模块」——因为 TS 原生不认识 `.vue` 扩展名。

- **深入**：两库类型冲突 → 靠 `paths` 映射、`typeRoots` 控制查找范围；实在编译不过用 `skipLibCheck` 跳过库类型检查（**权衡**：省心但可能漏掉真实类型错误）。

---

## 五、类型设计好不好（判别联合）

```ts
interface BaseAction { type: string; payload?: any }  // 反面教材
function dispatch(action: BaseAction) {
  switch (action.type) {
    case 'ADD_TODO': console.log(action.payload.text); break  // payload 是啥？没人知道
    case 'REMOVE_TODO': console.log(action.payload.id); break
  }
}
```
- **问题**：`type: string` 太宽、`payload: any` 完全没保护。写错字段名、传错类型都不报错。
- **重构：判别联合类型（Discriminated Union）**——每种 action 有确定的 `type` 字面量和对应 `payload`：
  ```ts
  type Action =
    | { type: 'ADD_TODO'; payload: { text: string } }
    | { type: 'REMOVE_TODO'; payload: { id: number } }

  function dispatch(action: Action) {
    switch (action.type) {
      case 'ADD_TODO': action.payload.text     // ✅ 自动收窄，text 有类型
      case 'REMOVE_TODO': action.payload.id    // ✅ id 有类型
    }
  }
  ```
  TS 靠公共的「判别字段」`type` 自动收窄 payload 类型。
- **深入（几十种 action 怎么组织）**：用**映射类型 + 泛型**从一张 `type -> payload` 的映射表自动生成 Action 联合和 action creator，或用 `satisfies` 校验一张类型安全的 action map，避免手写几十个接口。

---

## 面试速答卡

- **any vs unknown**？→ unknown 用前必须收窄，更安全。
- **never 两个场景**？→ 永不返回的函数、穷尽分支后的空联合。
- **`const x = 'a'` 推断成啥**？→ 字面量 `'a'`，不是 `string`。
- **payload: any 怎么救**？→ 判别联合，靠 `type` 字面量收窄。
