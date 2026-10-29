# iOS 内核信息泄露原语分析——从 Coruna 与 DarkSword 看地址随机化的攻防

> **摘要**：现代 iOS 在野利用链的成败，往往不取决于"能改内存"，而取决于"能看见地址"。本文以 2026 年公开披露的 Coruna 与 DarkSword 两条真实攻击链为样本，梳理"读原语→信息泄露→布局重建→控制流劫持"的依赖链条，盘点 iOS/XNU 侧的随机化与地址隐藏手段（内核 slide、ASLR、JIT 区域随机化、SPTM/TXM、PAC、MIE/EMTE、BlastDoor 与锁定模式），区分官方文档化机制与社区推断；再把泄露原语按危害分为用户态对象地址、JIT 页地址、内核 slide、指针语义（PAC 相关）四层并评估缓解强度，最后给出端点侧低误报狩猎信号。全文只做机制分析，不含可用利用步骤。

# 一、为什么信息泄露是现代利用链的前置原语

一个可写、可执行但"地址全盲"的漏洞，在当代 iOS 上几乎无法转化为稳定控制流劫持。原因在于攻击者面对的是一组相互独立的随机化域：内核映像整体带随机 slide 加载，用户态二进制与共享缓存按 ASLR 布局，JSC 的 JIT 可执行页在每次进程启动时位置不同，内核对象散布于不可预测的堆区。控制流劫持要求跳转目标是一个**具体数值**，而上述每一个数值在未泄露前都不可知。

因此公开研究把利用链刻画为一条依赖链：**读原语（information leak）→ 泄露某个基址或对象指针 → 重建布局（把"未知地址"换算成"基址+偏移"）→ 才谈得上定向写与跳转**。缺少第一环时，攻击者只剩两种低效选择：面向单一猜测值的爆破（在 PAC 与 Jetsam/看门狗约束下代价极高），或用大面积页面喷洒换取概率命中——所谓 JIT 喷洒即由此而来，也正是缓解措施针对性消灭的对象。泄露的价值随层级递增：泄露一个用户态字符串对象地址，只能支撑同一进程内的定向改写；泄露 JIT 页基址，可执行代码布局才可预测；泄露内核 slide，才能把内核态漏洞从"随机偏移破坏"升级为"精确指向符号化例程"。

需要强调：本篇只论证这条依赖关系的机制逻辑，不讨论任何具体泄露构造方法；类型混淆利用细节与 PAC 算法本身，详见系列《JavaScriptCore 类型混淆原理篇》与《PAC 绕过技术演进篇》。

# 二、iOS/XNU 的随机化与地址隐藏工具箱

下表把 Apple 侧机制按"是否官方文档化"分列。凡标注"社区推断"者，指依据开源 XNU 代码与公开研究（如 Project Zero 的 Mac 内核系列、公开议题演讲）形成的共识，Apple 并未将其作为对外承诺的安全属性逐一成文。

| 机制 | 隐藏/打乱了什么 | 文档化状态 |
| --- | --- | --- |
| 内核 KASLR / slide | 内核 Mach-O 整体加载偏移，每次启动随机 | XNU 开源实现与社区研究一致确认；Apple 官方指南未单独成文 |
| ASLR（用户态） | 可执行文件、dylib、共享缓存基址与堆起始区 | 历史版 Apple 官方安全指南有明确描述，属官方机制 |
| JIT 可执行区随机化 | JSC 生成的可执行页位置随进程变化 | WebKit 开源实现+社区推断；无官方参数文档 |
| 堆布局隔离（分配器分区、Gigacage 等） | 同类对象不必然相邻，跨区不可直接溢出 | WebKit/bmalloc 开源代码可查，属工程事实而非安全承诺 |
| 内核符号对非特权进程隐藏 | sysctl、崩溃日志、mach 接口不向普通进程暴露内核符号/地址 | 行为可由开源代码与端点遥测验证；官方以"纵深缓解"表述 |
| PPL → SPTM/TXM | 页表完整性；A15/M2 起 SPTM+TXM 取代 PPL，官方明确其威胁模型覆盖"内核可写且绕过控制流保护"的情形 | 官方文档（Apple 平台安全指南"操作系统完整性"） |
| PAC | 指针高位签名，验签失败即异常中止 | 官方文档 |
| MIE/EMTE | 硬件辅助的类型化内存安全，A19/M5 起提供 | 官方文档+Apple 安全研究博客（2025-09） |
| BlastDoor、锁定模式 | 收窄消息面攻击面/大幅削减 WebContent 能力 | 官方文档 |
| Tightbeam（WebContent 可见 IOKit 服务收窄） | 从 IPC 层限制浏览器进程能枚举/连接的系统服务 | 主要为社区转述；本次检索未见对应官方文档条目，按转述对待 |

