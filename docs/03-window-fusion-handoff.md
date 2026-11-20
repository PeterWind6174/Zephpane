# 远程窗口融合的原理与原型任务书

**交付对象**　负责分析代码和实现原型的 Codex  
**技术基线**　真实 Windows 11 x64 被控机 → Apple Silicon Mac，连接 Windows 当前已登录的桌面。  
**配套文档**　[01 · 产品架构与商业许可](01-architecture.md) 确定产品范围和授权方式；[02 · 协议与完整实现](02-implementation.md) 定义 P2P、Relay、剪贴板、文件和端口转发。本文补充逐窗画面、输入、驱动选型，以及第一批代码该怎么写。

## 1. 要复现的体验

Windows 上有一个写了半段文字、尚未保存的记事本。用户在 Mac 上从整屏远控手动切到融合模式，选中记事本和浏览器，得到两个可与 Finder 交错排列的 Mac 窗口。在记事本按「另存为」时，Mac 能呈现对应对话框，文件名输入到该对话框。切回整屏，原进程、原文和对话框状态都保留。若后台还有文件传输，切换视图也应让它继续。

**首版采用用户态 WGC 按窗口捕获，在 Mac 端为每个授权窗口创建 AppKit 窗口。先验证窗口行为，再优化采集到编码器之间的拷贝。** 虚拟显示驱动留给无头主机或专用虚拟显示器需求，进入开发前要有复现用例。

## 2. Parallels Tools 给了我们什么依据

| 问题 | 公开证据支持的结论 | 对本项目的意义 |
|---|---|---|
| Coherence 用了显示驱动吗？ | 是。Parallels 于 2017 年复核的 KB 123069 同时要求排查 WDDM 虚拟显示适配器、Coherence Service 和用户态进程 `prl_cc.exe`。另一份故障文档还列出 `coherence.exe` 与 `prl_hook.dll`。[KB 123069][pkb] · [KB 112647][phook] | 可借鉴驱动、会话服务和宿主窗口管理的分工。故障排查材料没有披露当前版本如何获取每个窗口的像素，也没有量化驱动带来的性能提升。 |
| 为什么同机虚拟化是重要差别？ | Parallels 管理同一台 Mac 的虚拟显示设备和宿主桌面。用户手册确认 Coherence Tools 提供跨桌面的应用窗口体验；具体每帧如何传递属于未公开细节。[Parallels Tools 手册][pmanual] | 我们的物理 PC 与 Mac 之间始终有网络。输入往返、编码排队、丢包和解码呈现都应单独测量。把 Windows 采集做快，只能改善其中一段。 |
| 虚拟显示驱动能提供独立输入焦点吗？ | 微软 IDD 是基于 UMDF/IddCx 的**用户态显示驱动**，用于报告虚拟显示器、提供显示模式和接收显示输出。新显示器仍可处在同一交互会话中。[微软 IDD 文档][idd] | 额外显示器可给远程应用安排显示空间。独立键盘焦点、第二套光标或独立登录会话需要另行设计。首版以一个控制者操作当前会话为边界。 |

## 3. Petal 的实际数据路径

源码统一固定在提交 `7c3b34fd8ad76c838ffb1667893ed55b0387f1b3`。下面追踪 **Windows 逐窗发送 → Mac 原生客户端接收**。Petal 上游用于多人共享，媒体与控制消息经过 LiveKit SFU。生产版的一对一连接及自建 TURN 方案沿用文档 02；原版 SFU 可先用于建立可复现的窗口基线。[上游架构][architecture]

