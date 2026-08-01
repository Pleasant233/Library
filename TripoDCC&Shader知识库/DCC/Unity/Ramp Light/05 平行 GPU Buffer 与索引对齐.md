# 05 平行 GPU Buffer 与索引对齐

> 承接 [[04 Runtime Bridge 与共享 Atlas]]。数据已经住在 Bridge 里，这一篇让它上 GPU：开一条与 URP Additional Light 同序的平行 Buffer。
> 关注点：**平行数组的索引合同**（本项目风险最高的一处）+ **读源码定合同的方法** + **两条关于断言和常量的判断**。
> 返回 [[Ramp Light 知识地图]]。

---

## 一、为什么这一步最容易出错

[[00 为什么要自定义 Ramp 灯光]] 选的方案是"平行 Buffer"：不改 URP 的 `_AdditionalLightsBuffer` 内存布局，另开一条**等长、同序**的 Buffer，用同一个 index 索引两边。

代价写在那篇里了：**必须保证两边索引严格对齐**。现在到了兑现这个代价的时候，而它的失效方式极其阴险：

```text
索引错位 1 位的表现：
  A 灯用了 B 灯的 Ramp 参数
  → 画面"不对"但不报错、不崩溃
  → 在纯 Point Light 场景下可能完全正常
  → 加一盏 Directional 灯才暴露
```

没有异常、没有日志、没有崩溃。**只有"颜色看着怪"**。所以这一阶段的功夫几乎全花在"把合同读准"上，而不是写代码上——真正新增的代码不到 80 行。

---

## 二、读源码定合同：四个必须查清的问题

动手前有四个问题必须从源码里得到确定答案，任何一个猜错，整个阶段都白做。

### Q1：URP 原生的 Structured Buffer 路径是开的还是关的？

```csharp
// RenderingUtils.cs
internal static bool useStructuredBuffer
{
    get
    {
        // TODO: For now disabling SSBO until figure out Vulkan binding issues.
        // When enabling this also enable USE_STRUCTURED_BUFFER_FOR_LIGHT_DATA in shader side in Input.hlsl
        return false;      // ← 硬编码
    }
}
```

```hlsl
// Input.hlsl
// Keep in sync with RenderingUtils.useStructuredBuffer
#define USE_STRUCTURED_BUFFER_FOR_LIGHT_DATA 0
```

**双侧固定关闭。** 灯光属性实际走 `_AdditionalLightsPosition` 等全局数组。

> [!warning] 这个发现改变了实现位置
> 如果按"计划里写着 `_AdditionalLightsBuffer`"就把 Ramp 挂到那条分支上，代码**永远不会执行**，而且不报错——你会对着一个静默的空实现调半天。
> 更别去开那个宏：Unity 自己的注释说这是因为 Vulkan 绑定问题未解决。开了等于替上游承担它还没解决的问题。
>
> **通用教训：计划里的 API 名不等于运行时真走的路径。** 涉及条件编译、平台分支、feature flag 的地方，必须查到那个 flag 的实际取值。

### Q2：CPU 侧的权威顺序是什么？

`ForwardLights.SetupAdditionalLightConstants()` 里两条分支（SSBO / 非 SSBO）都是同一个模式：

```csharp
int lightIter = 0;
for (int i = 0; i < lights.Length && lightIter < maxAdditionalLightsCount; ++i)
{
    if (mainLight != i)          // 跳过 Main Light
    {
        // ... 写入 m_AdditionalLight*[lightIter]
        lightIter++;
    }
}
```

`lightIter` 是**跳过 Main Light 后的紧凑编号**，和 `_AdditionalLightsPosition` 的下标严格相同。Main Light 恒为 Directional（`GetMainLightIndex()` 源码注释写着 `Main Light is always a directional light`，实现只返回最亮 Directional），所以排除项固定是一盏 Directional。

### Q3：Shader 侧直接用这个编号吗？

**不。** 这是最容易漏的一层：

```hlsl
// RealtimeLights.hlsl
#define LIGHT_LOOP_BEGIN(lightCount) { \
uint lightIndex; \
ClusterIterator _urp_internal_clusterIterator = ClusterInit(...); \
[loop] while (ClusterNext(_urp_internal_clusterIterator, lightIndex)) { \
    lightIndex += URP_FP_DIRECTIONAL_LIGHTS_COUNT; \    // ← 偏移
```

