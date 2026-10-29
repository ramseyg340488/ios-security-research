# iOS 沙箱配置文件与 Entitlement 结构解析——从 Coruna 与 DarkSword 的逃逸目标看权限最小化缺陷

> **摘要**：本文把沙箱配置文件（.sb）与 entitlement 当作两类可审计的配置资产来拆解。先梳理 iOS 沙箱的架构——MACF 框架、Sandbox 策略模块、per-process profile 的加载时机与指定方；再按公开文档与社区整理归纳 profile 的谓词类别，说明每类在评审时该读什么；随后给出 entitlement 与沙箱的双层权限模型，解释「能力申请」与「资源可达」的错位为何是攻击面评估核心。在此基础上，以 Coruna 与 DarkSword 公开披露的失陷后行为为参照，逐类分析最常被用作「出域目标」的能力面及防御侧核对动作，讨论沙箱的结构性弱点与苹果收敛方向，最后给出受管设备的能力审计清单与红旗条目。

# 一、沙箱架构与 profile 生命周期

iOS 沙箱（俗称 Seatbelt）是「框架 + 策略模块 + 配置资产」的组合，分四层理解：

1. **MACF 框架层**。XNU 内核的强制访问控制框架 MACF 源自 FreeBSD 的 TrustedBSD 项目。内核在文件、进程、Mach、IOKit、sysctl 等操作路径上埋设检查点，经 `MAC_CHECK` 宏依次调用各策略模块回调；任一策略拒绝即整体拒绝。
2. **Sandbox 策略模块层**。沙箱策略模块（历史上以 `com.apple.security.sandbox` 内核扩展形态存在，新版 macOS 已并入内核）持有每个受限进程绑定的编译后 profile 并执行匹配。社区研究显示其向 MACF 注册了上百个钩子；iOS 平台级 profile 与内核镜像紧密绑定以防篡改，早期版本用户态侧的规则数据与编译设施可在 sandboxd 等组件中观察到。
3. **profile 资产层**。规则载体是 Scheme 风格的文本 .sb 配置，编译后绑定进程；第三方 App 用与容器绑定的应用 profile，渲染、媒体等系统进程用各自的平台 profile，多来源片段经 import 叠加成最终规则集。
4. **装配与指定层**。进程由 launchd 或父进程拉起；AMFI 在 exec 时校验签名并解析 entitlement；容器管理组件建立数据容器并选定应用级 profile 模板；用户态库经 `sandbox_init` 类接口把规则装入内核。**用哪个 profile、叠加哪些片段，取决于签名 entitlement、任务类别（App/daemon/WebContent）与容器归属三者的组合**，而非进程自行申报。

| 层级 | 载体 | 评估者该看什么 |
| --- | --- | --- |
| MACF 检查点 | 内核 | 哪些操作被埋点；未埋点路径即沙箱盲区 |
| Sandbox 模块 | 内核策略模块 | 规则匹配语义（正则、标签、继承） |
| profile 资产 | .sb 及编译产物 | 允许清单宽度、import 链、参数占位符 |
| 装配指定 | AMFI/launchd/容器管理 | entitlement 如何决定 profile 选择 |

App 与 WebContent 类平台进程的 profile 差异是关键：前者是「困在容器里的普通用户代码」，允许面天然窄；后者要驱动渲染、GPU、通知等真实功能，必须持有实质性的跨进程与设备访问清单。**不可信代码 + 较宽能力面**的组合，正是它在两条在野链中成为第一逃逸跳板的原因。

# 二、profile 的谓词结构：一张能力白名单

以 Apple 公开文档与社区整理为准，.sb 规则由「allow/deny + 操作类别 + 匹配条件」构成，语义是能力白名单：

