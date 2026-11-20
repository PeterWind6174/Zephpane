# Windows → macOS 远程桌面与窗口融合 · 实现方案

**开发入口、实施顺序和验收｜2026-11-20｜总体设计见 `01-architecture.md`**

## 1. 第一轮应该做出什么

准备一台已登录的 Windows 11 x64 真机、一台 Apple Silicon Mac，以及能分别接入的两张网络。Windows 记事本里先写一段不保存的文字，再打开浏览器。Mac 连上整张桌面，点「融合」，选这两个已运行的窗口。两窗都要能盖在 Finder 前后、各自缩放、接收输入。切回整屏后检查同一进程、同一段文字仍在。另起一个文件传输，重复切换，验证连接级任务持续运行。

**实施顺序是窗口原型 → 一对一网络 → 模式切换与权限 → 通用远控 → 自建与商业交付。** 最难的两个问题是普通 Windows 用户会话能否可靠控制多个已打开窗口，以及禁 UDP 的环境能否完成独立于厂商的中继连接。两个都设为早期验收项。

## 2. 从哪些源文件开始

| 入口 | 可接入的部分 | 要改动的边界 |
|---|---|---|
| Petal 的 [Windows 目标标识][target]、[WGC 管线][wgc] | 复用用户会话里的显示器/单窗捕获与 D3D11 资源管理，避免把指针宽度的 HWND 直接暴露为跨端编号 | 媒体输出接到本项目的本地 WebRTC 视频源。确认代码内 LiveKit 对象、队列和帧所有权，对窗口与显示器分别建立适配器。 |
| Petal 的 [macOS 合成器][compositor]、[原生帧显示][display] | 它已将每个远程窗口做成独立 NSPanel，并有 CVPixelBuffer → AVSampleBufferDisplayLayer 的显示路径 | 把房间身份及订阅 ID 换成设备/会话/窗口 ID；AppKit 在主线程管窗口，解码队列只递交帧。首版可沿用原生显示层，毋须先重写 Metal 渲染器。 |
| Petal 的 [远控状态机][control]、[Windows 输入][inputsrc]、[协议契约][contracts] | 参考窗口授权、输入顺序、按键释放、逐窗文本复制/粘贴语义 | 解开房间消息与权限状态的绑定。就地测试 Win32、Chromium、模态框和中文输入。其定向文本粘贴与全局剪贴板同步应分开实现。 |
| 固定版 Petal 自带的 [PeerConnectionFactory][factory] 和 [PeerConnection][pc] | 底层存在建连接、加轨、加 transceiver 和建 DataChannel 的入口；可以先沿用同一 Rust/WebRTC 绑定体系 | 以原生 `PeerConnection` 取代 LiveKit Room/SFU。信令、设备身份、ICE 重启、轨道映射和订阅恢复由本项目实现。验证实际构建对 Windows/macOS、目标编码器、TURN/TLS 的支持。 |
| ManyDesk 的 [剪贴板][clip]、[文件处理][fm]、[TCP 监听][tun] | 参考系统剪贴板格式、文件操作和本地回环监听，复用可独立测试的适配逻辑 | 移除 QUIC stream、上游会话状态和 UI 依赖，连接本项目的数据通道。保留原作者许可声明。上游的整屏 H.264 帧协议只供参照。[帧格式][mdp] |
| [coturn][cot] | 采用现成 STUN/TURN 数据面 | 自建包自带信令和签发凭据服务。官方托管增加订阅与计量，保持两种部署使用相同协议。 |

**避免同时维护两套传输。** 原版 Petal 的 LiveKit SFU 可用于第一次双机窗口验证；发行版的一对一模式采用直接 WebRTC 或标准 TURN。若团队已经熟悉 ManyDesk，也可以保留其原生 QUIC 做早期局域网原型，但其视频帧缺少窗口编号、TURN 客户端只能走 UDP，且默认发现使用公共网络。将它改成正式分支前，应与上述单栈方案分别估算多窗口、禁 UDP 和自建信令三项工作量。[ManyDesk 网络解析][turnparser] · [公共发现入口][discovery]

