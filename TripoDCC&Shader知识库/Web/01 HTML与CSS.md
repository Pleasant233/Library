# 01 HTML 与 CSS

> 配套 [[面试题库]] 的「Html + CSS」部分。
> **前置**：先看 [[00.3 CSS上手]] 认全"标签/元素/div/class/样式"这些符号和 CSS 四大机制，再回来。
> 本篇前半段（第 0 节）把 **HTML 的完整全貌**补齐（文件骨架、常见标签、嵌套父子），面向零基础；后半段是语义化 / Flex/Grid / 响应式 / 样式架构的实用与面试层。

---

## 零、HTML 全貌：一个真实网页文件长什么样

[[00.3 CSS上手]] 讲了单个元素怎么读。这里给你**一整个 HTML 文件**，看它是怎么组织的。

### 0.1 文件骨架（每个网页都长这样）

```html
<!DOCTYPE html>                 <!-- ① 文档类型声明：告诉浏览器"用 HTML5 标准解析我" -->
<html lang="zh">                 <!-- ② 根元素：整个页面的最外层盒子，lang 声明语言 -->
  <head>                        <!-- ③ 头部：给浏览器/搜索引擎看的元信息，不直接显示在页面 -->
    <meta charset="UTF-8">      <!--    字符编码，固定写 UTF-8，否则中文可能乱码 -->
    <meta name="viewport" content="width=device-width, initial-scale=1"> <!-- 手机适配，见第三节 -->
    <title>页面标题</title>      <!--    浏览器标签页上显示的标题 -->
    <link rel="stylesheet" href="style.css"> <!-- 引入外部 CSS 文件 -->
  </head>
  <body>                        <!-- ④ 身体：真正显示在屏幕上的所有内容都在这里 -->
    <h1>你好</h1>
    <p>这是一段正文。</p>
    <script src="app.js"></script> <!-- ⑤ 引入 JS 文件，通常放 body 末尾 -->
  </body>
</html>
```

记住四个大块：
| 部分 | 作用 | 类比 |
| --- | --- | --- |
| `<!DOCTYPE html>` | 声明"我是 HTML5" | 文件头魔数 |
| `<head>` | **元信息**：标题、编码、引入 CSS/字体、SEO——**不显示** | 场景的导入设置/元数据 |
| `<body>` | **可见内容**：所有你看到的东西 | 场景里实际摆的物体 |
| `<link>` / `<script>` | 把外部 CSS / JS 文件"接"进来 | 引用外部材质 / 挂脚本 |

> 关键区分：**`<head>` 里的东西不显示，`<body>` 里的才显示。** 新手常犯的错是把要显示的内容写进了 `<head>`。

### 0.2 嵌套与父子：HTML 是一棵树

元素可以**套元素**，形成父子层级——这就是 [[00 前端是什么]] 说的 DOM 树的来源：

```html
<div class="card">          <!-- 父：卡片盒子 -->
  <img src="cat.png">       <!--   子：图片 -->
  <div class="info">        <!--   子：信息盒子（它又是下面两个的父） -->
    <h2>小猫</h2>            <!--     孙：标题 -->
    <p>￥199</p>            <!--     孙：价格 -->
  </div>
</div>
```

对应的树：
```
div.card
├─ img
└─ div.info
   ├─ h2
   └─ p
```
> 完全就是你的 **Hierarchy / 场景节点树**：`div.card` 是父节点，`img` 和 `div.info` 是子节点。CSS 的后代选择器（`.card .info`）、`position: absolute` 找祖先，全都基于这棵树。缩进只是给人看的，真正的父子关系由**谁套在谁的尖括号里**决定。

### 0.3 最常用的标签清单

按用途分组，先眼熟，用到再查：

| 分类 | 标签 | 说明 |
| --- | --- | --- |
| 标题 | `<h1>`~`<h6>` | 一级到六级标题，h1 一页通常只一个 |
| 文本 | `<p>` `<span>` | 段落 / 行内小段文字 |
| 链接图片 | `<a href="...">` `<img src="...">` | 跳转 / 图片 |
| 容器 | `<div>` `<section>` `<header>` `<footer>` | 分组盒子（语义版见第一节） |
| 列表 | `<ul><li>` `<ol><li>` | 无序 / 有序列表，`<li>` 是列表项 |
| 表单 | `<input>` `<button>` `<form>` `<label>` | 输入框 / 按钮 / 表单 / 标签 |
| 表格 | `<table><tr><td>` | 表格 / 行 / 单元格 |

一个典型交互元素——输入框 + 按钮：
```html
<label>邮箱：<input type="email" placeholder="你的邮箱"></label>
<button type="submit">提交</button>
```

**掌握了 0 节，你就能读懂任何一段 HTML 了。** 下面进实用/面试层。

---

## 一、语义化 HTML 与可访问性

**是什么**：用有含义的标签（`<header> <nav> <main> <section> <article> <aside> <footer>`）代替满屏 `<div>`。它们和 `<div>` 的**显示效果一样**（都是块级容器），区别只在"名字自带含义"。

**为什么好**：
- **可访问性（a11y）**：屏幕阅读器能识别结构，视障用户可以"跳到主内容""按标题导航"。`<div>` 对它们只是一团无意义的盒子。
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

**一句话记忆**：语义标签 = 给机器（爬虫、读屏软件）看的结构说明书；`<div>` 是没含义时的兜底容器。

---

