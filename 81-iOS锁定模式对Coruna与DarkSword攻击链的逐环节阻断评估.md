# 锁定模式对 Coruna 与 DarkSword 攻击链的逐环节阻断评估——iOS 高防护配置的效果与代价

> **摘要**：Coruna 与 DarkSword 都以水坑网页投递、以 JavaScriptCore/WebKit 内存破坏为入口，经 PAC 绕过、沙箱逃逸、内核提权后落地窃密载荷。本文把两条链拆成七个可评估环节，逐条核对 Apple 官方文档的原文限制，给出"限制—命中环节—阻断强度—残余风险"映射。核心结论：锁定模式的决定性价值在入口环节——禁用 JIT 直接抽走两条链依赖的 JSC 类型混淆原语；对入侵后的提权与外传环节几乎无影响。对已修补设备，及时更新仍是第一道防线，锁定模式是高价值目标的纵深选项。

# 一、官方口径：锁定模式是什么、由谁开启、何时失效

Apple 支持文档《About Lockdown Mode》（105120）的原文定位："an optional, extreme protection that's designed for the very few individuals who... might be personally targeted by some of the most sophisticated digital threats."——可选、极端、面向极少数被定向攻击者；不是全员基线，而是收缩功能面以提高攻击成本的开关。

本文核对以两份官方文档为准：105120（终端用户限制清单，随版本更新）与《Personal Safety》指南章节 ipsd5baf79d0（措辞更技术化，JIT 条款出自此处）。三个常被忽略的口径要点：

1. **逐设备启用**："needs to be turned on separately for your iPhone, iPad, and Mac"；iPhone 开启后配对 Apple Watch 自动开启。家庭共享 Apple ID 时，未开启的设备就是旁路。
2. **例外即撕口子**：个人安全指南原文称 JIT 禁用"unless the user excludes a trusted site"——被排除网站恢复 JIT；App、联系人例外同理。
3. **MDM 不可远程配置**："Lockdown Mode is not a configurable option for Mobile Device Management"，只能由使用者本人开关并重启。

与网传"7 类清单"的两处措辞差异须如实标注：其一，"共享 Apple 日历"在现行 105120 中**无独立条目**，日历共享邀请归入"Apple services（服务邀请）"总条款；其二，蜂窝方面官方逐字表述只有"2G and 3G cellular support is turned off"，所谓"数据切换"条款不存在。评估以原文为准，不采纳媒体转述。

# 二、把两条攻击链拆成七个可评估环节

依据 GTIG 与 iVerify 公开报道的环节划分（口径分歧见 2.1）：

| 环节 | Coruna（iOS 13.0—17.2.1，5 链 23 漏洞） | DarkSword（iOS 18.4—18.7，6 漏洞全链） | 依赖的系统能力 |
| --- | --- | --- | --- |
| 1 投递 | 水坑站与被入侵站隐藏 iframe；UNC6691 仿冒加密货币站海量诱导 | 水坑为主，波及沙特、土耳其、马来西亚、乌克兰 | Safari drive-by 解析；消息链接与预览渲染 |
| 2 Web 入口 | 按版本加载 terrorbird（CVE-2023-43000）/cassowary（CVE-2024-23222）等 JSC 内存破坏 | CVE-2025-31277（JSC JIT 类型混淆）或 CVE-2025-43529（DFG JIT 垃圾回收缺陷） | JSC JIT 编译路径与类型系统 |
| 3 PAC 绕过 | WebKit RCE 与沙箱逃逸之间的 ARM64e PAC 绕过模块 | dyld 用户态 PAC 绕过（CVE-2026-20700，iOS 26.3 修复） | 指针认证实现、dyld 加载路径 |
| 4 沙箱逃逸 | IronLoader（CVE-2023-32409）从 WebContent 出域 | ANGLE/WebGL 触发 GPU 进程越界（CVE-2025-14174），再借 XNU 写时复制缺陷进 mediaplaybackd（CVE-2025-43510） | WebContent 可达的 GPU/媒体系统服务、Mach IPC |
| 5 内核提权 | Photon（CVE-2023-32434）、Parallax（CVE-2023-41974），Gallium/Sparrow/Rocket 绕 PPL | XNU VFS 竞态（CVE-2025-43520）构建内核读写 | 内核驱动与文件系统路径 |
| 6 载荷落地 | PlasmaLoader 注入 root 进程 powerd，伪装 com.apple.assistd | GHOSTKNIFE/GHOSTSABER/GHOSTBLADE，可删崩溃日志；P7 变体被报道与 Coruna 组件组合 | 内核读写后的任务操控；内存驻留 |
| 7 数据外传 | 扫描钱包助记词/私钥、备忘录与短信关键词，HTTPS C2 回传 | 窃取消息、钥匙串、位置、照片与钱包数据 | 出网通道、用户数据容器访问 |