## 3. 调用与数据怎么拆

采用一个仓库，按平台组织端点代码。Windows、macOS、Linux 各预留 `endpoints/<platform>/host/`（被控）和 `viewer/`（控制），长期目标是各平台可双向组合，均支持整屏和融合视图。下表只列首轮 Windows 被控、macOS 控制所涉及的模块。Windows/Rust 与 macOS/AppKit 可以走薄 FFI。

| 交付物 | 模块 | 对外职责 |
|---|---|---|
| Windows 端点的被控角色 | `endpoints/windows/host/` 下的 `session`、`surfaces`、`capture`、`input`、`io` | 选当前交互会话；枚举、授权和捕获源；接受输入；处理剪贴板、文件、TCP。若有系统服务，只承担生命周期，通过受限 IPC 与当前用户进程交互。 |
| macOS 端点的控制角色 | `endpoints/macos/viewer/` 下的 `windows`、`renderer`、`events`、`transfers` | 面板、逐窗显示、事件路由、拖放与文件传输。共享库完成重连与权限，不把授权判断丢给 UI。 |
| 端点共用库 | `session-core/protocol`、`rtc`、`permissions`、`transfer` | 能力协商、轨道注册、模式切换、文件续传与流控。协议规范、兼容性样例可独立公开；自有端点实现适用端点许可。 |
| 自建/托管服务 | `server/signaling`、`turn-credentials`、`deploy`，官方另有 `hosted/billing` | HTTPS/WSS 配对与信令、设备注册和撤销、短期 TURN 凭据；coturn 转发。单机部署可用 SQLite；多实例托管再引入共享状态库和计量账本。 |

### 3.1 模式切换与窗口身份

设备配对后建立一个逻辑会话。应用层为每次连接分配 `session_id`、授权代际 `epoch`，为每个捕获实例分配随机 `surface_id`；Windows 保留 `surface_id → HWND/PID/窗口实例` 的本地映射。HWND 可重用，同名进程也可有多个窗口。目录只返回当前策略允许展示的标题与图标，缩略图视为画面授权。

| 消息或事件 | 关键信息 | 必须满足的语义 |
|---|---|---|
| `Hello / Capabilities` | 协议版本、视图、媒体格式、最大并行源、文件/剪贴板/转发能力 | 取双方交集。旧端点可走用户已授权的整屏模式，不能把「只分享窗口」降级为整屏。 |
| `RequestMode / ModeCommitted` | 请求 ID、期望旧代际、新模式、获准窗口集合、新代际 | 手动触发，主机最终裁决。先停旧源与输入、清队列并让 Mac 清屏，再绑定新源。切到整屏必须已有整屏授权或再次确认。切换失败保持空白或原有安全授权。 |
| `SurfaceSnapshot / Added / Changed / Closed` | 代际、单调版本、窗口 ID、所属弹窗、像素边界、DPI、最小化状态 | 快照替换整个窗口表，增量幂等应用。ID、代际或版本不一致就重新要快照。关闭后清解码器与输入映射，清除末帧。 |
| `TrackBinding / FirstFrameReady` | 代际、窗口、轨道 MID、绑定版本、内容尺寸 | 先注册后显示，首个可解码帧到达再显示窗口。切换授权时新建可区分的轨道绑定，丢弃旧订阅和旧解码回调；严禁把上一轮桌面末帧放进新窗口。 |
| `Focus / Resize / Input / ReleaseControl` | 窗口、代际、几何版本、授权、序号、操作 ID | 主机返回实际尺寸与前台状态；过时代际和尺寸的输入丢弃。失焦、撤权、掉线释放已按键与鼠标键，避免卡住快捷键。 |
| `Resume / Revoke` | 新连接、最后确认的传输偏移、当前权限、重建的窗口快照 | 恢复前重新验证端点和权限，只续传可验证的任务。窗口授权、MID 或输入焦点不能照搬。主动撤权立即停源、停输入，清空缓存。 |

