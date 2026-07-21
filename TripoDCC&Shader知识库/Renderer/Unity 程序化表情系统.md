# 前置

> [[Unity 软体模拟实践]] 的表情部分。给史莱姆加一张会眨眼、朝向运动方向的脸。
> 关注点：**为什么用独立 billboard 组件而非塞进主 shader** + **SDF 程序化画眼睛** + **朝向/眨眼的时间平滑**。

* 组件：`Assets/Scripts/SoftBody/SoftBodyFace.cs`
* Shader：`Assets/Shader/SoftSlime/SlimeFace.shader`（程序化，不用贴图）

# 架构决策：独立 billboard，不塞主 shader

一开始试过把眼睛画进史莱姆主 shader（[[Unity 半透明果冻 Shader]] 的精灵方案），问题不断：
* 精灵贴 mesh 表面 → 随软体形变拉伸变形，眼睛歪扭
* 双 pass 下正/背面各画一次 → 透过半透明外壳看到镜像的第二只眼，orbit 时漂移
* UV 依赖 mesh 顶点 UV，形变时不稳定

**最终改为独立组件**：一个单独的 quad（billboard）朝相机，贴在质心表面外侧，用自己的 shader 程序化画眼睛。

收益：
* **脸不受 mesh 形变影响**——它是独立 quad，只跟随质心位置和整体朝向/尺度，不跟单个顶点走 → 眼睛始终端正
* **只画一次**，无双 pass 镜像问题
* **和物理/主渲染完全解耦**，各自演化

> 教训：一个视觉元素若**不该跟随载体的局部形变**（脸就该整体稳定），把它做成**独立的、只吃载体整体状态（质心/朝向/尺度）的组件**，比嵌进载体自身的着色更稳、更好调。

# 组件如何吃软体的整体状态

`SoftBodySimulation` 暴露给脸的接口（每帧从 surface 质点算）：
* `SurfaceCenter`：surface 质心（脸贴附的锚点）
* `VisualExtents` / `VisualRadius` / `FaceSurfaceRadius`：整体尺度（脸的大小/贴附距离）
* `FaceDirection`：朝向（运动方向在水平面的投影；静止时回退到一个默认朝向）

`SoftBodyFace.LateUpdate` 用这些把 quad 摆到 `SurfaceCenter + smoothedDir * (surfaceRadius * inset)`，`LookRotation(dir)` 朝外，`localScale` 跟随整体尺寸。

## 时间平滑，避免抖动

朝向和尺度都用**帧率无关的指数平滑**：

```
t = 1 - exp(-responsiveness * deltaTime)
smoothedDir = Slerp(smoothedDir, targetDir, t)   // 方向用 Slerp
smoothedScale = Lerp(smoothedScale, targetScale, t)
```

* 朝向用较高 responsiveness（转脸跟手），尺度用较低（挤压拉伸时脸别乱缩）
* `1 - exp(-k·dt)` 是标准的帧率无关插值率，和求解器里 `ToProjectionFraction` 同源。

# SDF 画眼睛 + 眨眼

`SlimeFace.shader` 在 quad 的 UV 空间用 SDF 画两只椭圆眼 + 瞳孔，不用贴图：

* 椭圆 mask：`EllipseMask(uv, center, radii)`，`smoothstep` 出软边（`_EdgeSoftness`）
* 左右眼：中心对称偏移 `_EyeSpacing`
* 瞳孔：更小的椭圆叠在眼白上
* **眨眼**：`_BlinkAmount`（0→1）把眼睛的**高度**从 `_EyeHeight` 压到 `_ClosedEyeHeight`，闭眼是一条缝

眨眼节奏在 C# 侧驱动（`SoftBodyFace.UpdateBlink`）：按 `blinkInterval` 触发，`blinkDuration` 内用一个 0→1→0 的曲线驱动 `_BlinkAmount`，通过 `MaterialPropertyBlock` 传给 shader。

* 渲染队列 `Transparent+20`：确保脸画在史莱姆本体之后（在其表面之上）

> SDF 画简单表情比贴图省事：分辨率无关、参数化（眼距/大小/瞳孔/软边都是 Range 滑块）、眨眼只是改一个高度参数。适合程序化、可调、无美术资源的场景。

# 参数与生命周期

* 所有参数走 `[SerializeField]` + `OnValidate` clamp，Inspector 可调
* `MaterialPropertyBlock` 传参，不实例化材质、无 GC
* `OnDestroy` 清理动态创建的 quad object / mesh / material
* 属性 ID 用 `Shader.PropertyToID` 静态缓存

# 关联

* 载体物理/整体状态来源：[[Unity 软体模拟实践]]
* 被取代的精灵贴表面方案 + 双 pass 镜像问题：[[Unity 半透明果冻 Shader]]

#Renderer