**2.1 机构口径分歧**：GTIG 把 DarkSword 定义为"6 漏洞全链"；2026 年 8 月 parkpeark.xyz 事件中，MyCERT 将其归类"钓鱼站"，而第三方 CTI 与媒体认为是水坑投递、属于低报；Censys/The Hacker News 则把 Coruna 描述为与 DarkSword 同生态分发的"配套载荷工具包"。分歧不改变七环节模型，但投递与持久化环节的归因各家不一。

# 三、Apple 官方限制逐条核对（原文措辞）

105120 现行版本的限制条目共 9 类，逐字核对如下（关键条目保留英文原文摘录）：

| # | 条目 | 原文要点（摘录） | 中文口径 |
| --- | --- | --- | --- |
| 1 | Web browsing | "Certain complex web technologies are blocked"；个人安全指南："like just-in-time (JIT) JavaScript compilation... are disabled unless the user excludes a trusted site" | 复杂网页技术封锁；JIT 禁用；网页字体/图像降级 |
| 2 | Messages | "Most message attachment types are blocked, other than certain images, video, and audio... links and link previews are unavailable" | 图像/视频/音频外的多数附件被阻止；链接与预览不可用 |
| 3 | FaceTime | "blocked unless you have previously called that person... within the past 30 days" | 30 天未联系者来电被阻止；SharePlay/实况照片不可用 |
| 4 | Apple services | "Incoming invitations for Apple services... are blocked unless you have previously invited that person" | 陌生对象的 Apple 服务邀请被阻止（日历类邀请归此条） |
| 5 | Device connections | "the device needs to be unlocked" | 连接配件/电脑必须解锁 |
| 6 | Configuration profiles | "can't be installed, and the device can't be enrolled in Mobile Device Management or device supervision" | 锁定期间禁止新装描述文件与 MDM 注册；此前已有的 MDM 描述文件保留（个人安全指南补充） |
| 7 | Wireless connectivity | "won't automatically join non-secure Wi-Fi... 2G and 3G cellular support is turned off" | 不加入并断开不安全 Wi-Fi；关 2G/3G |
| 8 | Apps | "apps on iOS and iPadOS are strictly limited... some experiences are completely unavailable" | App 内网页内容受限 |
| 9 | Photos | 共享剥离位置、共享相簿移除 | 元数据与共享面收缩 |

现行官方文本中**不存在**的常见误传：禁止 OTA 更新（旧版曾限制、现已无此条）、单列 WebAssembly 条款、"蜂窝数据切换"条款。

# 四、映射表：限制 → 命中环节 → 阻断强度 → 残余风险

| 锁定模式限制 | 命中环节 | 阻断强度 | 残余风险 |
| --- | --- | --- | --- |
| 禁用 JSC JIT（条 1） | 环节 2：两条链的 JSC 类型混淆/DFG GC 原语均依赖 JIT | **完全阻断**（前提：无例外且利用依赖 JIT） | 非 JIT 依赖的 WebKit/解析器缺陷仍可达；加网站例外即恢复 |
| 复杂网页技术整体封锁（条 1） | 环节 2 部分；环节 4 的 WebGL/ANGLE 触发面 | **提高成本**（封锁粒度官方未逐字列举） | 攻击者可实测筛出仍放行的接口作新入口 |
| 消息附件与链接预览封锁（条 2） | 环节 1：消息投递、预览触发 | **提高成本/部分阻断** | 水坑投递不经消息通道；图像/视频/音频解析面被官方保留 |
| 可疑 FaceTime 阻止（条 3） | 未直接命中两条链 | **无影响** | 社工诱导入链不受限 |
| Apple 服务邀请封锁（条 4） | 环节 1 替代投递（垃圾邀请诱导） | **部分阻断** | "日历邀请"靠总条款解释，非独立承诺 |
| 有线连接需解锁（条 5） | 环节 6/7 的物理配件路径 | **完全阻断（仅有线/近距场景）** | 解锁瞬间仍可信配件；对远程链无影响 |
| 描述文件/MDM 注册禁止（条 6） | 环节 6 替代持久化（恶意描述文件） | **完全阻断（新安装）** | 存量恶意描述文件不清除；可被社工诱导自行关闭后安装 |
| 不安全 Wi-Fi 与 2G/3G 关闭（条 7） | 环节 1 的降级网络/伪基站注入 | **提高成本** | HTTPS C2 外传完全不受影响 |

