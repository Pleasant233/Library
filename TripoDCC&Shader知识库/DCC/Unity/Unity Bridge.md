# Tripo3D Unity Bridge — 项目学习笔记

## 项目定位

Unity Editor 插件包（UPM Package），在 Editor 内运行 WebSocket 服务器，接收来自 Tripo3D 网页前端推送的 3D 模型文件，自动完成解包、导入、材质配置，并将模型实例化到当前场景。

## 技术栈

| 层次 | 技术 |
|---|---|
| 宿主环境 | Unity Editor 2021.3+，.NET Standard 2.1 |
| 语言 | C# |
| 网络通信 | WebSocket（`websocket-sharp.dll` 第三方库） |
| 序列化 | `UnityEngine.JsonUtility` |
| 文件处理 | `System.IO.Compression.ZipFile`（内置） |
| 依赖管理 | Unity Package Manager（UPM） |
| CI/CD | GitHub Actions + AWS S3 |
| AI 自动化 | GitHub Actions 调用 Claude API 处理 Issue/PR |

## 目录结构

```
/
├── Tripo3d_Unity_Bridge/        # UPM 包本体
│   ├── package.json             # 包元数据，version: 1.0.0，最低 Unity 2021.3
│   └── Editor/
│       ├── *.cs                 # 所有 C# 模块
│       ├── com.tripo3d.unitybridge.editor.asmdef
│       ├── Plugins/
│       │   └── websocket-sharp.dll   # WebSocket 第三方库
│       └── Resources/
│           └── TripoLogo.png
├── .github/
│   ├── workflows/
│   │   ├── ci.yml               # Issue/PR/评论 AI 自动化流水线
│   │   ├── release.yml          # 打包上传 S3 + GitHub Release
│   │   ├── pr-review.yml        # AI 自动 PR 审查
│   │   ├── implement.yml        # AI 自动实现 Issue
│   │   ├── issue-dispatch.yml   # Issue 类型路由
│   │   └── question.yml         # Issue 问题自动回答
│   └── actions/feishu-notify/   # 飞书通知 Action
├── pack.py                      # 本地打包脚本
├── README.md
└── CLAUDE.md                    # AI Agent 工作规范
```

## 主要模块

### TripoWebSocketWindow.cs
EditorWindow 入口，挂在 `Tools > Tripo Bridge` 菜单。负责 UI 渲染（Logo、服务器开关、状态、进度条、日志面板），将服务器事件桥接到主线程刷新 UI。

### WebSocketSharpServer.cs
WebSocket 服务器封装，监听 `ws://127.0.0.1:60610`。
- `WebSocketSharpServer`：管理 `websocket-sharp` 的 `WebSocketServer` 实例
- `TripoWebSocketBehavior`：继承 `WebSocketBehavior`，处理连接/断开/消息，解析二进制帧（JSON header + binary chunk），调度文件块接收和模型导入

### FileTransferManager.cs
管理分片文件传输会话。`FileTransferSession` 按 chunkIndex 存储每个 5MB 分片，全部到齐后 `Assemble()` 拼装完整文件。

### ModelImporter.cs
核心模型处理逻辑，完整流程：
1. 判断 ZIP / 直接 FBX|OBJ
2. ZIP 解压 → 查找模型文件
3. 拷贝到 `Assets/TripoModels/{uniqueName}/`
4. 预处理：修复 Normal Map 类型 + 合并 Metallic/Roughness → Unity 格式（R=Metallic, A=Smoothness）
5. `AssetDatabase.ImportAsset` 导入，配置材质导入模式
6. 按渲染管线（Standard/URP/HDRP）选 shader 和贴图属性名，绑定 BaseColor/MetallicSmoothness/Normal 贴图到材质
7. 实例化到当前场景 `Vector3.zero`

### ProtocolConstants.cs
协议常量：端口 60610、消息类型字符串、5MB 分片大小、心跳参数、错误码、支持格式（.fbx / .obj / .zip）。

