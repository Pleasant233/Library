# 前置

> [[Unity 软体模拟实践]] 的渲染部分。软体物理跑通后，给史莱姆做半透明果冻质感的表面 shader。
> 关注点：**凹形半透明网格的正确渲染** + **多层菲涅尔/深度描边** + **精灵动画**。物理侧不在此，见 hub 笔记。

* Shader：`Assets/Shader/SoftSlime/Soft.shader`（Built-in CG 写法，跑在 URP 下）
* 球形法线（球心用质心）见 [[Unity 软体模拟实践]] 的「渲染：球形法线 + 质心球心」一节，此处不重复。

# 双 pass 解决半透明自穿插

**问题**：史莱姆是凹凸形变的闭合网格，半透明（`ZWrite Off`）时同一 mesh 的三角面之间没有深度遮挡，只靠绘制顺序混合。单个 `Cull Back` 剔除背面后，剩下的正面之间仍乱序 → 落地折叠时露出「破碎面」。

**解法**：双 pass，先内后外
* Pass 1 `Cull Front`：先画背面（远侧）
* Pass 2 `Cull Back`：再画正面（近侧）叠上

这样同一物体的半透明混合有了稳定的「背面在下、正面在上」顺序，自穿插消除，同时保留通透感。

**工程写法**：两个 pass 的着色逻辑抽到 `CGINCLUDE`，pass 里只声明入口 + 各自 `Cull`，避免复制代码。

> 半透明凹形网格本身没有单 pass 的正确解，这是透明渲染的固有难题。不透明物体靠深度写入自动正确；透明只能靠画法约束顺序。

# 球形法线在双 pass 下要用 abs

双 pass 引入后，背面片元的球形法线指向远离摄像机，`dot(N,V)` 为负。菲涅尔若用 `1 - saturate(dot)`，负值被夹成 0 → fresnel 恒为 1 → 整个背面泛白盖住边缘光。

**修法**：用 `1 - abs(dot(N,V))`。只看法线与视线的夹角，正/背面都只在轮廓处出 rim。

# 双向菲涅尔（边缘 + 中心）

单层菲涅尔只有边缘光。加第二层「从中心出发」的辉光，两层**互补**做出果冻层次：

```
facing = abs(dot(sphereNormal, viewDir))
第一层（边缘）= pow(1 - facing, powerOuter)   // 轮廓处最亮
第二层（中心）= pow(facing, powerInner)        // 正对视线处最亮
```

关键：两层用**互补方向**（`1-facing` vs `facing`），亮区在空间上分开（边缘 vs 中心），不会互相淹没。

> 踩过的坑：一开始第二层也用 `1-facing`（只是不同 power），两层高度重叠，大 power 的宽辉光把锐边缘冲淡，看起来像「第一层也坏了」。**叠加两个同向效果 = 互相污染；要层次就用正交/互补的量。**

## 边缘硬度与边缘透明

* **硬度**：对第一层 fresnel 套 `smoothstep(0.5-edge, 0.5+edge, fresnel)`，`edge = 0.5*(1-sharpness)`。sharpness→1 时过渡带收窄成硬边描边；→0 退化为原始平滑渐变。
* **边缘透明**：`col.a *= 1 - fresnel * _FresnelAlpha`。边缘（fresnel≈1）更透、中心（fresnel≈0）保持底色 alpha。

# 深度相交 Rim（URP）

想让史莱姆与场景接触处发光。要点：

* **必须开 URP 深度纹理**：`m_RequireDepthTexture: 0→1`。注意 QualitySettings 每个质量档各引用一个 URP asset，**三个都要开**，否则切档失效。
* 采样 `_CameraDepthTexture`（`LinearEyeDepth(SAMPLE_DEPTH_TEXTURE_PROJ(...))`）vs 片元深度 `screenPos.w`，差值小于阈值处发光。

**关键认知（这是深度描边的天花板）**：URP 深度纹理只由**不透明**物体在透明阶段之前生成。史莱姆是半透明、不写深度 → 深度图里**根本没有史莱姆自己**。所以：
* 深度 Rim 只能做**相交发光**（史莱姆 vs 背后不透明场景），落地接触带发光。
* **做不出**沿史莱姆自身轮廓的描边——那需要它在被采样的深度图里，而它不在。想要贴合轮廓的描边，用菲涅尔（本就贴合视线轮廓），或单独把史莱姆渲染到一张 RT 再做屏幕空间 Sobel（RendererFeature，大工程）。

> 用 frame debugger 确认「史莱姆 mesh 没写入深度」是定位这个天花板的关键手段——不是 bug，是管线结构决定的。

# 精灵图集轮播

8×8 图集按帧轮播贴在表面：

```
frame = fmod(floor(_Time.y * fps), frameCount)
cell = 1/columns, 1/rows
uv = i.uv * cell + float2(col*cell.x, 1 - (row+1)*cell.y)  // 行从顶部往下：图集左上是第0帧，UV原点在左下
sprite = tex2D(_MainTex, uv); col.rgb = lerp(col.rgb, sprite.rgb, sprite.a)
```

**精灵只在正面 pass 出现**：否则半透明下正/背面各画一个，透过外壳同时看到两个镜像的精灵，orbit 时看似随视角漂移。用两个 fragment 入口区分（正面画、背面不画），共享核心函数传 `bool drawSprite`。

> 踩坑：pass 里的 `#define` 在 `CGINCLUDE` 之后，管不到已定义的 frag。想按 pass 区分行为，用**不同的 fragment 入口函数**（各自 `#pragma fragment fragFront/fragBack`），不要指望 `#define` + `#ifdef`。
>
> 注：此「精灵贴表面」方案最终被 [[Unity 程序化表情系统]] 的独立 billboard 组件取代——精灵贴 mesh 表面会随形变拉伸，billboard 更稳。

# 材质旧属性不序列化的坑（高频、隐蔽）

**现象**：给 shader 加了新属性，Inspector 里怎么调都不生效，甚至拖滑块也没用。

**根因**：材质是在加属性**之前**保存的。Unity 对已存在的 `.mat`，**不会自动把新加的 shader 属性写进它序列化的 `m_Floats`/`m_Colors` 表**。shader 读到未初始化值（0）→ 效果不出现或除零。

**定位手段（关键：拿数据而非猜）**：在 frag 里临时 `return half4(该参数, 该参数, 该参数, 1)` 直接可视化参数值。整片均匀且拖滑块不变 → 材质没存该属性。

**修法**：删掉重建材质，或直接在 `.mat` 的 `m_Floats`/`m_Colors` 里手写属性条目（按字母序）。

> 这个坑今天连踩两次（`_FresnelAlpha`、`_DepthRim*`）。规律：**加 shader 属性后，旧材质缺该属性值 → 表现为「新参数怎么调都没反应」，极易误判成 shader 逻辑错。**

#Renderer