同一连接中使用一个可靠、有序的控制 DataChannel，一个可合并旧数据的指针移动通道，以及单独的可靠批量通道。点击应随可靠事件携带点击坐标和几何版本；键盘、按下、抬起都要求可靠。文件和 TCP 传输设置分块、队列上限与背压，给输入和视频留带宽。多个 DataChannel 仍会共享底层拥塞和资源，需要在慢网压测中实测交互延迟。[WebRTC 数据通道 RFC 8831][rfcdata]

### 3.2 Windows 端采集和输入

1. **捕获源。** 每个选中的顶层窗口用 WGC 的 `CreateForWindow(HWND)` 建立会话。整屏从显示器源采集。用窗口事件订阅追踪显示、隐藏、销毁和位置变化，辅以周期性枚举纠错。在帧大小变动时重建资源并更新几何版本。MVP 先支持 2 个窗口，同台硬件验收后再开放更高上限。[微软窗口捕获 API][wgcapi] · [SetWinEventHook][hook]
2. **遮挡、最小化与弹窗。** 所选窗口被无关窗口遮挡时，检查捕获流中是否仍只有目标内容；应用停止绘制或最小化就标记暂停，恢复后请求关键帧。优先把已确认归属且获准的模态框作为子窗口呈现；归属不明时提醒在主机操作。Windows 的 `IncludeSecondaryWindows` 有版本与几何筛选条件，默认关闭，单独测试后再开放。[微软次级窗口定义][secondary]
3. **前台与坐标。** 将 Mac 内容区中的点减去留黑边距，转换为归一化坐标，结合最新实际捕获矩形还原为 Windows 物理像素，再映射到当前虚拟桌面坐标。混合 DPI、负显示器原点、窗口边框、Mac 的纵轴方向分别测试。输入前复核窗口实例、前台及指针命中，失配就不注入；焦点检查与输入之间存在竞争，检测变化要停止操作并提示重试。一次只有一个远端键盘焦点。[SetForegroundWindow][focus] · [SendInput][send]
4. **本地用户与权限。** 窗口捕获在当前登录用户进程运行，Session 0 服务不负责操作桌面。UIPI 可能阻止对高完整性应用输入；安全桌面、受保护画面、锁定或会话切换要明确暂停并撤销旧输入。窗口模式下禁止自动用整屏画面填补黑帧。[Windows 交互式服务][service] · [显示内容保护][protect]

### 3.3 Mac 端窗口与系统集成

- 一个远程顶层窗口对应一个本地 AppKit 窗口或可接收焦点的面板。模态子窗记录远端 owner，按需附着在对应父窗。AppKit 在主线程创建、显示、排序和销毁，解码层按最新绑定交帧。优先使用 Petal 的原生显示代码；先测主线程阻塞、拷贝次数和 Retina 文本清晰度，再决定是否换成 Metal。[Petal 显示实现][display]
- 本地移动只影响 Mac 布局。拖边默认可先本地缩放预览，再发限频的远端尺寸请求，以主机实际尺寸完成布局。缩到 Dock 在 Mac 端暂停或降低该轨道优先级。关闭本地查看窗口解除订阅，显式「关闭远端应用」才向 Windows 请求正常关闭，保留未保存提示。
- 首版支持同一 Mac 客户端名下的多个独立窗口、应用切换时焦点正确。**每个远程应用一个 Dock 图标或一个 ⌘Tab 项是另一项功能**，需为应用身份、启动器、签名及清理做单独设计。输入法将「提交的 Unicode 文本」与「物理按键/快捷键」分流，验证中文选词只提交一次。应用内快捷键和 Mac 系统快捷键允许配置。
- 默认融合只传入选窗口画面。系统音频、全局剪贴板、文件与 TCP 转发另设会话权限。系统声音可能包含其它应用，应在启用时标明来源；若需要逐应用音频，另做进程音频采集与回退策略。