### MessageTypes.cs
WebSocket 消息的可序列化数据结构：handshake、handshake_ack、file_transfer、file_transfer_ack、file_transfer_complete、import_complete、ping/pong。

### Localization.cs
UI 多语言支持，根据 `Application.systemLanguage` 自动切换，支持：英、简中、繁中、日、韩、俄、法、德、西、葡共 10 种语言。

### LogHelper.cs
静态日志事件总线，带时间戳格式化，统一分发到 UI 面板。

### StartupCleanup.cs
`[InitializeOnLoad]` 启动清理，每 24 小时一次：删除旧模型文件夹（7 天以上）和 temp 目录中 GUID 格式的遗留文件夹。

## 端到端工作流程

### 阶段 0：服务器启动

打开 `Tools > Tripo Bridge` 菜单 → `TripoWebSocketWindow.OnEnable()` 做三件事：

1. 加载 Logo
2. `ModelImporter.DetectAndSetRenderPipeline()` 自动探测渲染管线（Standard / URP / HDRP）
3. **自动启动**服务器，监听 `ws://127.0.0.1:60610`

> [!note] 关闭窗口会自动 `Stop()`（`OnDisable`）。调试时窗口需保持打开。

### 阶段 1：握手

网页连上后，服务端 `OnOpen()` **立即主动**回发 `handshake_ack`，携带 Unity 版本、插件版本。注意它不等客户端的 `handshake` 消息——一连上就发 ack。客户端后续发的 `handshake` 只会打一条日志。

### 阶段 2：心跳

`ping` / `pong`，文本和二进制两种格式都支持。

### 阶段 3：分片文件传输（核心）

每个分片是一个二进制帧，格式为 `{JSON header}<二进制数据>`（**直接拼接，无长度前缀**）：

1. `FindJsonEnd()` 用**花括号配对**找到 JSON 结束位置，拆出 header 和 chunk 数据
2. 第一个分片（`chunkIndex == 0`）触发 `OnFileTransferStart`，更新 UI 文件名
3. `FileTransferManager.AddChunk()` 按 `fileId` 建 session，用 `Dictionary<int, byte[]>` 存分片
4. 每收一片回一个 `file_transfer_ack`
5. `IsComplete()` 判断是否收齐（`_chunks.Count == TotalChunks`）

### 阶段 4：组装 + 导入

收齐后：

1. 发 `file_transfer_complete`（status = importing）
2. `AssembleFile()` 按索引顺序拼接，**立即 `RemoveSession()`** 防止重复导入
3. `EditorApplication.delayCall` 把导入切回主线程（Unity API 只能主线程调用）

导入流程 `ModelImporter.ImportModel()`：

```
判断 ZIP / 直接文件
  → 解压 / 落盘到 temp (Application.temporaryCachePath/{fileId})
  → 找模型文件 (FindModelFile: 先找 .fbx 再找 .obj)
  → 生成唯一名 (GetCleanModelName → GetUniqueModelName)
  → 重命名模型文件 + .fbm 贴图文件夹
  → 拷贝到 Assets/TripoModels/{uniqueName}/
  → FixNormalMaps (含 normal 的贴图设为 NormalMap 类型)
  → CombineMetallicRoughnessTextures (合并成 Unity 的 R=Metallic / A=Smoothness)
  → AssetDatabase.ImportAsset + 配置 materialImportMode
  → ApplyTexturesToMaterials (按渲染管线选 shader / 属性名，绑定贴图)
  → AddToScene (实例化到 Vector3.zero，重命名，选中)
```

### 阶段 5：完成通知

发 `import_complete`（success + message）。全程通过 `OnProgressUpdate` 上报进度（0 → 0.5 → 0.7 → 0.9 → 1.0）。

## 网页端传输内容

每个分片的 JSON header（`FileTransferPayload`）字段：

