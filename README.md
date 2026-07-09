# Library · 实习学习记录

> 我在 Tripo 实习期间的个人知识库，用 [Obsidian](https://obsidian.md) 管理，通过 Git 同步。
> 记录 DCC 插件开发、渲染器、Shader，以及为对接前端工作而系统整理的 Web 知识。

## 📚 内容导航

### Web 前端（从入门到深入）
为对接前端同事、补齐 Web 生态而整理，按「入门 → 语言基础 → 运行环境 → 框架 → 业务专项」组织。

| 阶段 | 文档 |
| --- | --- |
| 🧭 索引 | 前端知识地图 |
| 入门 | 00 前端是什么 / 00.1 环境与工具链 / 00.2 与同事对接 |
| 语言基础 | 01 HTML与CSS / 02 JavaScript核心 / 03 TypeScript |
| 运行环境 | 04 浏览器原理 / 05 计算机网络 |
| 框架 | 06 Vue与React / 07 Nuxt与SSR |
| 专项 | 08 Web3D与AI |
| 业务专项 | 09 前端接WebGL / 10 前端接WebSocket |
| 面试 | 面试题库 |

> 路径：`TripoDCC&Shader知识库/Web/`

### DCC 插件
| 主题 | 说明 |
| --- | --- |
| Unity Bridge | Tripo3D Unity Bridge 插件学习笔记（WebSocket + 分片传输 + 模型导入） |
| Unity 开发相关 | 项目概览与开发记录 |

> 路径：`TripoDCC&Shader知识库/DCC/`

### 渲染器
| 主题 | 说明 |
| --- | --- |
| Tripo Orbit | 轻量级桌面端 3D 模型查看器（Rust + OpenGL/WebGPU） |

> 路径：`TripoDCC&Shader知识库/Renderer/`

## 🗂️ 目录结构

```
.
├── TripoDCC&Shader知识库/
│   ├── Web/            # 前端学习文档（本库重点）
│   ├── DCC/            # DCC 插件（Unity 等）
│   ├── Renderer/       # 渲染器
│   ├── 任务需求.md      # 当前工作方向与进度
│   └── 文档整理.md      # 外部文档索引
├── Resource/           # 图片等资源
└── .obsidian/          # Obsidian 配置（主题/插件/快捷键，随库同步）
```

## 🔧 使用说明

- **推荐用 Obsidian 打开**：双链 `[[...]]`、关系图谱等功能才能正常使用。将本仓库根目录作为 Vault 打开即可。
- 直接在 GitHub 上阅读 `.md` 也可以，但双链不会跳转。

## 🔄 同步方式

通过 Git 同步，日常使用 **Obsidian Git** 插件自动 commit + push（间隔约 10 分钟）。

手动同步：

```bash
git add -A && git commit -m "docs: 更新笔记" && git push
```

---

> 持续维护中 🚧
