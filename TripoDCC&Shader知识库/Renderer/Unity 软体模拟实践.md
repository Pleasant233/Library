# 前置

> 个人自学项目，对应 [[任务需求]] 7.16 个人项的「软体模拟」。
> 目标：在 Unity 里做一个史莱姆软体，落地能压扁摊开、有弹性、可被推动。方案是**质点弹簧 + shape matching + 体积投影**的混合模型，CPU 求解，后续扩展了 GPU compute 后端。
> 与 [[Unity 光线追踪实践]] 同属 Renderer 自学线。

* 本地工程：`/Users/vast/DccDev/Unity Project/Graphics Learning`
* 代码：`Assets/Scripts/SoftBody/`（`SlimeTopology` 几何 / `CpuSlimeSolver` 物理 / `SoftBodySimulation` 驱动 / `SoftBodyDebugVisualizer` 可视化）
* GPU 后端：`Assets/Resources/SoftBody/SlimeSolver.compute` + `GpuSlimeSolver.cs`

# 架构：几何与物理分离

三层职责，边界清晰（这也是能加 GPU 后端的前提）：

* **SlimeTopology**（纯几何）：质点布局、弹簧拓扑、mesh 顶点绑定。**不含任何积分/碰撞代码**，所以 GPU 求解器能复用同一套拓扑。
* **CpuSlimeSolver / GpuSlimeSolver**（物理）：每帧 `Simulate(dt, substeps, params)` 做积分、约束、碰撞。
* **SoftBodySimulation**（MonoBehaviour）：驱动求解、把质点位移映射回 mesh 顶点。
* **ISoftBodySolver** 接口把 MonoBehaviour 和具体后端解耦，`Auto` 模式自动选 GPU、失败回退 CPU。

# 质点布局：结构化径向点阵（从随机撒点重构而来）

最初用 Fibonacci 随机撒表面点 + 随机相位内层壳，内部质点靠求解器**事后钳制**塞回外壳。这套「后处理补丁」问题不断，最终**从根源重构**为参考实现的**径向分层点阵**：

* `directionCount` 条 Fibonacci 球面方向 × `shellCount` 层
* 每条方向从表面往内放点，**内层点是表面点沿同方向的缩放副本**（`radiusFraction < 1`）
* `DistributionFactor > 1` 让内层点向表面聚集（质量集中在皮上），贴合史莱姆手感
* 弹簧：径向辐条（同方向相邻壳）+ 环向切线（同壳相邻方向）+ 中心 hub，规则网格

**关键收益**：内层点在结构上被径向弹簧拴在对应表面点内侧，**横向溢出从拓扑上被杜绝**，不再需要事后径向钳制。

> 教训：能用**拓扑/数据结构**从根源保证的性质，就不要用**每帧后处理**去补。后处理补丁不符合代码规范，且会掩盖真正的问题（见下面「穿地」一节——被补丁掩盖了很久）。

# 调试踩坑（软体物理最花时间的部分）

## 溢出不是一个问题，是三个不同的问题

「内部质点溢出 mesh」这个现象我误判过多次，实际是三个独立根因，**必须对症**：

* **横向溢出**（压扁成盘时内部点从侧面漏出）→ 拓扑问题。径向点阵重构解决：内层是表面点的径向缩放副本，结构上出不去。
* **垂直穿地**（内部点被重力拽穿地面成锥形，把整体拽塌）→ **碰撞问题，不是拓扑问题**。根因：地面碰撞只作用于 surface 质点，内部质点没有任何地面约束。修法：地面碰撞作用于全体质点（O(n)，廉价；O(n²) 的自碰撞仍只在 surface 间）。
* **softness 高时内部溢出** → 速度响应问题（见下）。

> 最大的教训：**「内部质点穿地」我一开始当拓扑问题查，绕了很久。它其实是碰撞问题**——内部质点压根没参与地面碰撞。现象相似的 bug 可能根因完全不同，别被「看起来像上次那个」带偏。

## 落地要「停住」，光钳位置不够，必须处理速度