| 字段 | 类型 | 含义 |
|---|---|---|
| `fileId` | string | 传输会话唯一 ID，作为 session key 和 temp 文件夹名 |
| `fileName` | string | 原始文件名（含扩展名） |
| `fileType` | string | 文件类型（`fbx` / `obj` / `zip`） |
| `chunkIndex` | int | 当前分片序号（从 0 开始） |
| `chunkTotal` | int | 总分片数 |
| `chunkSize` | int | 当前分片字节数 |

**加上帧尾的二进制数据本身**（最多 5MB / 片）。即网页端传的是：一段 JSON 元信息 + 一个模型文件的字节流（FBX / OBJ / ZIP），分片发送。ZIP 内可包含模型 + 贴图 + `.mtl`。

## fileName 参数的来源与流转

**来源：完全由网页端提供**，在每个分片的 JSON header 里传入。插件端从不生成它，只消费和加工。完整流转链：

1. **入口** — `ProcessBinaryMessage` 中 `JsonUtility.FromJson<FileTransferMessage>()` 解析出 `payload.fileName`
2. **存入 session** — `FileTransferManager.AddChunk()` 存进 `FileTransferSession.FileName`
3. **传给导入器** — `ProcessReceivedFile(fileId, fileName, fileType, ...)`
4. **加工成模型名**（`ModelImporter` 关键两步）：
   - `GetCleanModelName()`：剥掉 `.zip / .fbx / .obj / .glb / .gltf` 扩展名，**空格换成下划线**，空名兜底为 `"model"`
   - `GetUniqueModelName()`：若 `Assets/TripoModels/{name}` 已存在，追加 `_1`、`_2`… 保证唯一
5. **决定三处命名**（都用加工后的 `uniqueModelName`）：
   - Assets 子文件夹名：`Assets/TripoModels/{uniqueModelName}/`
   - 模型文件本身重命名 + 关联的 `.fbm` 贴图文件夹
   - 场景里 GameObject 的名字（`AddToScene`）
6. **UI 显示** — `OnFileTransferStarted` 存进 `_currentFileName`，状态栏显示（去扩展名）

## 调试方式

### 本地包 + 实时编译（改代码时用）

在测试 Unity 工程的 `Packages/manifest.json` 中以 `file:` 协议引用插件源码目录：

```json
"com.tripo3d.unitybridge": "file:/path/to/Tripo3d_Unity_Bridge"
```

插件源码与 Unity 工程完全分离，改 `.cs` 保存后 Unity 自动重新编译，无需重装包。

### Python 脚本模拟前端（测协议 / 导入逻辑）

用 `test_client.py`（仓库根目录）模拟完整前端行为，无需打开浏览器：

```bash
pip install websockets
python test_client.py                    # 仅测试握手
python test_client.py path/to/model.fbx  # 发送模型文件
python test_client.py path/to/model.zip  # 发送 ZIP 包
```

> [!important] 二进制帧格式为 `{JSON header}<二进制数据>` 直接拼接，**无 4 字节长度前缀**。插件端 `FindJsonEnd()` 靠花括号配对定位 JSON 结束位置，脚本必须与之一致。

### Unity Console 日志（运行时观察）

`Window > General > Console` 查看实时日志；代码里用 `LogHelper.Log(...)` 输出，会同时分发到插件 UI 面板。开启 **Error Pause** 可在异常时自动暂停。

## 已知问题

| 问题             | 位置                                | 说明                                                                                                       |
| -------------- | --------------------------------- | -------------------------------------------------------------------------------------------------------- |
| 清理路径不一致        | `StartupCleanup.cs`               | 清理的是 `Assets/ImportedModels/`，实际导入目录是 `Assets/TripoModels/`，清理逻辑不生效                                      |
| 材质重命名注释掉       | `ModelImporter.cs`                | 跨 Unity 版本会产生重复 .mat 文件，暂未决策                                                                             |
| 直接传 FBX 可能双扩展名 | `ModelImporter.ProcessDirectFile` | 拼成 `{fileName}{extension}`，若网页端 `fileName` 已含扩展名会生成 `model.fbx.fbx`；ZIP 路径不受影响（从解压结果 `FindModelFile` 查找） |