Cluster 只对 **local lights** 分 bin（C# 侧用 `firstLocalLightIdx` 跳过开头连续的 Directional 灯后 `GetSubArray`），所以迭代器吐出的是局部下标，**必须加回 Additional Directional 灯数**才能索引回同一批数组。

### Q4：Deferred+ 的取灯循环覆盖全集吗？

`ClusterDeferred.hlsl` 里有**两个**循环：

```hlsl
LIGHT_LOOP_BEGIN(pixelLightCount)                 // 覆盖 Local Light 区段
    Light light = GetAdditionalLight(lightIndex, inputData, ...);
LIGHT_LOOP_END

UNITY_LOOP for (uint lightIndex = 0; lightIndex < min(URP_FP_DIRECTIONAL_LIGHTS_COUNT, MAX_VISIBLE_LIGHTS); lightIndex++)
{                                                  // 覆盖 Additional Directional 区段
    Light light = GetAdditionalLight(lightIndex, inputData, ...);
```

两者共用同一个 `GetAdditionalLight()` 重载，**合起来才是 Additional Light 全集**。

---

## 三、合同的结论：必须写满每个槽位

Q1–Q4 合起来推出一条实现纪律：

> **Ramp Buffer 必须为全部 `lightIter` 槽位逐一写入记录**，包含 Additional Directional 灯的槽位。Ramp 只支持 Point Light，这些槽位写 `atlasRowIndex = -1` 占位。
> **严禁只为 Ramp 灯紧凑写入。**

```csharp
if (mainLight != i)
{
    // ... URP 原有的写入
    InitializeRampLightShaderData(lights[i].light, lightIter, ref hasVisibleRampLight);
    lightIter++;
}
```

```csharp
void InitializeRampLightShaderData(Light light, int additionalLightIndex, ref bool hasVisibleRampLight)
{
    if (RampLightRuntimeBridge.TryGetLightData(light, out RampLightRegisteredData registeredData))
    {
        m_RampAdditionalLightsData[additionalLightIndex] = new RampLightShaderData(registeredData);
        hasVisibleRampLight = true;
        return;
    }

    m_RampAdditionalLightsData[additionalLightIndex] = RampLightShaderData.Disabled;   // ← 占位，不跳号
}
```

> [!tip] 平行数组的黄金律：宁可写占位，绝不跳号
> 紧凑写入（只写"有数据"的项）是个非常自然的直觉——它省内存、看起来更"干净"。但它**破坏了 index 的语义**：紧凑数组的第 3 项不再对应原数组的第 3 项。
> 一旦跳号，后续所有项整体偏移，且**没有任何机制会告诉你**。
> 判据：**如果两个数组要用同一个 index 索引，它们的长度和空位必须完全一致。** 这条在 ECS、GPU 粒子、骨骼动画里同样成立。

`InitializeRampLightShaderData` 放在 `lightIter++` **之前**，用的就是那个权威变量——不另算索引，是这里唯一安全的写法。

---

## 四、失误一：断言了一个不成立的等式

第一版加了这么一条：

```csharp
// ❌ 已删除
Assertions.Assert.AreEqual(additionalLightsCount, rampLightDataCount,
    "Ramp Light data count must match the URP additional light data count.");
```

意图是好的：想用断言守护索引对齐。但这两个量的定义不同：

| 量 | 来源 | 含义 |
| --- | --- | --- |
| `additionalLightsCount` | `SetupPerObjectLightIndices()`，Forward+ 下直接返回 `lightData.additionalLightsCount` = `Math.Min(visibleLights.Length - (hasMain?1:0), maxVisibleAdditionalLights)` | **预估上限**，只按数量截断 |
| `rampLightDataCount` | 循环结束后的 `lightIter` | **实际写入数** |

正常场景两者相等，但这是**巧合而非契约**。URP 自己从不假设它们相等——它用 `lightIter` 索引数组、用 `additionalLightsCount` 做容量决策。把两者钉成必须相等，等于给 URP 加了一条它自己没承诺的约束；上游定义稍变（比如开始扣除不支持的灯类型），断言就会在正常渲染中报错。

