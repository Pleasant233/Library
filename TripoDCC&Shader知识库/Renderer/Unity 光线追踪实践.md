# 前置

> 个人自学项目，对应 [[任务需求]] 7.16 个人项的「unity光线追踪」。
> 目标：在 Unity 里用 **Compute Shader** 手写光线追踪，跟的是经典的 *GPU Ray Tracing in Unity*（Three Eyed Games）系列教程。
> 教程基于 **Built-in 管线**，但本项目工程是 **URP 14.0.12 / Unity 2022**，所以第一道坎就是把教程的后处理入口迁移到 URP。着色器管线本身（顶点→图元→片元、RT、Dispatch）你已经会，这里主要是 Unity 侧的 API 外壳。

* 本地工程：`/Users/vast/DccDev/Unity Project/Graphics Learning`
* 核心脚本：`Assets/Raytracing.cs`（挂在 Camera 上）
* 计算着色器：`Assets/NewComputeShader.compute`
* 相关：[[09 前端接WebGL]]（同为「已懂 shader，换一层 API 外壳」的迁移场景）

# 核心踩坑：URP 没有 OnRenderImage

这是本次实践最关键的一条，卡了最久：**脚本挂上相机、Compute Shader 也拖进去了、Console 不报错，但画面毫无变化。**

## 根因

`OnRenderImage` / `OnPreRender` / `OnPostRender` 这套后处理回调是 **Built-in 管线专属**的。URP 把它们全部移除了，所以方法根本不会被 Unity 调用——不是代码错，是入口失效。

> 判断项目是不是 URP：看 `ProjectSettings/GraphicsSettings.asset` 的 `m_CustomRenderPipeline` 是否指向一个 SRP 资产，或 `Packages/manifest.json` 里是否有 `com.unity.render-pipelines.universal`。

顺带澄清另一个易混点：教程里的 `Render()` **不是 Unity 内置方法**，是教程自定义的，需要自己写；`OnRenderImage`（不是 `OderImage`）才是内置回调，且它在 URP 下才失效，Built-in 下依然可用。

## 解法对比

| 方案 | 改动量 | 说明 |
| --- | --- | --- |
| **RenderPipelineManager 回调**（本项目采用） | 小 | 订阅 `endCameraRendering` 事件替代 `OnRenderImage`，教程的 `Render`/`InitRenderTexture` 逻辑几乎原样保留，最快看到画面 |
| 切回 Built-in 管线 | 小 | 教程代码一字不改，但会失去 URP 功能和现有 URP 场景设置 |
| ScriptableRendererFeature | 大 | URP 官方正统做法，最贴近工程实践，学习曲线陡 |

# 实现：RenderPipelineManager 事件回调

思路：在 `OnEnable`/`OnDisable` 订阅/取消订阅 `RenderPipelineManager.endCameraRendering`，事件在每台相机渲染结束后触发，用缓存的相机引用过滤，只处理挂脚本的这台相机。

```csharp
using UnityEngine;
using UnityEngine.Rendering;   // RenderPipelineManager / ScriptableRenderContext 在这个命名空间

[RequireComponent(typeof(Camera))]
public class Raytracing : MonoBehaviour
{
    public ComputeShader RayTracingShader;
    private RenderTexture _target;
    private Camera _camera;

    private void Awake() => _camera = GetComponent<Camera>();

    private void OnEnable()  => RenderPipelineManager.endCameraRendering += OnEndCameraRendering;
    private void OnDisable() => RenderPipelineManager.endCameraRendering -= OnEndCameraRendering;

    private void OnEndCameraRendering(ScriptableRenderContext context, Camera camera)
    {
        if (camera != _camera) return;   // 只处理本相机，避免影响 Scene 视图等其它相机
        Render();
    }

    private void Render()
    {
        InitRenderTexture();
        RayTracingShader.SetTexture(0, "Result", _target);
        int gx = Mathf.CeilToInt(Screen.width  / 8.0f);
        int gy = Mathf.CeilToInt(Screen.height / 8.0f);
        RayTracingShader.Dispatch(0, gx, gy, 1);
        Graphics.Blit(_target, (RenderTexture)null);   // null = 当前相机目标（屏幕）
    }

    private void InitRenderTexture()
    {
        if (_target == null || _target.width != Screen.width || _target.height != Screen.height)
        {
            if (_target != null) _target.Release();
            _target = new RenderTexture(Screen.width, Screen.height, 0,
                RenderTextureFormat.ARGBFloat, RenderTextureReadWrite.Linear);
            _target.enableRandomWrite = true;   // Compute Shader 随机写入必须开
            _target.Create();
        }
    }
}
```

