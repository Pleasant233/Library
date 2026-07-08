# 08 Web3D 与 AI

> 配套 [[面试题库]] 的「专项类」部分。这块和 Tripo 的实际业务（[[Tripo Orbit]]、Shader）最相关，值得重点准备。

---

# 一、Web3D

## 1. 脏节点更新（Dirty Flag）

**背景**：3D 场景是一棵节点树，每个节点有 `localMatrix`（相对父节点）和 `worldMatrix`（世界坐标）。世界矩阵 = 父世界矩阵 × 本地矩阵。

**问题**：每帧都重算所有 worldMatrix 太浪费。**脏标记**思路：只有变过的节点才重算，用缓存。

**关键逻辑**：
- 修改自己的 transform → 把自己**和所有后代**标记为「脏」（因为父变了，子的世界矩阵也失效）。
- 取 worldMatrix 时：脏了才重算并缓存、清除脏标记；没脏直接返回缓存。

```ts
import { mat4 } from "gl-matrix";

class Node3D {
  localMatrix = mat4.create();
  private _worldMatrix = mat4.create();
  private _isDirty = true;
  private _children: Node3D[] = [];
  private _parent: Node3D | null = null;

  updateTransform(newMtx: mat4): void {
    mat4.copy(this.localMatrix, newMtx);
    this._markDirty();               // 自己和后代都脏
  }

  private _markDirty(): void {
    if (this._isDirty) return;       // 已脏则剪枝，避免重复遍历
    this._isDirty = true;
    for (const c of this._children) c._markDirty();
  }

  getWorldMatrix(): mat4 {
    if (this._isDirty) {
      if (this._parent) {
        mat4.multiply(this._worldMatrix, this._parent.getWorldMatrix(), this.localMatrix);
      } else {
        mat4.copy(this._worldMatrix, this.localMatrix);  // 根节点
      }
      this._isDirty = false;         // 闭合脏标记（缓存生效）
    }
    return this._worldMatrix;
  }
}
```
**考点**：懒计算 + 缓存 + 脏标记剪枝（已脏就不用再往下传）。

---

## 2. 模糊 Shader（1D 高斯采样）

**背景**：高斯模糊可**线性可分**——二维模糊 = 先横向一维 + 再纵向一维，采样次数从 N² 降到 2N。

**任务**：水平方向半径 2，中心 + 左右各 2 像素 = 5 次采样，权重 `uW = [0.4, 0.2, 0.1]`。

```glsl
uniform sampler2D uTexture;
uniform vec2 uResolution;
uniform float uW[3];   // [中心权重, 偏移1, 偏移2]
varying vec2 vUV;

void main() {
    vec2 texel = 1.0 / uResolution;          // 一个像素的 UV 步长
    vec3 color = texture2D(uTexture, vUV).rgb * uW[0];        // 中心
    for (int i = 1; i <= 2; i++) {
        vec2 off = vec2(texel.x * float(i), 0.0);            // 水平偏移
        color += texture2D(uTexture, vUV + off).rgb * uW[i]; // 右
        color += texture2D(uTexture, vUV - off).rgb * uW[i]; // 左
    }
    gl_FragColor = vec4(color, 1.0);
}
```
**考点**：理解 UV、texel 步长、对称采样、可分离卷积。

---

## 3. 高性能自发光描边（后处理）

**背景**：传统法线扩充描边处理不了硬边断裂。改用**基于物体 Mask 的后处理描边**。

**思路**（说清即可，无需完整写）：
1. **Mask 阶段**：把目标物体渲成一张 mask 图（物体=1、背景=0）。
2. **扩边**：在片元里对 mask 采样邻域，物体外围但靠近物体的像素得到一个衰减的 glow 值。可用：
   - **模糊 / 距离场**：模糊 mask，边缘外产生渐变；或用 **JFA（Jump Flood）** 算距离场，做精确的等距发光。
3. **合成**：`最终色 = 场景色 + glowColor × glowAlpha`，物体内部不叠加。
4. **性能**：用**线性采样（Linear Sampling）**——在两个纹素中间采样一次，硬件插值一次拿到两个像素的加权平均，**采样次数减半**。

```glsl
// 简化版：邻域采样求最大 mask，物体外侧产生衰减 glow
float glow = 0.0;
for (float x = -2.0; x <= 2.0; x++) {
  for (float y = -2.0; y <= 2.0; y++) {
    vec2 off = vec2(x, y) * uTexelSize * uRadius;
    float m = texture2D(uMaskTexture, vUv + off).r;
    float d = length(vec2(x, y));
    glow = max(glow, m * (1.0 - d / 3.0));   // 距离越远越弱
  }
}
glow *= step(centerMask, 0.5);   // 物体内部(centerMask>0.5)不发光
gl_FragColor = vec4(sceneColor.rgb + uGlowColor * glow, 1.0);
```
**考点**：后处理管线、距离衰减、线性采样优化。