| 环节 | 在源码里能看到什么 | 实现时要抓住的边界 |
|---|---|---|
| Windows 窗口 → 图像 | WGC 为目标窗口提供 D3D11 帧。所查路径调用 `CopyResource` 复制至 staging 纹理，`Map(D3D11_MAP_READ)` 后按实际行距把 BGRA 复制到 CPU 内存。[Windows 捕获实现][wc] | 捕获已经细化到窗口，无需从整张桌面裁剪。纹理宽度、有效内容尺寸与行距要分别处理。 |
| 原始图像 → 视频轨 | 发布端 `push_bgra` 把 BGRA 转成 I420，交给 WebRTC 视频源。使用上游 Windows Media Foundation H.264 硬编后端时，编码器适配层还会取 I420、分配普通内存缓冲，并转换成紧凑 NV12 送给 MFT。[发布端][pub] · [MF 编码器 C++ 实现][enc] | **已有硬编码能力，仍有 GPU→CPU 读回和 CPU 格式转换。** 是否实际选中硬编要从目标机器的运行统计确认。 |
| 视频轨 → Mac 窗口 | 接收端为每个远端共享窗口建立无边框 `NSPanel`。当解码器交付原生 `CVPixelBuffer` 时，代码将它封装为 `CMSampleBuffer`，送入 `AVSampleBufferDisplayLayer`。软件 I420 帧另有转换路径。[接收端][sub] · [面板][panel] · [显示层][display] | 可直接借用 AppKit 窗口和系统显示层。无 CPU 拷贝的描述仅适用于已拿到原生缓冲的显示段。每个应用独立的 Dock 图标仍是单独功能。 |
| Mac 输入 → Windows 应用 | 输入带目标窗口与授权。Windows 侧验证身份、前台或命中条件，按操作选择输入路径，如 `SendInput` 或向合适的窗口投递滚轮消息。部分被遮挡的 Chromium 窗口可能不响应已投递的滚轮。[Windows 控制实现][input] · [微软 SendInput][send] | 用「文字实际写到哪个窗口、页面实际滚动了多少」验收；API 已接受输入只能说明成功发出了操作。窗口被遮挡时能够获取画面，也不保证能安全向它输入。 |

## 4. Codex 的第一批工作

### A. 建立双窗口基线

1. 先读目标仓库已有的构建入口与上述固定源文件，列出采集、媒体轨、窗口与输入分别归谁管理。把设备型号、Windows/macOS 版本、上游提交与选中的实际编码器记入复现说明。
2. Windows Agent 只在当前交互用户会话枚举和捕获已获准的窗口。每个窗口使用稳定的会话内标识和授权代际，本地维护 HWND、进程及窗口实例的映射。Mac 按视频绑定建立两个独立面板；应用销毁并重开，即使复用了 HWND，也要创建新绑定。
3. 保留物理桌面上的应用及未保存内容。拖动 Mac 窗口先只改变本地位置；远端改尺寸是独立请求，Windows 回报实际尺寸和版本后再更新点击映射。关闭本地查看窗口默认解除订阅；「关闭远端应用」另作显式操作。
4. 模态对话框跟随已验证的 owner，不能仅凭同一 PID 自动共享所有窗口。无法确认归属时暂停相关输入并提示用户选择。切换或撤权时，停止旧源、清显示队列、丢弃旧代际回调。只获准逐窗的连接遇到捕获失败时保持空白或显示暂停状态。
5. 第一轮只要两个真实窗口通过交互验收。可在测试分支沿用 Petal 的 SFU；接入正式网络时，窗口模块只提供帧、身份和生命周期事件，共用一条一对一连接承载认证后的控制、文件和端口通道。

### B. 做一次能比较的 GPU 优化

| 版本 | 路径 | 要拿到的证据 |
|---|---|---|
| A · 基线 | WGC 纹理 → staging 读回 → BGRA → I420 → 现有 WebRTC 编码后端 | 每帧读回字节、Map 等待、CPU 转换时间、编码器名称及出帧率。若选中上游 MF H.264 硬编，也记录其内存 NV12 转换。 |
| B · 候选 | WGC 纹理 → 同一适配器上 GPU 转换为编码器接受的格式（优先探测 NV12）→ DXGI 表面输入 → WebRTC 编码回调 | 目标 MFT 确实接受纹理，关键帧与码率调整正常；画质、帧率、CPU 时间及端到端 p95 可与 A 对比。 |

实现 B 前查询 MFT 的 `MF_SA_D3D11_AWARE`、输入媒体类型和目标适配器。支持时配置 DXGI 设备管理器，用 `MFCreateDXGISurfaceBuffer` 封装纹理。缓冲池要处理编码器保留表面的生命周期，以及设备丢失、resize 和不兼容适配器。可以扩展已有 WebRTC 编码器适配器，继续让 WebRTC 管理码率、关键帧与网络反馈；已经编码的 H.264 应进入编码回调，再送入传输。无法协商纹理输入时回退到 A，并记录原因。[MFT D3D11 能力][aware] · [DXGI 表面封装][surface] · [H.264 输入格式][h264]

