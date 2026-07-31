# 前置
* 文档地址 
	https://studio.tripo3d.com/help.html#api
* 本地库地址
	/Users/vast/DccDev/Tripo3D-Unity-Bridge
* Git库地址
	https://github.com/vast-enterprise/Tripo3D-Unity-Bridge
# 项目概览
## 主要功能
* **Tripo3D Unity Bridge** 是一款轻量级插件，可将 Unity 直接连接到 Tripo Studio 首页。通过此 Bridge，您可以将生成的模型从浏览器直接发送到 Unity 编辑器中，无需手动下载、导入或额外的步骤。
* 基本技术分析：
	* 在Editor内运行WebSocket，接受web前端传输的模型信息
	* 使用分片传输，传输完成后拼装完整文件
	* 对文件进行预处理后导入，按渲染管线配置贴图，材质
	* 实例化到世界坐标原点
* 详细分析见[[Unity Bridge]]
# 24H 项目（TA / 渲染管线）
* 本地库地址
	/Users/shuqiwang/UnityProject/Project24H
* Unity `6000.3.10f1` + URP `17.3.0`（已 embed），PC Renderer 为 Deferred+
* 预研与需求拆解见 [[Unity 24H TA预研]]
* 自定义 Ramp 灯光实施见 [[Ramp Light 知识地图]]