### 3.4 剪贴板、文件和端口转发

| 功能 | 首版实现 | 正确性条件 |
|---|---|---|
| 双向剪贴板 | Windows 监听剪贴板变化，Mac 跟踪 NSPasteboard 的 changeCount。先支持 UTF-8 文本，另测 PNG；远端逐窗「复制/粘贴」可复用 Petal 的请求/响应语义 | 来源端 ID、修订号和摘要防回环，权限按方向区分；断开后不补发上次遗留内容。按字节限流，大文件转文件通道。 |
| 双向文件 | 文件面板和拖放共用传输 ID、大小、分块、确认偏移与 SHA-256。接收端写临时文件，同卷验证后原子改名。目录相对路径、重解析点/符号链接与文件覆盖单独校验 | 断线重连须复核大小与内容身份；源文件变化则重开任务。覆盖由用户选择，失败不破坏旧文件。模式切换不撤销既有文件传输授权。 |
| TCP 本地端口转发 | Mac 在 loopback 监听，为每个本地连接开独立逻辑流；Windows 仅连接用户或策略明确授权的 host:port。按方向传送数据、EOF 和错误，正确处理半关闭与背压 | 验证本机工具可访问授权 Windows 服务；拒绝未授权目标，转发关闭后释放监听和远端套接字。网络重连后可恢复监听，已断开的 TCP 连接由应用重建，禁止重放旧字节。 |
| 反向或 UDP 转发 | 双向 TCP 字节流已包含在本地转发中。若要 Windows 主动监听并转回 Mac，新增带独立授权的反向监听。UDP 端口转发另定义数据报、超时与配额 | 在客户端能力列表写清已支持方向。TURN 经 UDP/TCP/TLS 是连接建立方式，与给用户转发何种应用端口是两件事。 |

## 4. 自建部署、端点身份与收费

**自建交付包。** 一个部署入口启动信令/设备配对、凭据签发、coturn 和持久卷，支持自有域名、证书、密钥与备份。示例端点配置包含 `signaling_url`、`turn_credentials_url`、信任的服务器标识以及 `allow_official_fallback=false`。运维通常为信令提供 HTTPS/WSS 443，为 TURN 提供 UDP/TCP 3478、TLS 5349 与明确的 relay 端口段；禁 UDP 场景可在独立公网 IP 的 443 上提供 TURN/TLS。不能把普通 HTTP 反向代理当作 TURN 代理。公网地址、NAT 映射和防火墙规则随部署文件说明。[coturn 配置样例][cotcfg]

**连接流程。** 自建或官方信令以双方已验证的设备身份和短时配对授权交换 SDP/ICE。首次配对显示可核对的指纹或短码；双方对会话随机数、身份和 DTLS 证书指纹进行绑定确认，持久信任可撤销。仅依赖信令转发 SDP 时，无法防御信令端替换对端指纹。已配对的无人值守访问需单独由 Windows 所有者启用，按设备和能力授予。中继共享密钥仅保留在服务端，按会话签发短时 TURN 凭据；长期会话在过期前更新凭据并验证续期/重连行为。[WebRTC 安全架构][rfcsec] · [coturn][cot]

**选路与降级。** 正常模式由 ICE 检查直连与中继候选，尽量选择可用直连；测试工具另提供「强制 relay」。状态栏读取实际选中的 candidate pair，区分直连、自建中继与官方中继。TURN/TLS 的 TCP 是端点到 TURN 的连接；不能只看服务器端口开放就判定双端禁 UDP 已通。切换网络时尝试 ICE restart，失效则重建连接、重发窗口快照、重新检查授权。自建模式不能悄悄切到会产生费用的官方节点。[ICE][ice] · [TURN 服务端能力][cot]