---

## 4. ECS 查询优化与位掩码

**背景**：ECS（Entity-Component-System）里，System 每帧要查「同时拥有某几个组件」的实体。用数组求交集慢，用**位掩码**一次位运算搞定。

- 每种组件 = 一个二进制位：`Position=1<<0`、`Velocity=1<<1`、`Render=1<<2`。
- 实体的组件集合 = 各位 **OR** 起来的一个整数。
- 「是否拥有查询要求的所有组件」= `(mask & query) === query`。

```ts
class EntityManager {
  private entityMasks: Map<number, number> = new Map();

  addComponent(entityId: number, componentBit: number): void {
    const cur = this.entityMasks.get(entityId) ?? 0;
    this.entityMasks.set(entityId, cur | componentBit);   // OR 并入
  }

  match(entityId: number, queryMask: number): boolean {
    const mask = this.entityMasks.get(entityId) ?? 0;
    return (mask & queryMask) === queryMask;              // 完全包含
  }

  query(queryMask: number): number[] {
    const result: number[] = [];
    for (const [id, mask] of this.entityMasks) {
      if ((mask & queryMask) === queryMask) result.push(id);
    }
    return result;
  }
}
```
**考点**：位运算（`|` 加组件、`&` 求交、`=== query` 判包含），比集合求交快得多。

---

# 二、AI（前端向）

> 这块几乎全是「概念题」，考的是理解不是代码。抓住每个词「解决什么问题」。

## 核心概念区分

| 概念 | 是什么 | 解决什么 |
| --- | --- | --- |
| **Model** | 底座模型（权重） | 生成 / 推理的基础 |
| **LLM** | 大语言模型 | 理解和生成自然语言 |
| **RAG** | 检索增强生成 | 「模型不知道最新/私有信息」→ 先检索再回答 |
| **Function Calling** | 让模型按结构调用工具 | 让模型能查天气、下单、发请求 |
| **Agent** | 目标 + 计划 + 工具 + 记忆 | 自主完成**多步**任务 |

一句话串起来：**LLM 负责想，RAG 补知识，Function Calling 给它手，Agent 让它自己规划着干活。**

## AI 聊天前端三件事：流式 / 停止 / 重新生成

- **流式输出**：用 **SSE**（最常见）/ fetch stream / WebSocket，把 token 逐段追加进消息气泡（token 级增量渲染）。
- **停止生成**：`AbortController` 中断请求，把当前消息状态标记为「已停止」。
- **重新生成**：保留原 prompt + 历史上下文，重新发一次请求。
- **状态机**：消息分 `user / assistant / tool / error` 和 `生成中 / 已停止 / 可重新生成` 状态，避免 UI 乱跳。

## Prompt / Context Engineering

- **Prompt Engineering**：把**任务、约束、输出格式、示例**写清楚。稳定的代码生成 prompt 要给：技术栈、目录结构、命名规则、边界条件、输出格式；复杂任务**先出计划再出代码**，或拆多轮。
- **Context Engineering**：管理模型上下文窗口里的信息（系统提示、工具定义、检索结果、历史、记忆）。核心原则：**高信噪比、可复用、可评估**。让 AI 更懂代码库 → 提供架构说明、代码规范、组件示例、API 文档、错误处理约定。

## 工程实践态度（面试很看重）

- AI 适合**起草和辅助思考**，不适合**无审查直接合并**。
- 生成代码重点审查：**边界条件、可访问性、性能、安全、依赖**。
- **Vibe Coding**：更依赖 AI 快速生成试错，适合原型/demo/学新 API，**不适合**支付/权限等关键路径。
- 定位 AI 生成错误：分四层排查 → **prompt 是否明确 → 上下文是否缺失 → 模型能力是否够 → 代码逻辑是否对**。把 AI 当「可能出错的同事」。

---

## 面试速答卡

- **脏节点为什么高效**？→ 懒计算 + 缓存，只有脏了才重算，标记剪枝。
- **高斯模糊为什么分两次**？→ 线性可分，N² 降到 2N。
- **ECS 位掩码判包含**？→ `(mask & query) === query`。
- **RAG/FC/Agent 区别**？→ RAG 补知识、FC 给工具、Agent 会规划多步。
- **AI 生成代码错了怎么查**？→ prompt→上下文→模型→代码 四层定位。
