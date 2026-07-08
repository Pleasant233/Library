# 07 Nuxt 与 SSR

> 配套 [[面试题库]] 的「Nuxt」部分。先理解 SSR 概念，再看 Nuxt 怎么落地。

## 先搞清渲染模式

| 模式 | 全称 | 谁来生成 HTML | 特点 |
| --- | --- | --- | --- |
| **CSR** | 客户端渲染 | 浏览器跑 JS 生成 | 首屏慢、SEO 差；纯 SPA |
| **SSR** | 服务端渲染 | 服务器**每次请求**生成 | 首屏快、SEO 好；服务器有开销 |
| **SSG** | 静态生成 | **构建时**预生成 HTML | 最快、可上 CDN；内容不实时 |
| **Hybrid** | 混合渲染 | 按页面各选各的 | 灵活，实战常用 |

- **Hydration（水合）**：SSR/SSG 先给一份「静态 HTML」让用户秒看到，然后浏览器加载 JS「激活」它，绑定事件变成可交互的 SPA。
- **Hydration mismatch**：服务端和客户端渲染出的内容不一致（如用了随机数、时间），会报警告并重渲染，要避免。

---

## 一、Nuxt 相比纯 Vue SPA 的核心价值

Nuxt 是 Vue 的 **meta-framework**（元框架），解决纯 SPA 的痛点：

- **SEO / 首屏**：SPA 首屏是空白 + JS 加载，爬虫和用户体验都差；Nuxt 提供 SSR/SSG。
- **约定优于配置**：
  - **文件路由**：`pages/` 目录结构 = 路由，不用手写 router 配置。
  - **自动导入**：组件、composable、API 自动可用，不用到处 import。
  - **服务端能力**：内置 server routes、middleware。

**什么时候选 Nuxt**：内容站、营销页、需要 SEO 的页面、需要服务端逻辑的应用。
**什么时候纯 Vue + Vite 就够**：纯内部后台、纯客户端工具（如你做的 [[Tripo Orbit]] 这类编辑器），不需要 SEO。

---

## 二、Nitro 是什么

**Nitro** 是 Nuxt 的**服务器引擎**，负责：
- **server routes**（`server/api/`）、**middleware**、生产构建。
- **Universal Deployment（一次编写，处处部署）**：同一份代码通过不同 **preset** 适配 Node server、Vercel、Netlify、Cloudflare Workers 等平台，Nitro 负责差异适配。

**为什么是关键能力**：
- 面向 **边缘计算 / serverless**——部署到离用户最近的边缘节点，降低延迟（对全球分发、AI 产品很重要）。
- 你不用为每个平台重写部署逻辑，换个 preset 即可。

---

## 三、SSR / SSG / CSR / Hybrid 怎么选

按页面类型「分而治之」，这是 Nuxt 的典型用法（**页面级混合策略**）：

| 页面 | 推荐模式 | 理由 |
| --- | --- | --- |
| 营销页 / 落地页 | **SSG** | 内容基本不变，预生成上 CDN 最快，SEO 好 |
| 登录页 | SSG / SSR | 结构固定 |
| 控制台 / Dashboard | **CSR**（或 SSR 骨架） | 登录后才看，不需要 SEO，交互重 |
| AI 聊天页 | **CSR** 为主 | 高度动态、实时流式，SSR 意义不大 |

> 关键是理解每种模式在**性能、SEO、首屏、部署成本**上的取舍，而不是全站一刀切。

---

## 四、useAsyncData / useFetch / useState 协同

这三个是 Nuxt 在 SSR 下的数据/状态 API：

- **`useFetch`**：请求数据的「一站式」封装（内部就是 `useAsyncData` + `$fetch`）。最常用。
- **`useAsyncData`**：更底层，包裹任意异步逻辑，适合需要自定义请求方式时。
- **`useState`**：**SSR 友好的**全局响应式状态（替代直接用 `ref`，因为普通 ref 在服务端会跨请求共享，有污染风险）。

**SSR 下它们怎么配合避免重复请求**：
1. **服务端预取**：请求在服务器执行，数据填进 HTML。
2. **序列化**：数据随 HTML 传给浏览器（`payload`）。
3. **客户端复用**：水合时前端**直接用 payload 里的数据**，不再重复请求。靠请求的 `key` 去重。

**设计首屏数据流的原则**：
- **减少 waterfall（瀑布式串行请求）**：能并行的用 `Promise.all` / 同时发起多个 `useAsyncData`，别一个等一个。
- **避免 hydration mismatch**：服务端和客户端用同一份数据、同一个 key，别在渲染里用随机值/本地时间。

---

## 面试速答卡

- **Nuxt 比 SPA 强在哪**？→ SSR/SSG + 文件路由 + 自动导入 + 服务端能力。
- **Nitro 是啥**？→ Nuxt 的服务器引擎，一次编写处处部署（preset 适配边缘/serverless）。
- **渲染模式怎么选**？→ 营销页 SSG、控制台 CSR，页面级混合。
- **SSR 怎么不重复请求**？→ 服务端预取 → 序列化进 payload → 客户端按 key 复用。
