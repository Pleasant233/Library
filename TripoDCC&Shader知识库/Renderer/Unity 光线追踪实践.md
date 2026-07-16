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

# 后续学习计划

> 随学习进度持续维护，做到哪补到哪。

* [ ] Compute Shader 里搭建相机射线（从 `_CameraToWorld` / `_CameraInverseProjection` 生成 ray）
* [ ] 地面平面 + 球体求交，输出法线可视化
* [ ] 反射与多次弹射（累积 + 逐帧收敛降噪）
* [ ] 天空盒采样
* [ ] 性能：分辨率缩放、线程组尺寸调优
* [ ] 评估迁移到 ScriptableRendererFeature 的正统做法

#Renderer