**自建版不需要回拨官方。** 设备配对、注册、授权、ICE 凭据、窗口选择、文件和 TCP 转发全部能在自建域内完成。更新源可自配或离线安装。公司可通过离线签名许可文件激活商业端点，自建服务端不校验商业端点注册码。官方中继服务另按订阅发凭据。首版不使用企业网段、邮件后缀或设备指纹来推定商业用途。

**计费记账。** 建议先售官方 Relay 订阅，设置明示的包含流量。账本只接收官方节点的可归属中继用量。定义单位为十进制 GB，若按量则事先公布计量边界，例如两个方向实际转出官方中继集群的用户流量；排除控制探测、节点间重复转发、直连和自建用量。边缘记录会话匿名 ID、节点、方向、计数器代际、累计字节与序号，账本做单调差分与重复事件去重。coturn 的聚合计数口径须用抓包对账，再开放按量扣费。8 Mbit/s 的单向持续视频一小时约 3.6 GB，尚未计音频和传输开销；只作带宽预算。[coturn 配置与统计选项][cotcfg]

## 5. 里程碑与真机验收

| 阶段 | 操作 | 通过标准 |
|---|---|---|
| **P0 融合基线** | 在隔离测试环境按固定版 Petal 构建说明运行 Win→Mac，先用其自建 LiveKit SFU。并排打开记事本和浏览器，逐窗共享，测遮挡、弹窗、输入、缩放与撤权 | Mac 出现两个可独立前后排列的真实窗口；文字进入指定进程，远端不重启。遮挡或撤权时没有无关画面。记录双端版本、权限、录屏和未通过项。 |
| **P1 一对一传输** | 移植两端窗口适配器，只保留原生 PeerConnection、自有信令和 coturn。同 Wi-Fi、异地家宽/手机热点、强制中继、双端禁 UDP 四种环境各跑两个窗口及可靠数据回环 | 日志含候选对、方向、端口与实际 relay 节点；强制中继及禁 UDP 两项均需有真实视频和控制返回。自建时封禁官方域名仍通过。 |
| **P2 交互与模式切换** | 系统缩放 100%/150% 与 Retina、两块屏幕、中文输入法、应用被遮挡、最小化与恢复、模态保存对话框、快速切换应用。物理用户抢焦点、UAC、锁屏、远端窗口关闭及句柄复用 | 无错窗输入和遗留按键；最小化/受保护状态明确。主动撤权和失败回退不暴露桌面。未保存内容和进程实例保留；切换 50 次不遗留窗口、捕获会话与解码器。 |
| **P3 完整远控** | 双向中英文与 emoji；单个 1 GiB 文件中途断线续传、两端哈希比对；边传文件边切窗；用本地客户端连接获准 TCP 服务，测半关闭、断网和撤权；勾选声音后试听 | 复制不循环，文件不损坏或意外覆盖；传输持续且可取消；端口限制和资源释放通过；关闭音频后不再送音频数据。 |
| **P4 托管及发布** | 区分直连、自建、官方三类会话；做双向流量复算、重复用量事件和凭据过期测试。离线验证企业授权，构建签名包、依赖清单和第三方声明 | 前两类官方中继账单为 0；第三类可和服务器记录对账。全自建不依赖官方账号；两端安装与更新可验证签名；法律与第三方分发问题有书面处理结果。 |

**性能测试建议。** 先以一台 Windows 11 普通桌面机和一台 Apple Silicon Mac 作为具名基准，记录具体 CPU/GPU、系统版本、显示器 DPI、编解码器和网络 RTT。建议目标为局域网单个 1080p/30 窗口输入到画面反馈 p95 ≤ 100 ms；两个窗口同时动态更新时分别报告 FPS、码率、CPU/GPU、丢帧和切换后首帧时长。用同一台高速相机拍摄输入触发与 Mac 反馈，或用能关联的端到端标记；避免把两台未同步时钟相减。再测 RTT 80 ms、1% 丢包、大文件并发和中继。上述数字是立项目标，尚无本产品基准结果。