用相同分辨率、编解码格式、目标码率、网络和测试内容做 A/B。先测单窗 1080p/30，再测双窗动态内容和连续拖动边缘。分段记录采集、读回、转色、编码、传输、解码、呈现；同机分段使用单调时钟。端到端输入延迟可用高速相机同时拍到输入触发和 Mac 反馈。报告样本量、p50/p95、丢帧与 CPU/GPU 占用。只有在实际追踪确认后，才把某一段描述为消除了全帧读回。

### C. 决定是否做驱动

| 真实故障或产品需求 | 下一步 |
|---|---|
| 两个窗口能正确捕获，但 CPU 高或拖动卡顿 | 先做 B，检查读回、编码器重建和网络排队。 |
| 「另存为」不出现、错窗输入、焦点跳动 | 修正窗口所有者、授权代际、输入命中和焦点同步，复测同一会话的系统限制。 |
| 无头 PC 没有合适的桌面显示目标，或用户明确希望把远程应用放在专用虚拟屏 | 建独立的 IDD 实验，测试增删显示器、热插拔、远端布局恢复与安装/卸载。虚拟屏仍需逐窗授权。 |
| 希望远程用户和现场用户能同时独立打字 | 作为「独立会话」另行评估，明确原有窗口是否可见、应用状态如何保留以及 Windows 授权；普通融合模式继续服务当前会话。 |

## 5. 验收与交接

| 场景 | 客观通过条件 |
|---|---|
| 窗口身份与隐私 | 两个同名窗口都能独立编辑；关闭其中一个并重开后，旧输入和旧帧不进入新窗。用带唯一标记的无关窗口遮挡目标，录屏中不出现无关像素。 |
| 对话框与输入 | 从 Mac 唤起「另存为」，中文输入法提交一次且只到目标框。现场用户抢焦点时暂停或重新确认，禁止把后续按键送给新前台。 |
| 视图切换 | 用有未保存文字的同一进程往返切换整屏/融合 50 次；无遗留捕获、解码器或按键，文件传输继续。选中目标最小化、锁屏或受保护时有可见状态。 |
| 网络与商业边界 | 同局域网检查直连；另在两端禁 UDP 的环境实际通过自建 TURN/TLS 完成视频与输入。自建配置禁用官方服务后仍能运行。个人非商业全功能免费、企业端点商业许可及官方 Relay 订阅沿用文档 01。 |
| 本轮已有证据 | 固定版本源码已核对，11 个上游窗口协议测试体已复跑，11/11 通过。覆盖轨道名、源类型、缩放、层级等元数据。Windows→Mac 真机联测、GPU A/B 和中继实测均待执行。 |

**可直接交给 Codex 的任务**

> 按 01、02、03 的顺序阅读文档，然后检查现有仓库和构建环境。先给出要修改的实际文件和已有可复用接口，再实现本文件第 4 节 A 的双窗口原型。用未保存的记事本和一个浏览器验证窗口身份、对话框、输入与模式切换。保留可运行的基线和测量记录，再评估低拷贝路径。交付代码、双端构建/运行方法、已执行测试及其结果；缺少某一平台时写清未测项，并完成当前环境可验证的部分。

[pkb]: https://kb.parallels.com/en/123069
[phook]: https://kb.parallels.com/en/112647
[pmanual]: https://docs.parallels.com/landing/pdfm-ug/v26-zh-cn/parallels-desktop-for-mac-26-yong-hu-zhi-nan/gao-ji-zhu-ti/an-zhuang-huo-geng-xin-parallels-tools/parallels-tools-gai-shu
[idd]: https://learn.microsoft.com/en-us/windows-hardware/drivers/display/indirect-display-driver-model-overview
[architecture]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/docs/ARCHITECTURE.md
[wc]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/src-tauri/src/windows_screen_capture.rs#L1318-L1383
[pub]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/src-tauri/src/transport/publisher.rs#L10000-L10132
[enc]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/vendor/webrtc-sys/src/mf/h264_encoder_impl.cpp#L658-L731
[sub]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/src-tauri/src/transport/subscriber.rs#L1997-L2052
[panel]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/src-tauri/src/compositor.rs#L16-L29
[display]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/src-tauri/src/native_display.rs
[input]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/src-tauri/src/windows_remote_control.rs
[send]: https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput
[aware]: https://learn.microsoft.com/en-us/windows/win32/medfound/mf-sa-d3d11-aware
[surface]: https://learn.microsoft.com/en-us/windows/win32/api/mfapi/nf-mfapi-mfcreatedxgisurfacebuffer
[h264]: https://learn.microsoft.com/en-us/windows/win32/medfound/h-264-video-encoder