## 要点

* **必须成对订阅/取消订阅**事件，否则脚本禁用后回调仍在跑，会报错或重复执行。
* `Graphics.Blit(src, (RenderTexture)null)`：URP 下 `null` 目标表示画到当前相机的渲染目标（屏幕）。注意要显式转型 `(RenderTexture)null` 消歧义。
* `[RequireComponent(typeof(Camera))]` + `Awake` 缓存相机，既保证脚本只挂相机，又避免每帧 `GetComponent`。
* `RenderTextureFormat.ARGBFloat`：HDR 浮点，适合光追累积；`enableRandomWrite = true` 是 Compute Shader 能写入的前提，漏了就是黑屏。

# Compute Shader 侧的对齐约定

脚本用 `SetTexture(0, "Result", _target)` + `Dispatch(0, ...)`，所以 `.compute` 必须满足：

* 输出变量名是 **`Result`**，类型 `RWTexture2D<float4>`
* 目标是**第 0 个 kernel**（`#pragma kernel` 的第一个）
* 线程组 `[numthreads(8,8,1)]`，和脚本里的 `/8.0f` 对应

黑屏排查时先用「渐变测试色」确认管线通不通，再写真正的光追逻辑：

```hlsl
#pragma kernel CSMain
RWTexture2D<float4> Result;

[numthreads(8,8,1)]
void CSMain (uint3 id : SV_DispatchThreadID)
{
    uint w, h;
    Result.GetDimensions(w, h);
    Result[id.xy] = float4(id.x / (float)w, id.y / (float)h, 0.2, 1);
}
```

# 已知待观察问题

* URP 相机若开了 **Post Processing**，`endCameraRendering` 之后的 Blit 时机可能和 URP 自身后处理叠加，出现闪烁。若遇到再考虑改用 `beginCameraRendering` 或走 ScriptableRendererFeature。

# 进展：相机射线构建

在 Compute Shader 里从屏幕像素反推出世界空间射线，输出 `ray.direction*0.5+0.5` 做方向可视化（屏幕出现随视角变化的彩色渐变即为通了）。

两边配合：C# 传相机矩阵，shader 用矩阵把像素 → 世界射线。

```csharp
// Raytracing.cs：每帧把相机矩阵传给 shader
private void SetShaderParameters()
{
    RayTracingShader.SetMatrix("_CameraToWorld", _camera.cameraToWorldMatrix);
    RayTracingShader.SetMatrix("_CameraInverseProjection", _camera.projectionMatrix.inverse); // 注意 .inverse
}
```

```hlsl
// origin = 相机位置（相机空间原点 → 世界）
float3 origin = mul(_CameraToWorld, float4(0,0,0,1)).xyz;
// 像素方向：先用逆投影矩阵回到相机空间，再转到世界空间
float3 dir = mul(_CameraInverseProjection, float4(uv, 0, 1)).xyz;
dir = normalize(mul(_CameraToWorld, float4(dir, 0)).xyz);
```

其中 `uv` 是像素中心归一化到 `[-1,1]` 的 NDC 坐标：`(id.xy + 0.5) / 分辨率 * 2 - 1`。

## 本次两个典型坑

* **编译错误会让所有 kernel 失效**：`float4(0.0f 0.0f, ...)` 漏了个逗号 → shader 编译失败 → `Dispatch(0,...)` 报 `Kernel at index (0) is invalid`。看到「kernel index invalid」先去查 shader 有没有编译错，而不是怀疑 kernel 名或索引。
* **矩阵名要和内容一致**：shader 变量叫 `_CameraInverseProjection`，C# 就必须传 `projectionMatrix.inverse`（逆矩阵），传成 `projectionMatrix` 本身不报错但射线方向全错。

# 进展：渐进式抗锯齿（progressive AA）

思路：每帧对像素采样位置加一个随机亚像素抖动 `_PixelOffset`，再把历帧结果**加权平均**收敛。相机不动时，锯齿随采样数增加而逐渐平滑；相机一动就清零重来。

