# 01 HTML 与 CSS

> 配套 [[面试题库]] 的「Html + CSS」部分。入门到实用。
> **前置**：没写过 CSS、不清楚盒模型/选择器/定位，先看 [[01.0 CSS上手]] 再回来。本篇偏实用与面试层。

## 一、语义化 HTML 与可访问性

**是什么**：用有含义的标签（`<header> <nav> <main> <section> <article> <aside> <footer>`）代替满屏 `<div>`。

**为什么好**：
- **可访问性（a11y）**：屏幕阅读器能识别结构，视障用户可以「跳到主内容」「按标题导航」。`<div>` 对它们只是一团无意义的盒子。
- **SEO**：搜索引擎爬虫靠标签语义理解页面重点，`<article>`、`<h1>` 权重更高。
- **可维护性**：`<nav>` 一眼看出是导航，比 `<div class="nav">` 更自解释。

**典型页面结构**：
```html
<body>
  <header>网站 logo、顶部导航</header>
  <nav>主导航菜单</nav>
  <main>
    <article>
      <h1>文章标题</h1>
      <section>正文段落</section>
    </article>
    <aside>侧边栏 / 相关推荐</aside>
  </main>
  <footer>版权、备案信息</footer>
</body>
```

**一句话记忆**：语义标签 = 给机器（爬虫、读屏软件）看的结构说明书。

---

## 二、Flexbox 与 Grid 的取舍

| | Flexbox | Grid |
| --- | --- | --- |
| 维度 | **一维**（一行或一列） | **二维**（行 + 列） |
| 适合 | 局部排列：导航条、按钮组、居中 | 整体布局：页面骨架、卡片网格 |
| 核心属性 | `flex`、`justify-content`、`align-items` | `grid-template-columns/rows`、`gap` |

**实战组合**：外层用 Grid 搭页面骨架，内部每块用 Flex 排内容。
```css
/* 页面骨架：左侧栏 240px + 主区自适应 */
.layout { display: grid; grid-template-columns: 240px 1fr; gap: 16px; }
/* 局部：一行按钮靠右排列 */
.toolbar { display: flex; justify-content: flex-end; gap: 8px; }
```

### 进阶：flex 简写三兄弟
`flex: <grow> <shrink> <basis>`，常见 `flex: 1` = `flex: 1 1 0%`。

- **`flex-grow: 1`**：有剩余空间时，按比例「瓜分」放大。
- **`flex-shrink: 1`**：空间不够时，按比例「压缩」。设为 `0` 则拒绝被压缩。
- **`flex-basis: 0%`**：分配前的初始基准尺寸。

**踩坑：子元素被压没了怎么办？**
- 给「不想被压缩」的元素设 `flex-shrink: 0`（比如 logo、图标）。
- 或给它设 `min-width`，压缩不会突破 `min-width`。
- 文本溢出用 `min-width: 0` 配合 `overflow: hidden`（flex 子项默认 `min-width: auto` 会撑破容器，这是高频坑）。

---

## 三、响应式设计与移动优先

**移动优先（mobile-first）**：先写小屏样式作为默认，再用 `min-width` 媒体查询逐步「加宽加料」。好处是移动端不用覆盖多余样式，性能更好。

四件套：
- **流式布局**：宽度用 `%` / `fr` / `flex`，不写死 `px`。
- **弹性图片**：`img { max-width: 100%; height: auto; }`，图片不超出容器。
- **媒体查询**：按屏幕宽度切换样式。
- **断点**：常见 768px（平板）、1024px（桌面）。

```css
/* 默认：移动端单列 */
.grid { display: grid; grid-template-columns: 1fr; }
/* 平板及以上：两列 */
@media (min-width: 768px) {
  .grid { grid-template-columns: 1fr 1fr; }
}
```

**别忘了**：HTML 里加 `<meta name="viewport" content="width=device-width, initial-scale=1">`，否则手机会按桌面宽度缩放渲染。

---

## 四、现代 CSS：变量与函数

**CSS 自定义属性（CSS 变量）**：
```css
:root { --primary: #4f46e5; --gap: 16px; }
.btn { color: var(--primary); padding: var(--gap); }
```

**与 SASS 变量的关键区别**：
| | CSS 变量 | SASS 变量 |
| --- | --- | --- |
| 生效时机 | **运行时**，浏览器里可改 | **编译时**，编译后就没了 |
| 能否 JS 修改 | 能（`el.style.setProperty`） | 不能 |
| 作用域 | 跟随 DOM 层级，可继承/覆盖 | 词法作用域 |

正因为「运行时可变」，CSS 变量特别适合**主题切换**：
```css
:root { --bg: #fff; --text: #111; }
[data-theme="dark"] { --bg: #111; --text: #eee; }
body { background: var(--bg); color: var(--text); }
/* JS 里 document.documentElement.dataset.theme = 'dark' 即可切换 */
```

**实用函数 `clamp(min, preferred, max)`**：一行实现「最小不小于、最大不超过、中间跟随视口」的响应式字号：
```css
h1 { font-size: clamp(1.5rem, 4vw, 3rem); }
```

---

## 五、大型项目的样式架构

**要解决的问题**：全局污染、命名冲突、选择器过深（影响性能）、巨型 CSS 文件。

常见方案对比：
| 方案 | 思路 | 优点 |
| --- | --- | --- |
| **BEM** | 命名约定 `block__element--modifier` | 零工具、可读性强 |
| **CSS Modules** | 编译期把类名哈希成唯一值 | 天然隔离、无运行时开销 |
| **Tailwind / UnoCSS** | 原子化类名，`flex p-4 text-sm` | 无需起名、按需生成、体积小 |
| **CSS-in-JS** | 在 JS 里写样式（styled-components） | 动态样式、与组件强绑定 |

**性能要点**：
- 选择器别太深（`.a .b .c .d` 匹配慢），优先单类名。
- 按需加载 / 组件级样式，避免一个大 CSS 文件全量下发。
- 避免频繁触发**回流**（见 [[04 浏览器原理]] 的渲染流水线）。

---

## 面试速答卡

- **语义化好处**？→ a11y + SEO + 可维护。
- **Flex vs Grid**？→ 一维 vs 二维；Grid 搭骨架、Flex 排局部。
- **CSS 变量 vs SASS 变量**？→ 运行时可变 vs 编译时；主题切换靠前者。
- **样式怎么组织**？→ 结合项目说一种（如 Tailwind + CSS Modules），强调隔离和按需。