两点提醒。其一，随机化与"隐藏接口"是乘法关系：任何一处未加固的读通道（驱动返回未初始化内存、日志残留内核指针）都能让整套随机化归零。其二，Coruna 与 DarkSword 同时面向 iOS 13–17 与 18 两代随机化水位，其分版本适配本身就说明：**泄露原语的有效性是分版本、分硅片的属性，而非一劳永逸**。

# 三、Coruna：公开报道中的泄露面与未确认项

综合 Google GTIG、iVerify、Kaspersky Securelist、Cloud Security Alliance 与维基条目，Coruna 的可靠画像如下：

| 维度 | 公开报道内容 | 来源类型 |
| --- | --- | --- |
| 性质 | 纯 JavaScript 利用工具包，"23 个利用模块、5 条完整利用链"，覆盖 iOS 13.0–17.2.1 | GTIG/iVerify 披露 |
| 时间线 | 2025 年 2 月前后被发现流传，2025 年中见于针对乌克兰的钓鱼网站投放，2025 年末转向涉加密货币的规模金融犯罪，2026-03-03 正式披露 | GTIG/CSA/Wikipedia |
| 泄露面线索 | 阶段性描述为"指纹识别→WebKit RCE→ASLR/PAC 绕过→沙箱逃逸→载荷落地"，即报道承认链中存在 **ASLR/PAC 绕过环节** | THN/GTIG 转述 |
| CVE 构成 | 报道列出 CVE-2024-23222、CVE-2023-43000、CVE-2023-41974、CVE-2023-43010、CVE-2023-32434（内部代号 Photon）、CVE-2023-38606（代号 Gallium）、CVE-2023-32409、CVE-2023-48503、CVE-2021-30952 等跨 2021–2024 年的补丁期漏洞 | CSA/Securelist |
| 归因与流转 | 与美国政府承包商技术外流关联（Peter Williams 案，归因存在争议）；Securelist 将其与 Operation Triangulation 的 iOS 组件更新相联 | 公开报道 |
| 规避策略 | 入口阶段通过 IndexedDB、MathML 等行为特征探测锁定模式并绕开此类设备 | iVerify |

必须坦率区分：**"链中存在 ASLR/PAC 绕过环节"是报道事实；该环节具体采用越界读、未初始化内存读还是类型混淆带来的指针暴露，公开材料未确认**。社区常把 CVE-2023-32409 转述为可造成内存内容泄露的 WebKit 问题，但本次检索未能核验 Apple 公告原文措辞，按"未确认"处理。Coruna 按 iOS 版本族预烘焙五套内核代码这一事实本身就反证：每条链都必须为对应版本的随机化水位各自准备泄露方案。

# 四、DarkSword：公开报道中的泄露面与未确认项

DarkSword 于 2026-03-18 由 Google GTIG 联合 Lookout、iVerify 披露，经被攻陷的合法网站（含乌克兰资产）投放，影响 iOS 18.4 至 18.7.x/26.2 一线，修复见 iOS 26.3/26.3.1 与 18.7.5+。CSA 与 iVerify 给出的六漏洞构成如下：

| CVE | 组件（报道口径） | 报道给出的缺陷类别 | 在链中的环节 |
| --- | --- | --- | --- |
| CVE-2025-31277 | JavaScriptCore | 类型混淆 | WebKit/JSC 入口 |
| CVE-2025-43529 | JavaScriptCore/WebKit | 释放后使用 | 对象生命周期破坏 |
| CVE-2025-14174 | ANGLE（GPU 渲染栈） | 越界写 | GPU 环节 |
| CVE-2026-20700 | dyld | 内存破坏（PAC 绕过经由 dyld 结构） | PAC/控制流环节；Apple 公告措辞为"针对特定目标个体的极复杂攻击"，属官方确认在野利用 |
| CVE-2025-43510 | 内核 | iVerify 描述与写时复制（CoW）语义相关 | 内核提权 |
| CVE-2025-43520 | 内核 | 未披露细节 | 内核提权 |