**为何入口阻断是关键**：两条链的共同结构是"纯 JS 框架先做设备指纹，再按版本加载 JSC 利用模块"。JSC 类型混淆利用的工程前提是诱导 JIT 编译器对对象布局做错误推断——parkpeark.xyz 事件的 CTI 分析称之为"JIT warm-up"，并得出"锁定模式打掉攻击链关键前提"的结论；禁用 JIT 后，解释执行路径不具备 DFG/FTL 优化层的类型推断行为，公开版本的 terrorbird/cassowary/CVE-2025-31277/43529 难以复现原语。这是**移除依赖项**而非降低概率，故评"完全阻断"；须 hedge 的是：Apple 未公开封锁接口清单，"完全"仅对依赖 JIT 的利用成立，非 JIT 入口会退化为"提高成本"。

**对已修补设备影响有限**：两条链的漏洞已全部修复——Coruna 相关 CVE 甚至回补到 iOS 15.8.7、16.7.15，DarkSword 最后一环 CVE-2026-20700 在 iOS 26.3 关闭。补丁拆完每一环后，锁定模式只是对未来同型零日的冗余保险；环节 3—7（PAC、逃逸、内核、驻留、外传）不因锁定模式增加障碍，只能靠补丁与架构演进（PPL/SPTM/TXM、MIE）解决。

# 五、代价、误用场景与企业受管环境

**受管设备与锁定模式的结构性张力**：锁定期间新描述文件无法安装、无法注册 MDM/监督（须关闭后安装再重新开启）；已有 MDM 描述文件保留；MDM 无法远程开关它。现实含义：依赖描述文件的入职流程被打断；员工可自行开关使企业侧出现治理盲区；高敏岗位应改用"专用受管设备+专用个人设备"分离。恶意描述文件持久化被阻断对安全是收益、对运维是成本。

**被打断的工作流**：依赖 JIT/复杂网页技术的 Web 应用（在线办公套件、协作白板、WASM 类应用）加载变慢或不可用；与陌生对象的 FaceTime 面谈/远程采访被拦；SharePlay 类协作消失；弱覆盖与旧制式漫游地区受 2G/3G 关闭影响（官方未承诺覆盖兜底）；与电脑配对、有线配件需保持解锁。多数影响可加例外缓解，但每个例外都是撕开的防护口，例外清单应纳入风险审计。

**四类常见误用**：其一，"开了锁定模式=不用更新"——它只压 Web 入口，不修内核，正确顺序永远是先补丁、再锁定、再减例外；其二，例外蔓延——水坑站常模仿被例外的高信任域名；其三，设备缺口——只开 iPhone，iPad/Mac 裸奔；其四，把锁定模式当"未被入侵"的证据——它是配置状态，不是遥测。

# 六、分级建议

| 人群 | 建议 | 依据 |
| --- | --- | --- |
| 记者、活动人士、异见者及其家属 | 全部个人设备默认开启；例外按"天"临时启用并事后清理 | 商业监控重点客群；两条链的水坑投递即面向此类目标 |
| 大额/自托管加密资产持有者 | 开启；助记词操作放独立离线设备 | PlasmaLoader 与 P7 变体报道均显示钱包/助记词是首要目标；UNC6691 波次约 4.2 万台设备 |
| 政府、关键基础设施、涉密商务高管 | 开启+账号分离+硬件密钥 | 定向监控采购的常规目标画像 |
| 企业（针对上述岗位） | 不试图经 MDM 干预（技术不可行），发专用高风险设备 | 第五节的结构性张力 |
| 普通用户 | 不必开启；保持自动更新与最新版、关闭锁屏消息预览并过滤未知发件人、不点陌生链接/附件 | Apple 官方强调"绝大多数人不会被定向攻击"；补丁覆盖率远比功能收缩划算 |

# 七、结论

