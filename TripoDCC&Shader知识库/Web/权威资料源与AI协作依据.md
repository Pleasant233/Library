# 权威资料源与 AI 协作依据

> 本篇是知识库的**一手权威源清单**，由 Codex 联网调研、逐个 `curl` 核验存在性（2026-07 核验，均返回 200）后整理，我再落库。用途：查资料时**优先查这里的一手源**，而不是随机博客/教程；也是 [[学习路径设计方法论]] 原则 6「搭建权威知识源」的落地。
> 更新方式：让 Codex 外勤联网补充新源、核验 URL，再回写本篇（协作流程见文末）。

---

## 一、前端学习的权威一手资料源

| 名称 | 权威在哪 | 链接 |
| --- | --- | --- |
| **MDN Web Docs** | Mozilla 维护，系统覆盖 Web 标准、浏览器 API 与兼容性数据——**查语法/API 的首选** | https://developer.mozilla.org/en-US/docs/Web |
| **javascript.info** | 面向现代 JavaScript 的完整教程体系，兼顾语言基础、浏览器与工程实践 | https://javascript.info/ |
| **roadmap.sh Frontend** | 以可执行学习路径梳理前端知识图谱，适合查漏补缺与规划进阶 | https://roadmap.sh/frontend |
| **web.dev Learn** | Google Chrome 团队出品，强调性能、可访问性与现代最佳实践 | https://web.dev/learn |
| **Vue 官方文档** | Vue 核心团队维护，定义框架 API、设计理念与推荐实践 | https://vuejs.org/guide/introduction.html |
| **Three.js 官方文档** | 官方 API 参考与示例入口，WebGL/3D 开发的直接依据（对应你的图形主场） | https://threejs.org/docs/ |

> 用法建议：概念先问 AI 建立直觉 → 细节以这些一手文档为准 → 卡壳的官方页可喂 NotebookLM/RAG 做可问答库（见 [[学习路径设计方法论]]）。

---

## 二、「AI 全代写时代，人还必须掌握什么」——有据可查的观点

支撑本库"新规范：AI 写代码、你练读/判/查/定"的外部依据（均来自 2025 权威来源）：

1. **测试、调试与验收仍是人的责任**：代码是否真能跑、是否满足需求，责任在人；AI 的实现不能跳过验证。
   —— Simon Willison, *Here's how I use LLMs to help me write code* (2025)。https://simonwillison.net/2025/Mar/11/using-llms-for-code/
2. **代码审查与可维护性判断不可外包**：AI 容易产出重复、脆弱或过度复杂的实现，长期维护成本要由人判断。
   —— Martin Fowler / Thoughtworks, *The Role of Developer Skills in Agentic Coding* (2025)。https://martinfowler.com/articles/exploring-gen-ai/13-role-of-developer-skills.html
3. **必须理解架构、依赖与运行环境**：定位跨平台、构建、部署类问题依赖经验性的系统理解，不能只信模型的表面解释。
   —— 同上（Fowler / Thoughtworks）。https://martinfowler.com/articles/exploring-gen-ai/13-role-of-developer-skills.html
4. **默认验证、而非盲信输出**：Stack Overflow 2025 开发者调查中，**46% 不信任** AI 工具输出的准确性、仅 33% 信任——人工复核仍是工程流程的一环。
   —— Stack Overflow Developer Survey 2025 (AI)。https://survey.stackoverflow.co/2025/ai
5. **"上下文工程"与需求表达是新核心技能**：讲清技术约束、既有代码、示例与验收条件，才能让 AI 从"自动补全"变成"受控协作者"。
   —— Simon Willison (2025)。https://simonwillison.net/2025/Mar/11/using-llms-for-code/

> 这 5 条正好印证 [[11 实战项目路线（AI协作版）]] 的设计：每级都练"验证 + 审查 + 架构判断 + 把需求讲清给 AI"，而不是练手打代码。

---

## 三、这份清单是怎么来的（可复用的协作流程）

本篇是"主 Agent（Claude）+ 外勤 Agent（Codex）"协作的产物，流程可复用：

1. 主 Agent 出**调研规格**（要哪些源、什么主题、硬性要求 URL 真实可核）。
2. `codex exec --dangerously-bypass-approvals-and-sandbox` 派给 Codex 联网执行；它用 `curl` 逐个核验 URL（本次自动剔除了一个 404）。
3. 主 Agent 把回传结果**核对、去重、结构化**，落成本篇双链笔记。

> 以后要给别的领域（Roblox/USD/WebGPU…）建权威源清单，复用这套：换调研主题即可。

## 速记

- 查资料优先级：本篇一手源 > 二手教程 > 随机博客。
- 前端一手源：MDN（API）、javascript.info（JS）、roadmap.sh（路径）、web.dev（最佳实践）、Vue/Three.js 官方（框架/图形）。
- AI 时代人的不可替代项：测试验收、代码审查、架构/环境理解、默认验证、上下文工程——见上 5 条外部依据。
- 相关 → [[学习路径设计方法论]]、[[11 实战项目路线（AI协作版）]]、[[前端知识地图]]。