CSA 将阶段概括为 WebKit→JIT→GPU→PAC 绕过→内核，载荷即公开报道中的 GHOSTBLADE/KNIFE/SABER 家族，目标是钥匙串、加密钱包、消息与定位数据，纯 JS、无落盘。就泄露原语而言：一条需要 JIT 与 GPU 子系统协同、并以 dyld 结构承载 PAC 绕过的链，必然以**进程内地址知识**为前提——类型混淆与 UAF 在缺陷类别上天然可能同时提供"读"的能力（对象指针被错误类型解释即为语义泄露），这是机制层推断，**具体泄露构造公开材料未确认**。把机制推断写成报道事实，正是这类题材最常见的失实点。

# 五、泄露原语的危害分层

| 层级 | 泄露使攻击者确定什么 | 对应缓解与强度评价 |
| --- | --- | --- |
| L1 用户态对象地址 | JSC 堆上对象/缓冲区位置，支撑同进程定向读写 | ASLR+bmalloc 分区+Gigacage；只能提高成本，进程内读漏洞即破防；强度：低 |
| L2 JIT 页地址 | 可执行代码基址，跳转/返回目标可算 | W^X、页粒度执行权限、对喷量触发看门狗；强度：中 |
| L3 内核 slide | 内核符号相对基址的绝对地址，内核写漏洞由盲打变精打 | KASLR+符号/接口隐藏+崩溃日志脱敏；SPTM/TXM 后"内核可写"本身不再等于接管页表；强度：中→高（随 A15/M2 分层） |
| L4 指针语义/PAC 相关结构位置 | 可签名目标的分布、dyld/系统结构位置，为 PAC 绕过铺路 | PAC 验签失败即中止；但签名对象若被泄露/可预测（如 DarkSword 报道中的 dyld 结构路径），L4 即被消费；强度：高但依赖不泄露签名上下文 |

规律是清晰的：**缓解强度与泄露层级赛跑**。L1/L2 在现代 iOS 上早已"破防常态化"，分水岭在 L3/L4——这也是两条链都把资源砸在内核与 dyld 环节、而 SPTM/TXM 与 MIE/EMTE 路线试图让"即使 L3/L4 泄露并被利用也不再等于完全接管"的原因。

# 六、防御侧与检测侧：让泄露贬值，让链现形

**6.1 直接削弱泄露价值的架构升级**

- MIE/EMTE（A19/M5 起）：硬件标签+类型化分配，使越界读同样落入标签校验范围，泄露与破坏共享同一失效面；官方路线，见 Apple 安全研究博客。
- SPTM/TXM：官方文档明确其设计覆盖"攻击者已具备内核写并可绕过控制流保护"的场景，相当于把 L3 泄露的收益封顶——内核失守不再自动获得页表与代码完整性控制权。
- GPU/Mach-IPC 隔离与 BlastDoor、锁定模式：缩小 WebContent 能观察、能连接的对象池，L1/L2 之外的"跨界读"机会随之减少（Tightbeam 相关表述按社区转述对待）。
- 分配器与接口加固：崩溃日志脱敏、遥测不含内核指针、驱动禁止向非特权调用方返回未初始化内存——堵住的都是"合法读通道"。
- 越界即崩策略：PAC 验签失败即 abort 是官方行为。它是双刃剑：对攻击者是反馈信号（崩溃边界即谓词 oracle），对用户是告警疲劳源，对检测方则是高价值遥测——收益取决于谁先读懂崩溃数据。

**6.2 给端点检测与加固工程师的狩猎信号**

一个 WebContent 进程若出现下列组合，应视为"利用链正在重建地址布局"的强信号，而非普通页面故障：其一，短时间**异常大量 JIT 可执行页分配**（喷洒/预占的共性痕迹）；其二，同一站点访问内**反复的 WebContent 中止-重载循环**，呈现"每次崩溃后立即重试"的迭代节律——这是链在用崩溃当反馈通道；其三，页面加载后先出现一小段**目标探测流量**（能力探测/水印校验类请求），随后才进入大负载脚本阶段，Coruna 报道中的锁定模式探测行为即属此类；其四，同一网络出口下**跨 iOS 小版本的分散命中**——工具包按版本选链的指纹。低误报做法是把上述信号做成组合谓词而非单条规则：JIT 突发+中止循环+无对应交互时长三者同现才告警，并用企业 MDM 的设备版本画像校正基线。加固结论则简单得多：iOS 13–17.2.1 与 18.4–18.7 两代设备分别对应 Coruna 与 DarkSword 的打击面，无法升级的旧设备应纳入强隔离处置，高价值账户开启锁定模式——两条链均被报道会检查并绕开锁定模式设备，恰说明该开关仍改变攻击经济学。