| 谓词类别 | 管住什么 | 读 profile 时看什么 |
| --- | --- | --- |
| file（读/写/执行/元数据等） | 路径级文件可达 | 默认方向；正则是否退化为全盘可读；容器参数是否严格锚定 |
| mach-lookup / mach 私有端口类 | 能否连接命名 Mach 服务、获取他进程端口 | **允许的服务名清单是出域面第一指标** |
| IOKit 类（打开设备、匹配用户客户端） | 能否探测调用图形、传感器等驱动 | 匹配条件是否只按宽泛类别放行 |
| POSIX 共享内存/信号量类 | 命名共享区、信号量的创建与接入 | 全局命名对象是否可任意 attach |
| sysctl 类 | 内核状态读数 | 是否放行泄漏布局信息的节点 |
| system/process 类 | socket、exec、fork、调试等操作 | 是否允许派生不受限子进程（能力继承旁路） |
| syscall 过滤 | 调用编号级粗粒度拦截（较新版本方向） | 兜底机制，非细粒度语义控制 |
| import 与参数机制 | 规则叠加、模板参数 | 是否引入历史宽松片段；参数有无默认兜底 |

三点阅读方法论：**看默认策略与例外方向**——现代 profile 应默认拒绝、逐项 allow，见到默认放行要立刻追问遗漏面；**看正则与参数的松紧**——锚点缺失会把「一个容器」放大为「一片区域」；**看 mach-lookup 清单**——文件规则决定能读什么，Mach 清单决定能「委托谁干活」，跨进程继承风险集中在这一张表。

边界声明：以上是公开结构的一般性描述，本文不提供针对任何具体系统版本可用的 profile 构造方法；私有片段内容不随版本公开，以官方文档与授权实测为准。

# 三、Entitlement 与沙箱：双层模型的分离、耦合与错位

| 维度 | Entitlement | 沙箱 profile |
| --- | --- | --- |
| 本质 | 进程「可申请的能力」声明 | 能力落地时的「资源可达范围」 |
| 载体 | 代码签名内的属性列表，exec 时由 AMFI 校验 | 编译后规则集，绑定进程 |
| 授予方 | 签名流程与描述文件（公共键有文档，另有大量未公开的系统侧键） | 系统装配链按任务类别与容器选定 |
| 公开性 | 公共键族有文档（应用服务、共享组、安全类前缀等）；私有键按类别描述、不逐枚罗列 | 片段数量大、多数不公开 |

两层不一致产生两类错位，都是评估重点：

1. **有 entitlement 但被沙箱挡住**。功能上表现为访问失败；结构上说明申请面宽于实际可达面——审计要问「为什么申请」，多余申请面既是供应链风险也是画像素材。
2. **沙箱放宽到可访问系统服务、无需显式 entitlement**。平台进程 profile 的 Mach 放行项往往没有对应 entitlement 概念，真正边界在**服务端**：守护进程收到连接后，按「调用方是谁」决定给多少权限。若服务端把连接本身当信任凭据，客户端一条 mach-lookup 放行就等于间接持有服务端特权。**沙箱限制客户端、不约束服务端语义；能力沿 IPC 继承，是 iOS 权限模型最结构性的软肋。**

因此真实最小权限是：**entitlement 交集 ∩ 沙箱可达集 ∩ IPC 服务端校验强度**。单独看任何一层都不构成完整攻击面账本。

# 四、从两条在野链的失陷后行为看「出域目标」

公开事实基线（只述结果面，不复述利用过程）：Coruna 由 GTIG 与 iVerify、Lookout 于 2026 年 3 月详细披露，5 条链、23 个漏洞，覆盖 iOS 13.0–17.2.1，链中含 WebContent 沙箱逃逸与内核提权、页面保护绕过，失陷后注入系统进程驻留；2026 年 8–9 月扩散期曝光的 C2 面板把窃取类目分为沙箱数据（浏览器档案）、Keychain、Wi-Fi、短信、照片、钱包、Cookies。DarkSword 于 2026 年 3 月 18 日披露，面向 iOS 18.4–18.7，公开报道称用 6 个 CVE（含 WebKit 零点击漏洞 CVE-2025-43529、CVE-2025-14174）；取回代码显示其数据模块批量读取 SystemConfiguration 下的 Wi-Fi 配置等系统路径与钥匙串类目；10 月披露的 P7 变体注入 SpringBoard、约 15 秒周期回连 C2。

映射到结构层面，六类能力面最常被作为「出域目标」，逐类给出核对动作：