**核心判断**：锁定模式对 Coruna/DarkSword 的阻断是"单点决定性"的——它不干预链条后半段，只在环节 2 移除了 JIT 依赖项，附带在环节 1 提高投递成本；入口断，则 3—7 环节不可达。这与系列综合报告"已更新并启用锁定模式的高风险用户，风险降为低—中"的口径一致：降风险的主力仍是补丁，锁定模式是对未来同型入口零日的保险。

**可验证自查清单**：

1. 设置 → 隐私与安全性 → 锁定模式：确认 iPhone、iPad、Mac **分别**开启（配对 Watch 随 iPhone）；
2. 同一页面核查三类例外（网站/App/联系人），无法回忆添加理由者一律删除——被排除网站即恢复 JIT；
3. 设置 → 通用 → "VPN 与设备管理"：确认无未知描述文件（锁定模式只拦新装、不清存量）；
4. 关于本机核对版本并对照 Apple 安全发布列表（100100）：DarkSword 修复线为 iOS 18.7.3/26.1 以上、CVE-2026-20700 需 26.3；Coruna 老设备需 15.8.7 或 16.7.15；无法达线的老机型应换机；
5. 行为代理抽查：陌生 FaceTime 是否被拦、陌生短信链接预览是否失效——异常说明锁定模式被例外或被关过；
6. 预期管理：被阻断后的替代路径已知存在——非 JIT 的 WebKit/媒体解析缺陷（图像/视频/音频附件面被官方保留）、诱骗用户自行关闭或加例外的社工、家庭内未加固设备旁路、npm 供应链注入网页资源（2026 年 5 月 art-template 事件）、转向应用安装与消息通道。锁定模式改变的是成本分布，不是攻击意愿。

**一句话结论**：高价值目标的可行组合是"及时更新+锁定模式+零例外+账号设备分离"；普通用户保持补丁前沿，就是最好的锁定模式。

> **合规声明**：本文仅基于公开资料做防御性研究与配置建议，不提供利用代码、实现细节或锁定模式绕过方法。环节划分与 CVE 归属以各机构公开报告为准，口径分歧已并列呈现；具体处置以 Apple 官方安全公告与授权机构取证结论为准。

# 参考来源

- Apple Support：[About Lockdown Mode](https://support.apple.com/en-us/105120)
- Apple Personal Safety Guide：[Harden your devices with Lockdown Mode](https://support.apple.com/en-ae/guide/personal-safety/ipsd5baf79d0/web)
- Google Threat Intelligence Group：[Coruna: The Mysterious Journey of a Powerful iOS Exploit Kit](https://cloud.google.com/blog/topics/threat-intelligence/coruna-powerful-ios-exploit-kit)
- Google Threat Intelligence Group：[The Proliferation of DarkSword: iOS Exploit Chain Adopted by Multiple Threat Actors](https://cloud.google.com/blog/topics/threat-intelligence/darksword-ios-exploit-chain)
- iVerify：[Inside DarkSword: A New iOS Exploit Kit Delivered Via Watering Hole Attacks](https://www.iverify.com/blog/darksword-ios-exploit-kit-explained)
- The Hacker News：[P7 DarkSword iOS Exploit Kit Adds Crypto Wallet Data Theft and Remote Commands](https://thehackernews.com/2026/10/p7-darksword-ios-exploit-kit-adds.html)
- Socket：[Coruna Respawned: Compromised art-template npm Package Leads to iOS Browser Exploit Kit](https://socket.dev/blog/coruna-respawned-compromised-art-template-npm-package)
- FreeBuf：[Coruna iOS漏洞利用套件C2面板泄露：Safari打开就失陷](https://www.freebuf.com/articles/499174.html)
- Apple Support：[Apple security releases](https://support.apple.com/en-us/100100)
- Apple Support：[About the security content of iOS 15.8.7 and iPadOS 15.8.7](https://support.apple.com/en-us/126632)
- Apple Support：[About the security content of iOS 16.7.15 and iPadOS 16.7.15](https://support.apple.com/en-us/126646)
- Apple Support：[About the security content of iOS 26.3 and iPadOS 26.3](https://support.apple.com/en-us/126346)

*报告结束。本报告基于截至 2026-10-28 的公开资料编写，锁定模式的限制清单随系统版本调整，建议每次大版本发布后复核。*

**咨询iOS系统安全相关问题，请联系 Telegram：https://t.me/one00190**