**先保留证据再扩功能。** 建议优先拿到三份录屏：真实应用在 Windows 的重叠遮挡、弹出保存对话框与焦点争用、关闭双端 UDP 后通过自建 TURN/TLS 切窗打字。三项都过，再投入每个应用独立 Dock 图标、企业管理界面与按量收费。

## 6. 已执行的验证与发布清单

### 固定源码与本轮运行结果

| 对象 | 版本或指纹 | 核实方式与结果 |
|---|---|---|
| Petal | `7c3b34fd8ad76c838ffb1667893ed55b0387f1b3` | 阅读 Windows 采集、macOS 合成器、输入处理、RTC 绑定、协议及上游验证文档。双机跨平台的持续实测在上游验证矩阵有明确缺口。 |
| Petal 窗口协议 | Node `v24.22.3`，11 项通过、0 项失败 | 从固定提交的原始测试文件原样选取 11 个窗口元数据测试体，仅替换测试入口导入；使用上游原样的 TypeScript 实现和 JSON 样例。覆盖窗口轨道名 1 项、源类型及发布缩放默认值等 3 项、采集缩放 1 项、层级 2 项、标题/URL 2 项、色彩元数据 2 项。 |
| ManyDesk | `5bf1897062e69fd48ee82cb5ac6c4bbfdf14c31d`，v0.25.10 | 阅读 MDP 帧格式、TURN URL 判断、发现、文件/剪贴板与端口转发。上游 Mac 测试/打包和 Windows 交叉编译记录可追溯；Windows 发布包编译成功不涵盖 Win→Mac 真机联测。 |
| WinFleet / DeskVNC | `6e5b1565f6498eb855f1f0a0f69860ef15902b62` / `81b25747233cb7cd03622793109b80ba1136297d` | 前者确认虚拟显示器迁移、部分路径会重启应用，后者确认导入器拒绝 RemoteApp 配置文件。 |
| 端到端、TURN、性能与商业计量 | 待 P0–P4 | 本环境是 Linux，缺 Windows/Mac 真机和真实 TURN 服务端。浏览器 WebRTC 探针尝试在浏览器启动时受套接字限制，未得到媒体连接结果。没有将源码阅读、CI 或协议测试算作端到端通过。 |

本轮校验过的 Git blob 分别为测试文件 `f9f197f69c3ef221eaba35f8d0fae3cddfbbdc31`、窗口协议实现 `f7a66a4574cd7eb6ba26110ebfe5e5c308137f82`、互操作样例 `3ce0d46938ca3b96fdb96cde692eb1b001d69ea6`。在上游完整开发环境可按该版本的依赖安装方式进入 `web-harness`，运行其 `npm test` 复查更大的测试集；本轮只计上表 11 项。[原测试][tests] · [窗口协议][tracks] · [测试样例][fixtures] · [上游验证矩阵][validation] · [ManyDesk CI][ci]

### 商业发行需处理的具体依赖

- **Petal。** 根项目 Apache-2.0；固定版本的 `@userdispatch/sdk` 未明确给第三方分发授权。上游审计说明它是作者自有包。我们的发行版只取必要模块，移除不需要的 SDK 或取得明确授权；检查自带的 WebRTC、补丁和第三方 Notice。[许可声明][notices] · [审计记录][audit]
- **ManyDesk 与编解码。** 源码在软件回退路径使用 `OpenH264API::from_source()`。Cisco 针对其分发二进制的安排不当然覆盖自行编译的产物。原型可先使用双方协商且构建中可用的 VP8；商用选择 VP8/H.264/其它编码时，针对实际编解码库、硬件路径、发行方式及地区检查许可。系统硬编/硬解也应记录进发布审查。[ManyDesk 编码源码][encode] · [Cisco FAQ][cisco] · [WebM 项目许可][webm]
- **公司许可边界。** 建立依赖锁文件与 SBOM，保留上游版权、许可证及必要修改声明。把自研端点授权、官方托管合同、自建服务端开源许可分成三个文档。保留贡献者商业再许可所需权利。合同确认哪些组织或业务需要付费后，再写具体激活策略。[OSI 开源定义][osi] · [Apache-2.0][apache] · [PolyForm Noncommercial][poly]