1. **跨进程 Mach 服务可达性**（mach-lookup）：WebContent 必须连渲染、媒体、通知服务，这张清单是逃逸者眼里的「特权代理目录」。核对：为每个平台进程建立可达服务清单并版本化对比，新增项须评审；服务侧排查「按连接默许」的接口。
2. **系统守护进程 XPC 接口**：Mach 连接建立后的语义层。核对：对涉及凭据、文件、设备控制的守护进程做调用方鉴权与参数校验审查；关注「代取数据」型方法。
3. **IOKit 设备访问**：图形类驱动的用户客户端历史上多次被证明可从受限进程触达并泄漏内核侧信息。核对：清点允许非特权进程打开的驱动，推动收紧匹配谓词。
4. **共享内存与全局命名对象**：名字可猜的共享段是低门槛信息泄漏通道。核对：清点受限进程可 attach 的对象；高风险服务改带鉴权通道。
5. **系统配置的越界可读**：外泄面不止利用链——备份、崩溃日志同样带出受限内容。核对：确认对系统配置目录、他应用容器的默认拒绝锚定完整。
6. **可写容器与临时目录**：立足后的模块暂存与持久化落点。核对：写入面是否严格限自身容器；对自动重载、异常长驻等 Web 投递特征建立网络侧告警。

# 五、结构性弱点与苹果的收敛方向

**弱点**：① profile 数百级、文本 DSL + 正则 + 参数 + import 叠加，跨片段交互难穷尽，几乎不可能形式化验证；② **能力沿 IPC 继承**——客户端受限不代表服务端语义受限，架构上无解，只能服务端逐个补鉴权；③ 功能型不可信进程的能力面不可能清零，「要干活就要 IPC」与「IPC 即移动通道」天然矛盾；④ 私有 entitlement 与平台片段不公开，外部审计只见部分账本；⑤ 旧机型长尾使攻击框架可离线维护（Coruna 公开的偏移量数据库覆盖大量旧版本），收敛速度受制于升级率而非补丁日。

**收敛方向**：一是**把检查点从用户态语义移向内核与硬件**——《Apple 平台安全保护》所示方向：SPTM/可信执行监控类机制重建页表与代码完整性信任根，Tightbeam 类可验证签名把执行校验移到硬件信任路径，syscall 过滤与驱动收紧同属此线。二是**裁剪不可信进程 IPC 面并强化服务端鉴权**：历次在野事件后可观察到对 WebContent 可达清单的收紧与 XPC 鉴权补强；BlastDoor 把不可信数据解析放进极小能力的独立沙箱，是「按能力最小化设计进程」的正面样本。三是**把能力开关产品化**：锁定模式直接关闭 JavaScriptCore JIT——两条链的公开分析均显示其依赖 JIT 相关设施，这等于用户拉下的整类能力闸门，是权限最小化思想在消费端的极端表达。

防御方结论：不用「沙箱挡住了什么」单一问题评估风险，而按「profile 允许面 × entitlement 申请面 × IPC 服务端信任假设 × 硬件信任根版本覆盖」四个变量分别打分。

# 六、能力审计清单：受管设备上的应用与扩展能力面

以下动作限定在授权与不越权前提（只用 MDM 通道、用户同意范围内的备份日志、可得的应用元数据）：

| 审计输入 | 内容 | 通道 |
| --- | --- | --- |
| 描述文件 | 限制载荷、凭证、单 App 模式、内容过滤 | MDM 下发记录与回执 |
| MDM 上报 | 应用清单、权限授权状态、托管配置 | MDM 协议接口 |
| 应用申报 | 用途描述键、隐私清单、后台模式、共享组等签名期能力 | 商店元数据、审批单 |
| 企业配置 | 企业签名应用、扩展授权 | 证书与描述文件台账 |
| 设备遥测 | 崩溃日志、网络元数据（经同意） | 合规采集通道 |

**六步流程**：① 建「设备—应用—申报能力」三表台账；② 按业务基线标注与用途不符的申报；③ 高危人群单独核查能力面与系统版本；④ 把版本长尾当第一风险因子（Coruna 覆盖 13.0–17.2.1、DarkSword 覆盖 18.4–18.7，旧版本应默认视为暴露面）；⑤ 核对共享容器/扩展授权范围，避免一个宽 App Group 串联多应用；⑥ 泄露类攻击的取证窗口常在网络层（投递域名判白、自动重载、beacon 周期），把网络异常与能力台账关联。