## 二、Flexbox 与 Grid 的取舍

这两个是 [[00.3 CSS上手]] 提到的 `display: flex` / `display: grid`——**主动接管排版**的两套布局引擎，替代手动 `float`/`position` 摆盒子。

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
（`1fr` = "剩下的空间全给我"，`fr` 是 Grid 特有的"份数"单位；`gap` 是格子之间的间距。）

### 进阶：flex 简写三兄弟
`flex: <grow> <shrink> <basis>`，常见 `flex: 1` = `flex: 1 1 0%`。

- **`flex-grow: 1`**：有剩余空间时，按比例"瓜分"放大。
- **`flex-shrink: 1`**：空间不够时，按比例"压缩"。设为 `0` 则拒绝被压缩。
- **`flex-basis: 0%`**：分配前的初始基准尺寸。

**踩坑：子元素被压没了怎么办？**
- 给"不想被压缩"的元素设 `flex-shrink: 0`（比如 logo、图标）。
- 或给它设 `min-width`，压缩不会突破 `min-width`。
- 文本溢出用 `min-width: 0` 配合 `overflow: hidden`（flex 子项默认 `min-width: auto` 会撑破容器，这是高频坑）。

---

## 三、响应式设计与移动优先

**响应式（responsive）**：同一套页面，在手机窄屏和电脑宽屏上都能自适应排布，而不是各写一套。

**移动优先（mobile-first）**：先写小屏样式作为默认，再用 `min-width` 媒体查询逐步"加宽加料"。好处是移动端不用覆盖多余样式，性能更好。

四件套：
- **流式布局**：宽度用 `%` / `fr` / `flex`，不写死 `px`。
- **弹性图片**：`img { max-width: 100%; height: auto; }`，图片不超出容器。
- **媒体查询（media query）**：按屏幕宽度切换样式（下面的 `@media`）。
- **断点（breakpoint）**：切换样式的宽度阈值，常见 768px（平板）、1024px（桌面）。

```css
/* 默认：移动端单列 */
.grid { display: grid; grid-template-columns: 1fr; }
/* 平板及以上：两列 */
@media (min-width: 768px) {
  .grid { grid-template-columns: 1fr 1fr; }
}
```

**别忘了**：HTML 的 `<head>` 里要加 `<meta name="viewport" content="width=device-width, initial-scale=1">`（就是第 0 节骨架里那行），否则手机会按桌面宽度缩放渲染。

---

## 四、现代 CSS：变量与函数

**CSS 自定义属性（CSS 变量）**：像编程里的变量，把常用值存起来复用。
```css
:root { --primary: #4f46e5; --gap: 16px; }   /* :root 相当于全局作用域 */
.btn { color: var(--primary); padding: var(--gap); }  /* var() 取用 */
```

**与 SASS 变量的关键区别**（SASS 是一种"增强版 CSS"，写完编译成普通 CSS）：
| | CSS 变量 | SASS 变量 |
| --- | --- | --- |
| 生效时机 | **运行时**，浏览器里可改 | **编译时**，编译后就没了 |
| 能否 JS 修改 | 能（`el.style.setProperty`） | 不能 |
| 作用域 | 跟随 DOM 层级，可继承/覆盖 | 词法作用域 |

正因为"运行时可变"，CSS 变量特别适合**主题切换**：
```css
:root { --bg: #fff; --text: #111; }
[data-theme="dark"] { --bg: #111; --text: #eee; }
body { background: var(--bg); color: var(--text); }
/* JS 里 document.documentElement.dataset.theme = 'dark' 即可整页切换深色 */
```

**实用函数 `clamp(min, preferred, max)`**：一行实现"最小不小于、最大不超过、中间跟随视口"的响应式字号：
```css
h1 { font-size: clamp(1.5rem, 4vw, 3rem); }
```

---

## 五、大型项目的样式架构

**要解决的问题**：CSS 默认是**全局生效**的（一条 `.title{}` 会影响全页所有 `class="title"`），项目一大就会全局污染、命名冲突、选择器过深（影响性能）、巨型 CSS 文件。

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
- 避免频繁触发**回流**（元素尺寸/位置变化导致重新排版，见 [[04 浏览器原理]] 的渲染流水线）。

---

## 面试速答卡

- **语义化好处**？→ a11y + SEO + 可维护。
- **Flex vs Grid**？→ 一维 vs 二维；Grid 搭骨架、Flex 排局部。
- **CSS 变量 vs SASS 变量**？→ 运行时可变 vs 编译时；主题切换靠前者。
- **样式怎么组织**？→ 结合项目说一种（如 Tailwind + CSS Modules），强调隔离和按需。

---

## 速记

- HTML 文件 = `<!DOCTYPE>` + `<html>`（`<head>` 不显示的元信息 + `<body>` 显示的内容）。
- 元素可嵌套成树（= DOM 树 = 你的场景 Hierarchy）；父子关系由谁套在谁里决定。
- 语义标签（header/nav/main/article…）= 有含义的 div，利于 a11y/SEO/维护。
- 布局：Grid 搭二维骨架、Flex 排一维局部；响应式靠流式单位 + `@media` 断点 + viewport meta。
- CSS 变量运行时可变（主题切换）；大项目用 BEM/CSS Modules/Tailwind 做样式隔离。
- 下一步 → [[02.0 JavaScript上手]] 学"行为"，或补 [[05 计算机网络]] 地基。