> [!warning] 断言要守"不变量"，不要守"当前恰好成立的巧合"
> 判据：**这个等式是被谁保证的？** 如果答案是"我观察到目前是这样"，那它不是不变量。
> 附带一个 Unity 特性：`Assertions.Assert` 受 `UNITY_ASSERTIONS` 条件编译控制（Editor 与 Development Build 开，Release 关）。所以一条错误的断言是**开发期噪音 + 发布期无保护**，两头不落好。
>
> 真正该守的是会导致越界的那个条件：`rampLightDataCount <= m_RampAdditionalLightsData.Length`。而这里它已经由循环条件 `lightIter < maxAdditionalLightsCount` 与数组按 `maxVisibleAdditionalLights` 分配的一致性隐式保证了。

---

## 五、失误二：手写 stride 常量

```csharp
// ❌ 已删除
internal const int Stride = sizeof(float) * 8;
```

而实际创建 Buffer 时走的是另一条路：

```csharp
buffer = new ComputeBuffer(size, Marshal.SizeOf<T>());     // ShaderData.GetOrUpdateBuffer<T>
```

**同一个事实有了两份定义。** 往 struct 里加字段而忘了改 `Stride`，两者就分叉——而且是静默分叉，只有测试会发现（前提是测试能跑，见第七节）。

修法是删掉常量，让 `Marshal.SizeOf<T>()` 成为唯一来源，测试直接断言布局：

```csharp
int actualStride = Marshal.SizeOf<RampLightShaderData>();
Assert.That(actualStride, Is.EqualTo(sizeof(float) * 8));
```

> [!tip] 同一事实只留一份可执行定义
> "结构体多大"这件事，`Marshal.SizeOf<T>()` 已经**从类型本身算出来**了。再手写一个常量，就是把一个可推导的事实变成了需要人工维护的副本。
> 这和 [[02 Gradient 采样与颜色空间]] 里"采样器收成唯一实现"、[[01 附加组件与单一数据源]] 里"删掉 Atlas 双数据源"是同一条原则在不同层的体现：**单一数据源，从数据到算法到常量。**
> 例外是需要跨语言同步的常量（比如阶段 4 的 HLSL struct），那里没法共享定义，只能靠纪律 + 测试。

---

## 六、GPU 数据布局：32 bytes 与预留位

```csharp
[StructLayout(LayoutKind.Sequential)]
internal readonly struct RampLightShaderData
{
    internal static readonly RampLightShaderData Disabled = new RampLightShaderData(
        new Vector4(0.0f, 1.0f, 0.0f, 1.0f),      // 参数取安全默认
        new Vector4(-1.0f, 0.0f, 0.0f, 0.0f));    // 行号 -1 = 禁用

    internal readonly Vector4 Parameters;      // distanceStart, distanceEnd, normalInfluence, rampIntensity
    internal readonly Vector4 AtlasMetadata;   // atlasRowIndex, 0, 0, 0（三个分量预留）
}
```

几个决定：

- **`[StructLayout(LayoutKind.Sequential)]`** 保证字段顺序不被 CLR 重排——GPU 侧按偏移读，顺序变了就全错。
- **行号用 float 存整数**：float 可精确表示 2²⁴ 内的整数，256 行量级完全安全。Shader 侧按 `< 0` 判禁用，不依赖精确相等。
- **预留三个分量**：后续参数可以填进来而不改 stride。但预留是双刃的——**stride 一旦定下，C# / HLSL / 布局测试三处必须同步**，缺一即静默错位。

`Disabled` 的参数部分故意填成合法值（`0, 1, 0, 1`）而不是全 0：万一 Shader 侧漏判 `rowIndex < 0`，用这组值算出来的也是"无变化"的结果，而不是除零或者黑块。**兜底值本身也该是 Feature-off 等价的**，和 [[01 附加组件与单一数据源]] 里默认白 Ramp 同理。

---

## 七、Buffer 生命周期与零灯情形

```csharp
internal ComputeBuffer GetRampAdditionalLightsBuffer(int size)
{
    if (size < 1)
        throw new ArgumentOutOfRangeException(nameof(size));

    int bufferSize = Mathf.NextPowerOfTwo(size);
    return GetOrUpdateBuffer<RampLightShaderData>(ref m_RampAdditionalLightsBuffer, bufferSize);
}
```

`GetOrUpdateBuffer` 只在 `size > buffer.count` 时重建，所以**不扩容就复用**；`NextPowerOfTwo` 摊销扩容次数（和 [[04 Runtime Bridge 与共享 Atlas]] 的 Atlas 高度同一手法）。`ShaderData.Dispose()` 里挂了释放。