**红旗条目**：企业签名应用无源码级审查记录；MDM 自动批准权限弹窗；限制载荷被放宽；同一证书下多应用共享宽容器；旧机型集中且无补丁通道；高危岗位未启用锁定模式且无 Web 投递监测；设备侧出现数据整体带出类异常（备份体积突变、钥匙串访问报错模式异常）；网络侧出现与两类工具包基础设施特征重合的域名注册周期与判白记录。

# 七、结论

- iOS 沙箱是「MACF 框架 + Sandbox 策略模块 + 文本 profile + 装配指定链」的组合；理解 profile 由谁指定，比背规则语法更重要。
- profile 与 entitlement 都应作为**可审计的配置资产**：前者是资源可达白名单，后者是能力申请清单，真实最小权限是两者与 IPC 服务端校验的交集。
- Coruna 与 DarkSword 的失陷后行为表明，攻击者的目的地集中在 Mach 可达性、XPC 语义、IOKit、共享内存与容器写面——恰是评审 profile 时逐项打分的条目。
- 规模、继承、功能依赖、不透明与长尾五个结构性弱点决定沙箱不是防线终点；苹果的收敛是把信任锚移向硬件与内核侧，并以锁定模式做极端裁剪。
- 组织可执行项：能力台账、版本长尾、网络投递监测三者并表管理，红旗按第六节逐项核对。

> **合规声明**：本文仅做沙箱配置结构、entitlement 权限模型与能力最小化的防御性分析，案例信息均来自公开报道与已披露取证结论；不提供沙箱 profile 构造或篡改方法、不提供签名绕过与沙箱逃逸操作指导、不提供针对具体系统版本可用的利用细节，仅适用于授权范围内的安全评估与防护建设场景。

# 参考来源

- [Coruna: Powerful iOS exploit kit — Google Cloud Blog (GTIG)](https://cloud.google.com/blog/topics/threat-intelligence/coruna-powerful-ios-exploit-kit)
- [DarkSword iOS exploit chain — Google Cloud Blog (GTIG)](https://cloud.google.com/blog/topics/threat-intelligence/darksword-ios-exploit-chain)
- [DarkSword — Lookout Blog](https://www.lookout.com/blog/darksword)
- [Coruna iOS 漏洞利用套件 C2 面板泄露（FreeBuf）](https://www.freebuf.com/articles/499174.html)
- [Coruna 与 DarkSword：iOS 高端攻击武器扩散的威胁（安全内参）](https://www.secrss.com/index.php/articles/88939)
- [分析一下最近的 iOS 内核级漏洞利用链 DarkSword（博客园）](https://www.cnblogs.com/websec80/p/22329483)
- [P7 DarkSword iOS Exploit Kit (The Hacker News)](https://thehackernews.com/2026/10/p7-darksword-ios-exploit-kit-adds.html)
- [art-template npm Hijack Delivers iOS Browser Exploit Kit (SafeDep)](https://safedep.io/art-template-npm-supply-chain-compromise)
- [《Apple 平台安全保护》官方指南（中文 PDF）](https://help.apple.com/pdf/security/zh_CN/apple-platform-security-guide-cn.pdf)
- [Apple Developer: App Entitlements 文档](https://developer.apple.com/documentation/bundleresources/entitlements)
- [HackTricks：macOS/iOS 沙箱机制整理](https://hacktricks.wiki/zh/macos-hardening/macos-security-and-privilege-escalation/macos-security-protections/macos-sandbox/index.html)
- [*OS 的安全机制（上）：MACF 与沙箱](https://blog.betamao.me/posts/2024/xos-security-1/)

*报告结束。本报告基于截至 2026-10-28 的公开资料编写，私有 entitlement 键与 profile 结构不随版本公开，建议以官方文档与实测行为为准。*

**咨询iOS系统安全相关问题，请联系 Telegram：https://t.me/one00190**