需求：内部质点接触地面就停住（Y 不再降），只横向摊开。
一开始只把内部质点**位置**钳到地面，但**没记接触法线** → `UpdateVelocities` 不给它做地面速度响应 → 重力每帧继续给向下速度，位置又被钳回 → 贴地抖动 + 拽 surface 下沉。
修法：内部质点落地时**也 `AddContact`**，和 surface 走完全相同的速度响应（消除向下速度分量）。区别仅在钳制高度（surface 留球半径间隙，内部直接到地面）。

> 教训：位置约束（PBD 式钳位）和速度必须一致。只改位置不改速度，下一帧速度会把位置「拽回去」，表现为抖动。

## 体积投影引入的净平移与振荡

径向点阵的每层复用同一组方向，Fibonacci 方向和有微小残差，**层间相干累加**，使「全体质心 ≠ 表面质心」。体积投影以表面质心为参考放大偏移，对全体求和不为零 → **一股净水平漂移**。

* 修法一：体积投影加 **COM 守恒**——投影后减去平均位移，只改形状不引入平移。
* 修法二（关键订正）：COM 守恒和体积投影都**只作用于 surface 质点**——因为体积是由 surface 度量的，作用集合必须 = 度量集合。一开始在「全体质点」上减平均，破坏了径向层次导致内部点被推出、且落地振荡（度量与作用集合不一致 → 每帧 scale 忽大忽小）。

> 教训：位置型体积约束不该给刚体引入净平移（要 COM 守恒）；且**约束的作用集合必须和它的度量集合一致**，否则会引入你意想不到的耦合。

# 渲染：球形法线 + 质心球心

软体形变让 mesh 顶点法线变脏，菲涅尔边缘光斑驳。改用**几何球形法线**：`normalize(worldPos - center)`，从中心指向表面点的径向方向，无视形变始终干净。

* 球心不能用 `unity_ObjectToWorld` 原点——物理质点在世界空间独立演化，`transform` 并不跟随质心移动，会脱节。
* 也**不要**去改 `UpdateMesh` 让 transform 跟随质心（试过，导致穿模——破坏了顶点写回的坐标一致性，已回退）。
* 正解：C# 每帧用 `MaterialPropertyBlock` 把真实质心 `_focusPoint` 作为 `_Center` 传给 shader 当球心。不碰顶点逻辑、不实例化材质、无 GC。

> 教训：物理表示（世界空间质点）和 Transform 表示脱节时，优先**把物理量传给需要它的地方**（shader 参数），而不是强行让 Transform 去追物理——后者容易破坏依赖 Transform 的其它逻辑（顶点写回、碰撞查询）。

# GPU 后端要点（compute 求解器 review 结论）

把求解器搬到 compute shader 时，正确处理并行竞态是核心：

* **spring 用 per-particle gather**：每线程读自己的邻接表、只写自己 → 天然无竞态，不需要 atomic。别用 scatter（多线程写同一质点）。
* **自碰撞 gather-into-scratch**：读全体、只写自己的 scratch，再统一拷回。
* **速度双缓冲 ping-pong**：`_VelocitiesRead` / `_VelocitiesWrite` 分离，避免读写同一 buffer 竞态。
* **每个约束 pass 独立 Dispatch**：dispatch 之间有隐式同步，替代 pass 内 barrier。
* **weld（合并共享质点）跨线程写共享 buffer** → 仅当 weld 组严格互斥才安全，这个不变量要在 C# 侧强制。
* **资源生命周期**：18 个 ComputeBuffer 必须全在 `Dispose` 释放；构造函数里 buffer 分配和 `FindKernel` 的顺序要小心——FindKernel 抛异常会让已分配 buffer 泄漏（对象没构造完，调用方 Dispose 不到）。
* **回读代价**：每帧 `GetData` 是同步阻塞回读，冲掉 GPU 流水线；mesh 更新需要 CPU 侧位置时难免，可考虑 `AsyncGPUReadback` 延迟一帧。

# 后续

* [ ] GPU 后端的 3 个 major（构造函数资源泄漏、surface 索引假设 vs `_IsSurface` flag、weld 组互斥不变量）修复
* [ ] CPU / GPU 约束顺序 side-by-side 对齐，避免 Auto 切后端手感跳变
* [ ] 更真实的碰撞（任意 collider、软体间碰撞）
* [ ] 自碰撞 O(n²) 的空间哈希优化

#Renderer