零灯时的处理值得单独看：

```csharp
int uploadCount = Math.Max(1, rampLightDataCount);
if (rampLightDataCount == 0)
    m_RampAdditionalLightsData[0] = RampLightShaderData.Disabled;
```

**永远绑定合法资源，元素数最少 1。** 加上 Bridge 侧的 1×1 白色后备纹理，Shader 侧任何时候都能安全采样，不需要判"资源是否存在"。

> [!note] `_RampLightActive` 必须是逐相机语义
> 它取自 `hasVisibleRampLight`——本次 `SetupAdditionalLightConstants` 循环里是否遇到过有效 Ramp 灯，而**不是** Bridge 的全局 `Entries.Count`。
> Registry 是全局的，可见灯是**逐相机**的。拿前者当后者，在多相机场景直接错。这类"作用域错配"是渲染代码里的常见 bug 源：**问自己"这个量的作用域是全局、逐相机、还是逐物体？"**

---

## 八、这一阶段没能验证的部分

诚实记录，因为它决定了下一步该先做什么：

**索引对齐没有自动化测试覆盖。** 五个新测试覆盖的是布局、禁用值、Buffer 复用/扩容、零尺寸抛异常——都是**能单元测试的部分**。而风险最高的索引对齐依赖 `Light` 与 `CullingResults`，只能靠 Frame Debugger 实测。

> [!warning] 验收必须用「非紧凑」灯光场景
> 全是 Ramp Point Light 的场景**无法暴露跳号错位**——因为没有需要占位的槽位。
> 必须构造：Main Directional + 1 盏 Additional Directional + 交错排列的 Ramp Point 与普通 Point 灯，逐槽位核对。
> **测试场景要包含"会让错误实现通过"的那种情况**，否则绿色是假的。

**测试的可运行性仍未确认。** [[04 Runtime Bridge 与共享 Atlas]] 提的 `testables` 已经加进 `Packages/manifest.json`，但这一轮的 `8/8 passed` 依然是靠一次性 Editor 反射入口跑的（因为活动场景 Dirty，不便切场景）。也就是说 `testables` **是否真的让 Test Runner 发现了这些测试，还没被验证过**。

---

## 九、下一步

到这里 GPU 已经拿到了数据：Buffer、Atlas、TexelSize、Active 四个全局量每相机都绑定了。但**没有任何 HLSL 读取它们**，画面依旧零变化。

阶段 4 才是第一次真正改光照：在 `GetAdditionalLight()` 返回前用 `rampU` 采 Atlas 并调制 `light.color`。那时前面五篇建立的所有约定——端点包含的采样、纵向锁行中心、索引合同、`-1` 判禁用——会一次性全部兑现。

## 速记

- 平行数组的黄金律：**宁可写占位，绝不跳号**。同 index 索引的两个数组，长度和空位必须完全一致。
- 索引错位不报错、不崩溃，只是"颜色怪"，且**可能在简单场景下完全正常**——所以功夫全在读准合同。
- **计划里的 API 名 ≠ 运行时真走的路径**。条件编译 / feature flag 必须查实际取值（`useStructuredBuffer` 硬编码 `false`）。
- Cluster 路径的 Shader 索引 = 迭代器输出 **+ `URP_FP_DIRECTIONAL_LIGHTS_COUNT`**；Deferred+ 用两个互补循环覆盖全集。
- **断言要守不变量，不守"当前恰好成立"**。判据：这个等式被谁保证？`Assertions.Assert` 还受 `UNITY_ASSERTIONS` 控制，错的断言 = 开发期噪音 + 发布期无保护。
- **同一事实只留一份可执行定义**：删掉手写 stride，统一 `Marshal.SizeOf<T>()`。
- `[StructLayout(LayoutKind.Sequential)]` 保证字段不被重排；float 存整数在 2²⁴ 内精确。
- **兜底值本身要 Feature-off 等价**（`Disabled` 的参数填 `0,1,0,1` 而非全 0）。
- 永远绑定合法资源（元素数 ≥ 1 + 1×1 后备纹理），Shader 侧无需判存在性。
- 追问每个量的**作用域**：`_RampLightActive` 是逐相机的，不能用全局 Registry 数量代替。
- **测试场景要包含"会让错误实现通过"的情况**：全 Ramp 灯场景验不出跳号错位。

#DCC #Unity #RampLight
