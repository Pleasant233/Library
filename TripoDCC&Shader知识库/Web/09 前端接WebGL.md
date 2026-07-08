# 09 前端接 WebGL

> 给你这种「会 OpenGL / GLSL,但没在浏览器里接过图形」的人。
> 重点讲**前端侧怎么把 GLSL 跑起来**,GLSL 本身你已经会了(见 [[08 Web3D与AI]] 的模糊 shader、描边)。
> 对应你的业务:Web 端 shader(ts/js + WebGL + glsl,见 [[任务需求]])、[[Tripo Orbit]] 的网页版。

## 一、WebGL 是什么

**WebGL = 浏览器里的 OpenGL ES**。你已有的 OpenGL 知识几乎直接迁移:

| 你熟悉的 | 浏览器里对应 |
| --- | --- |
| OpenGL / OpenGL ES | **WebGL**(基于 OpenGL ES 2.0/3.0) |
| GLSL | **GLSL ES**(几乎一样,注意精度限定符) |
| 窗口/FBO | `<canvas>` 元素 |
| `glGetUniformLocation` 等 C API | 同名的 JS 方法(`gl.getUniformLocation`) |

**关键差异**:
- 没有 C/C++,用 **JS/TS** 调用 `gl.*` API,风格几乎一一对应,只是换了语言外壳。
- 渲染目标是页面上的 `<canvas>`,不是操作系统窗口。
- **WebGL 1** 对应 GLSL ES 1.0(`attribute`/`varying`,你题库里的 shader 就是这版);**WebGL 2** 对应 GLSL ES 3.0(`in`/`out`)。
- 更现代的选择是 **WebGPU**(对标 Vulkan/Metal,[[Tripo Orbit]] 的技术栈里提到了),但普及度还不如 WebGL。

## 二、从零画一帧的最小流程

在浏览器里跑 shader,前端要做这几步(和 OpenGL 一模一样的套路,只是 JS 写法):

```ts
// 1. 拿到 canvas 和 GL 上下文(相当于创建 GL context)
const canvas = document.querySelector('canvas')!
const gl = canvas.getContext('webgl2')!   // 或 'webgl'

// 2. 编译着色器(和 glCompileShader 一样)
function compile(type: number, src: string) {
  const s = gl.createShader(type)!
  gl.shaderSource(s, src)
  gl.compileShader(s)
  if (!gl.getShaderParameter(s, gl.COMPILE_STATUS))
    throw new Error(gl.getShaderInfoLog(s)!)  // 编译报错原文,务必打印
  return s
}
const vs = compile(gl.VERTEX_SHADER, vertexSrc)
const fs = compile(gl.FRAGMENT_SHADER, fragmentSrc)

// 3. 链接成 program
const program = gl.createProgram()!
gl.attachShader(program, vs); gl.attachShader(program, fs)
gl.linkProgram(program)
gl.useProgram(program)

// 4. 准备顶点数据(VBO,和 glGenBuffers/glBufferData 一样)
const buf = gl.createBuffer()
gl.bindBuffer(gl.ARRAY_BUFFER, buf)
gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([...]), gl.STATIC_DRAW)

// 5. 绑定 attribute
const loc = gl.getAttribLocation(program, 'aPosition')
gl.enableVertexAttribArray(loc)
gl.vertexAttribPointer(loc, 2, gl.FLOAT, false, 0, 0)

// 6. 传 uniform(和 glUniform* 一样)
gl.uniform2f(gl.getUniformLocation(program, 'uResolution'), canvas.width, canvas.height)

// 7. 绘制
gl.drawArrays(gl.TRIANGLES, 0, 3)
```

> 你会发现:**每一步都能在 OpenGL 里找到对应**。前端要额外操心的只是「怎么拿 canvas、怎么把数据从 JS 传进去」。

## 三、GLSL ES 的几个前端坑

你的 GLSL 功底够,但浏览器版有几个必须注意的:

1. **必须写精度限定符**:片元着色器开头要有 `precision highp float;`(题库里那份描边 shader 就写了),否则报错。
2. **纹理采样函数名**:WebGL1 是 `texture2D(...)`,WebGL2 是 `texture(...)`。你题库的 shader 用的 `texture2D` 是 WebGL1 写法。
3. **UV 原点**:纹理坐标 (0,0) 在**左下角**,和 OpenGL 一致;但图片加载进来常是**左上角原点**,采样上下颠倒时用 `gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true)` 或在 shader 里翻转 `vUV.y`。
4. **循环限制**:WebGL1 里 `for` 循环次数必须是**编译期常量**(不能用 uniform 当上界)。所以题库那道高斯采样把半径写死成 `i <= 2`,而不是 `i <= uRadius`——这是 WebGL1 的硬限制,不是随便写的。

## 四、渲染循环与前端事件模型

原生里你有个 `while` 主循环渲染。浏览器**不能**用 `while(true)`——会卡死单线程(见 [[04 浏览器原理]] 事件循环)。正确做法是 `requestAnimationFrame`:

```ts
function frame(time: number) {
  gl.clear(gl.COLOR_BUFFER_BIT)
  // ... 更新 uniform、drawArrays ...
  requestAnimationFrame(frame)   // 请求下一帧,浏览器按屏幕刷新率(~60fps)调用
}
requestAnimationFrame(frame)
```
- `requestAnimationFrame` = 「下一帧渲染前叫我一次」,自动对齐屏幕刷新率、页面切后台时暂停,省电。
- 这就是前端的「主循环」,是事件驱动模型下的正确姿势。

## 五、实战:别自己造轮子,用 Three.js

生产项目里,直接手写裸 WebGL 很少见,通常用 **Three.js**(WebGL 封装库,类似一个轻量渲染引擎):
- 帮你管理场景图、相机、光照、加载 glTF/FBX 模型(呼应你的 [[任务需求]] 三维数据结构方向)。
- 你仍能写**自定义 shader**:用 `ShaderMaterial` 传入你的 GLSL,后处理用 `EffectComposer`(题库那道描边就是典型后处理 pass)。
- **迁移建议**:先用 Three.js 快速搭起场景,把精力放在你擅长的 shader/后处理上,而不是重写一遍 WebGL 样板。
- WebGPU 场景可看 **three.js 的 WebGPURenderer** 或原生 WebGPU API。

## 六、调试

- **报错原文**:shader 编译失败时,`getShaderInfoLog` 会给出行号和原因,一定打印出来(呼应 [[00.2 与同事对接]] 的「报错原文」原则)。
- **Spector.js**:浏览器扩展,能抓一帧所有 WebGL 调用,看每个 draw call 的状态,类似 RenderDoc。
- **精度问题**:移动端 `highp` 可能被降级,颜色/位置异常先怀疑精度。

## 速记

- WebGL = 浏览器版 OpenGL ES,API 一一对应,只是换成 JS 调用。
- 流程:canvas→compile shader→link program→传 VBO/uniform→drawArrays。
- GLSL ES 坑:必写精度、`texture2D`(WebGL1)、UV 翻转、循环上界要常量。
- 主循环用 `requestAnimationFrame`,别用 while。
- 生产用 Three.js 搭场景,自己专注写 shader。
