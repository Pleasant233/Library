# 10 前端接 WebSocket

> 讲**网页侧怎么和 Unity 插件通信**,直接对应你在做的 DCC Bridge。
> 插件端(C#)的实现细节见 [[Unity Bridge]],这篇讲**前端要怎么配合**。

## 一、先分清:HTTP 请求 vs WebSocket

前端和外部通信主要两种模式:

| | HTTP(fetch) | WebSocket |
| --- | --- | --- |
| 模式 | 请求-响应,一问一答 | **全双工长连接**,双方随时互发 |
| 谁发起 | 只能客户端发起 | 建连后双方都能主动推 |
| 适合 | 拉取数据、提交表单 | 实时通信、推送、**大文件流式传输** |
| 类比 | 寄一封信等回信 | 打电话,通话中随时说话 |

**Bridge 为什么用 WebSocket**:网页要把模型**分片流式**推给 Unity,Unity 也要实时回 ack / 进度(见 [[Unity Bridge]] 的握手、心跳、分片 ack)——这是双向实时,HTTP 的一问一答不合适。

## 二、fetch:最基础的 HTTP 请求

先掌握这个,日常调后端接口全靠它:

```ts
// GET
const res = await fetch('https://api.example.com/models')
const data = await res.json()   // 解析 JSON 响应

// POST
const res2 = await fetch('/api/upload', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'cube' }),
})
```
- `fetch` 返回 **Promise**(见 [[02 JavaScript核心]] 事件循环),用 `await` 等结果。
- **取消请求**用 `AbortController`(呼应 [[08 Web3D与AI]] AI 聊天「停止生成」):
  ```ts
  const ctrl = new AbortController()
  fetch(url, { signal: ctrl.signal })
  ctrl.abort()   // 取消
  ```
- 跨域会遇到 **CORS**,那是服务器要配的,前端只是配合(见 [[04 浏览器原理]] 安全模型)。

## 三、WebSocket 前端 API(核心)

浏览器内建 `WebSocket`,4 个事件就够用:

```ts
const ws = new WebSocket('ws://127.0.0.1:60610')   // 对应 Bridge 的端口
ws.binaryType = 'arraybuffer'   // 收发二进制用 ArrayBuffer(传模型必须)

ws.onopen = () => {
  console.log('连上了')          // 对应 Bridge 阶段1:连上后插件立即回 handshake_ack
}
ws.onmessage = (e) => {
  // e.data 可能是字符串(JSON)或 ArrayBuffer(二进制)
  if (typeof e.data === 'string') {
    const msg = JSON.parse(e.data)   // 如 file_transfer_ack / import_complete
  }
}
ws.onerror = (e) => console.error('出错', e)
ws.onclose = () => console.log('断开')

// 发送:字符串或二进制都行
ws.send(JSON.stringify({ type: 'ping' }))
ws.send(arrayBuffer)   // 发二进制帧
```

**对应 [[Unity Bridge]] 的消息类型**(网页侧要收发这些):
- 连上 → 收到 `handshake_ack`(带 Unity 版本、插件版本)。
- 心跳 → 互发 `ping` / `pong`。
- 传文件 → 逐片发二进制帧,每片收一个 `file_transfer_ack`。
- 收齐 → 收到 `file_transfer_complete`(importing)→ 最后 `import_complete`。

## 四、重点:分片二进制帧怎么拼(前端侧)

这是 Bridge 最关键、也最容易踩坑的地方。插件端约定的帧格式是:

```
{JSON header}<二进制数据>      ← 直接拼接,无长度前缀!
```
插件靠**花括号配对**(`FindJsonEnd`)找 JSON 结束位置(见 [[Unity Bridge]] 阶段3),所以**前端拼帧必须和它完全一致**,否则解析错位。

前端构造一个分片帧:
```ts
function buildChunkFrame(header: object, chunk: ArrayBuffer): ArrayBuffer {
  const headerStr = JSON.stringify(header)          // 如 {fileId, fileName, chunkIndex,...}
  const headerBytes = new TextEncoder().encode(headerStr)   // 字符串→UTF-8 字节
  // 拼接:header 字节 + chunk 字节,中间无分隔符、无长度前缀
  const frame = new Uint8Array(headerBytes.length + chunk.byteLength)
  frame.set(headerBytes, 0)
  frame.set(new Uint8Array(chunk), headerBytes.length)
  return frame.buffer
}
```

分片发送整个文件(5MB/片,对应 [[Unity Bridge]] 的 `ProtocolConstants`):
```ts
const CHUNK = 5 * 1024 * 1024
async function sendFile(ws: WebSocket, file: ArrayBuffer, meta: { fileId: string; fileName: string; fileType: string }) {
  const total = Math.ceil(file.byteLength / CHUNK)
  for (let i = 0; i < total; i++) {
    const slice = file.slice(i * CHUNK, (i + 1) * CHUNK)
    const header = {
      fileId: meta.fileId,
      fileName: meta.fileName,      // 注意:含扩展名,插件靠它命名(见 Unity Bridge 流转链)
      fileType: meta.fileType,      // 'fbx' / 'obj' / 'zip'
      chunkIndex: i,                // 从 0 开始,第 0 片触发插件更新 UI 文件名
      chunkTotal: total,
      chunkSize: slice.byteLength,
    }
    ws.send(buildChunkFrame(header, slice))
    // 生产中应等对应的 file_transfer_ack 再发下一片,做背压控制(见下)
  }
}
```

> [!warning] 三个和 [[Unity Bridge]] 已知问题呼应的坑:
> - `fileName` **不要重复带扩展名**:插件 `ProcessDirectFile` 会拼 `{fileName}{ext}`,直传 FBX 时可能生成 `model.fbx.fbx`。
> - header 里**别用花括号字符串值**(如文件名含 `{}`),会干扰插件的花括号配对。
> - 字段名、大小写要和 `FileTransferPayload` 完全一致,`JsonUtility` 对不上就丢字段。

## 五、必须处理的健壮性问题

WebSocket 是长连接,前端要处理这些(面试也常问):

- **背压 / 流控**:别一次性 `send` 几百片,`ws.bufferedAmount` 会堆爆内存。等 `file_transfer_ack` 再发下一片,或监控 `bufferedAmount` 到阈值就暂停。
- **断线重连**:`onclose` 里用**指数退避**重连(1s、2s、4s…),别死循环猛连。
- **心跳保活**:定时发 `ping`,一段时间没 `pong` 就判定断线(对应 Bridge 阶段2)。
- **连接状态机**:维护 `连接中/已连接/传输中/已断开` 状态,驱动 UI(呼应 [[08 Web3D与AI]] 的消息状态机思路)。
- **二进制类型**:忘了设 `ws.binaryType = 'arraybuffer'`,收到的会是 `Blob`,处理方式不同。

## 六、调试

- **浏览器 DevTools → Network → WS**:能看每一帧的收发内容和时间线,排协议问题第一站。
- 插件侧可用 `test_client.py`(见 [[Unity Bridge]] 调试方式)先跑通协议,再拿前端对齐——两边帧格式必须一致。
- 收发不上先查:端口对不对(60610)、`binaryType` 设了没、帧拼接顺序对不对。

## 速记

- 一问一答用 fetch(Promise + await);双向实时用 WebSocket。
- WebSocket 记 4 事件:onopen/onmessage/onerror/onclose,`send` 发。
- 传二进制先设 `binaryType='arraybuffer'`。
- Bridge 帧 = `{JSON header}<二进制>` 直接拼接,前端拼法必须和插件一致。
- 长连接要做:背压、断线重连、心跳、状态机。