# 七、结论

Coruna 与 DarkSword 的公开材料再次验证了一条老规律：地址随机化从不直接拒绝攻击，它只是把攻击的成本从"写一个漏洞"抬高到"维护一条泄露管线"；而工具包的流通与复用（泄露方案随代码一起外流、被不同团伙继承）说明，一旦某个版本的泄露方案被烘焙进 kit，随机化对该版本的红利即刻清零。因此防御重心必须从"再加一层随机化"转向"让被泄露的坐标失去裁决权"：SPTM/TXM、MIE/EMTE 与锁定模式正是这条路线——它们不阻止读，而是让读完之后仍然写不动页表、越不出类型域。对检测方，泄露原语是链中最难伪装的一环：没有地址就没有定向写，而"获取地址"的行为本身就是最稳定的狩猎面。建议按季度复核：本报告事实截至 2026-10-28，两条链各 CVE 的泄露构造细节在公开层面仍是空白，若后续有厂商或 Project Zero 级分析补位，第五节的案例归属需重估。

> **合规声明**：本文仅提供 iOS/XNU 安全机制的分析、公开报道的事实梳理与防御/检测视角的建议，不包含任何可用的利用代码、内存布局推导脚本、泄露构造步骤或绕过方法；对公开材料未确认的技术细节均已明确标注，读者不应将本文的机制推断当作对特定漏洞能力的证实。

# 参考来源

- Google Threat Intelligence, "Coruna: The Mysterious Journey of a Powerful iOS Exploit Kit" — https://cloud.google.com/blog/topics/threat-intelligence/coruna-powerful-ios-exploit-kit
- Google Threat Intelligence, "DarkSword iOS Exploit Chain" — https://cloud.google.com/blog/topics/threat-intelligence/darksword-ios-exploit-chain
- iVerify, "Coruna: Inside the Nation-State-Grade iOS Exploit Kit We've Been Tracking" — https://iverify.io/blog/coruna-inside-the-nation-state-grade-ios-exploit-kit-we-ve-been-tracking
- iVerify, "Inside DarkSword: A New iOS Exploit Kit Delivered Via Compromised Legitimate Websites" — https://iverify.io/blog/darksword-ios-exploit-kit-explained
- Kaspersky Securelist, "Coruna framework: an exploit kit and ties to Operation Triangulation" — https://securelist.com/coruna-framework-updated-operation-triangulation-exploit/119228/
- Cloud Security Alliance, "Coruna iOS Exploit Kit: Multichain Research Note (2026-03-13)" — https://labs.cloudsecurityalliance.org/research/csa-research-note-coruna-ios-exploit-kit-multichain-20260313/
- Cloud Security Alliance, "Full-Chain iOS Zero-Day Exploitation by State Actors (DarkSword)" — https://labs.cloudsecurityalliance.org/research/csa-research-note-darksword-ios-fullchain-zeroday-multiactor/
- The Hacker News, "Coruna iOS Exploit Kit Uses 23 Exploits Across Five Chains Targeting iOS 13–17.2.1" — http://thehackernews.com/2026/03/coruna-ios-exploit-kit-uses-23-exploits.html
- Wikipedia, "Coruna (exploit kit)" — https://en.wikipedia.org/wiki/Coruna_(exploit_kit)
- Apple Platform Security Guide, "Operating system integrity"（SPTM/TXM/PAC/MIE） — https://support.apple.com/guide/security/operating-system-integrity-sec8b776536b/web
- Apple Security Research Blog, "Memory Integrity Enforcement: A complete vision for memory safety in Apple devices" — https://security.apple.com/blog/memory-integrity-enforcement/
- Apple Support, "iOS 26.3 Security Content"（含 CVE-2026-20700） — https://support.apple.com/en-us/126346

*报告结束。本报告基于截至 2026-10-28 的公开资料编写，内核随机化与缓解措施随版本变化较快，建议按季度复核。*

**咨询iOS系统安全相关问题，请联系 Telegram：https://t.me/one00190**