## 累积靠硬件混合，AddShader 只有一句

`AddShader` 带 `Blend SrcAlpha OneMinusSrcAlpha`，片元输出 alpha = `1/(_Sample+1)`。硬件混合公式 `dst = src·a + dst·(1-a)` 代入即：

```
dst_new = 本帧 · 1/(n+1) + dst_old · n/(n+1)   // 运行平均
```

**累积是硬件在「目标 RT」上做的**，所以片元 shader 一句 `tex2D` 就够，教程原版也确实只有一句。前提是**目标 RT 会保留上一帧内容**。

## 原文一行搞定，我这里为什么要改（关键教训）

教程原版直接 `Graphics.Blit(_target, destination, _addMaterial)`，把累积做在 `OnRenderImage` 给的 `destination`（屏幕）上，一行搞定。**但这依赖「帧缓冲保留上一帧内容」这个平台相关假设**：

* 教程环境 = Built-in + Windows/DX11，`destination` 恰好保留旧内容，取巧成立。
* 本项目 = URP + macOS/Metal，且累积目标是屏幕（swapchain 多缓冲轮换 + Metal tile 架构默认不 load 旧内容）→ 混合读到的 `dst_old` 是常量垃圾 → 运行平均收敛到常数 → **整屏一片灰**。

**修法（最小改动）**：别在屏幕上累积，改成累积到**自己持有的一张 RT**（`_accumulate`），它作渲染目标时硬件会 load 旧内容；再单独显示到屏幕。AddShader 一字不改，只多一张 RT：

```csharp
Graphics.Blit(_target, _accumulate, _addMaterial); // 累加进自己的 RT（保留旧内容）
Graphics.Blit(_accumulate, (RenderTexture)null);   // 再显示到屏幕
```

> 教训：跟平台无关的算法（运行平均）没错，错的是**照搬了教程对帧缓冲行为的隐式假设**。换了管线/平台，先怀疑这类「谁保留内容、谁被清空」的时序假设。ping-pong 双缓冲能更保险地绕开，但对这个场景是过度设计——单张自持 RT 足够。

## 本次三个坑（累积 AA 完全不生效，三者叠加，各自都是致命的）

* **C# 属性名和 shader 对不上**：C# 设 `_Sample`，但 AddShader 里属性叫 `_SampleRate` → 值永远是默认 `1.0`，权重恒为 `1/2`，每帧 50/50 混合，永不收敛。**shader 属性名必须和 `SetFloat/SetVector` 的字符串逐字一致**。
* **在屏幕上累积 → 一片灰**：把累积 blit 到屏幕（swapchain 不保留上一帧、Metal 默认不 load），硬件混合读到的历史值是常量，运行平均收敛到常数。见上一节「关键教训」，改成累积到自己持有的 `_accumulate` RT 再显示。
* **`transform.hasChanged` 不可靠**：挂了移动脚本后，`Controller` 每帧写 `transform.position`（即使加零向量），也会把 `hasChanged` 置 true → 采样计数每帧清零，永远累积不起来。改为**手动比较上一帧的 position/rotation**判断相机是否真的动了。

# 后续学习计划

> 随学习进度持续维护，做到哪补到哪。

* [x] Compute Shader 里搭建相机射线（从 `_CameraToWorld` / `_CameraInverseProjection` 生成 ray）
* [x] 地面平面 + 球体求交，输出法线可视化
* [x] 天空盒采样（equirectangular 2D 全景图，非 Cubemap）
* [x] 渐进式抗锯齿（亚像素抖动 + 收敛缓冲加权平均）
* [ ] 反射与多次弹射（累积 + 逐帧收敛降噪）
* [ ] 性能：分辨率缩放、线程组尺寸调优
* [ ] 评估迁移到 ScriptableRendererFeature 的正统做法

## 配套：相机控制器 Controller.cs

飞行相机 MVP，用于验证射线/天空盒随视角实时变化：

* 鼠标左键拖拽旋转（yaw + pitch，pitch 限 ±89° 防翻转）
* WASD 沿 `transform.forward`/`transform.right` 移动，`dir.normalized * (speed * Time.deltaTime)` 保证斜向不加速、帧率无关
* 注意：它每帧写 `transform.position` 会触发上面第三个坑

#Renderer