[target]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/src-tauri/src/windows_capture_target.rs
[wgc]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/src-tauri/src/windows_screen_capture.rs
[compositor]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/src-tauri/src/compositor.rs
[display]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/src-tauri/src/native_display.rs
[control]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/src-tauri/src/remote_control_core.rs
[inputsrc]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/src-tauri/src/windows_remote_control.rs
[contracts]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/docs/CONTRACTS.md
[factory]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/vendor/libwebrtc/src/peer_connection_factory.rs
[pc]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/apps/desktop/vendor/libwebrtc/src/peer_connection.rs
[clip]: https://github.com/manydesk/many-desk/blob/5bf1897062e69fd48ee82cb5ac6c4bbfdf14c31d/app/src/core/clip.rs
[fm]: https://github.com/manydesk/many-desk/blob/5bf1897062e69fd48ee82cb5ac6c4bbfdf14c31d/app/src/fm.rs
[tun]: https://github.com/manydesk/many-desk/blob/5bf1897062e69fd48ee82cb5ac6c4bbfdf14c31d/app/src/core/tunnel.rs
[mdp]: https://github.com/manydesk/many-desk/blob/5bf1897062e69fd48ee82cb5ac6c4bbfdf14c31d/app/src/mdp/session.rs#L3-L13
[turnparser]: https://github.com/manydesk/many-desk/blob/5bf1897062e69fd48ee82cb5ac6c4bbfdf14c31d/app/src/mdp/turn.rs#L203-L212
[discovery]: https://github.com/manydesk/many-desk/blob/5bf1897062e69fd48ee82cb5ac6c4bbfdf14c31d/app/src/rendezvous.rs#L26-L44
[cot]: https://github.com/coturn/coturn
[cotcfg]: https://github.com/coturn/coturn/blob/master/examples/etc/turnserver.conf
[wgcapi]: https://learn.microsoft.com/en-us/windows/win32/api/windows.graphics.capture.interop/nf-windows-graphics-capture-interop-igraphicscaptureiteminterop-createforwindow
[hook]: https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwineventhook
[secondary]: https://learn.microsoft.com/en-us/uwp/api/windows.graphics.capture.graphicscapturesession.includesecondarywindows?view=winrt-26100
[focus]: https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setforegroundwindow
[send]: https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput
[service]: https://learn.microsoft.com/en-us/windows/win32/services/interactive-services
[protect]: https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowdisplayaffinity
[rfcdata]: https://www.rfc-editor.org/rfc/rfc8831
[rfcsec]: https://www.rfc-editor.org/rfc/rfc8827
[ice]: https://www.rfc-editor.org/rfc/rfc8445
[tests]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/web-harness/tests/contracts.test.ts
[tracks]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/web-harness/src/trackNames.ts
[fixtures]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/contracts/petal-contracts.json
[validation]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/docs/VALIDATION.md
[ci]: https://github.com/manydesk/many-desk/actions/runs/44545748184
[notices]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/THIRD_PARTY_NOTICES.md
[audit]: https://github.com/kiruna-labs/petal/blob/7c3b34fd8ad76c838ffb1667893ed55b0387f1b3/sbom/LICENSE_AUDIT.md
[encode]: https://github.com/manydesk/many-desk/blob/5bf1897062e69fd48ee82cb5ac6c4bbfdf14c31d/app/src/media/encode.rs#L355
[cisco]: https://www.openh264.org/faq.html
[webm]: https://www.webmproject.org/license/
[osi]: https://opensource.org/osd
[apache]: https://www.apache.org/licenses/LICENSE-2.0
[poly]: https://polyformproject.org/licenses/noncommercial/1.0.0
