# 企业移动安全视角下的 Coruna 与 DarkSword 防护——MDM 策略、MTD 能力边界与补丁 SLA 设计

> **摘要**：Coruna 与 DarkSword 都是打开网页即中招的纯 JS 利用套件，而企业在 iOS 上能拿到的控制面远比厂商口径窄：MDM/DDM 只给策略开关、网络出口与账号入口，不给内存可见性与跨 App 读取权。本篇把受管与 BYOD 比例换算成防护上限，给出可落地的策略语义，澄清锁定模式与 MDM 的冲突边界，按平台允许能力对齐 MTD 采购判断，按人群分层设计补丁 SLA 与长尾处置。判断：硬防线只有版本收敛与入口抑制，其余皆为补偿与取证。

# 一、资产盘点：受管比例决定防护上限

iOS 上"受管"不是布尔值：监督设备可用完整 Restrictions 载荷、主屏幕布局、BuiltIn 过滤器允许清单与远程触发更新，是唯一能默认拒绝未知域名的形态；BYOD 用户注册只有 Apple 标注可用于用户注册的键生效，管不了 Safari 个人浏览；未注册设备只剩网络出口与账号条件访问。上限可写成：**有效防护 = 受管占比 × 受管面键覆盖率 × (1 − 例外漏出率)**，失守点往往不在缺策略，而在分母里那三到六成未注册设备。

| 控制面 | 挡住链路哪一环 | 挡不住什么 |
| --- | --- | --- |
| MDM/DDM 策略载荷 | 执行前提：版本压到已收口分支后，Coruna 5 条链（iOS 13.0–17.2.1）与 DarkSword 6 个 CVE（含 CVE-2025-14174 ANGLE/GPU 越界写、CVE-2026-20700 dyld 侧 PAC 绕过）同时失效 | 零日窗口；已到版本上限的老机型 |
| 托管 App 配置 | 外泄与二次扩散：企业 App 内的复制、分享、附件预览、离线缓存 | Safari 与个人区；键语义由 App 厂商定义 |
| 代理与 Web 内容过滤器 | 投递：水坑页、隐藏 iframe、标签页自动重载的域名来源 | 站点被攻陷后的同源脚本；不做 TLS 检查时内容层不可见 |
| Network Extension 集中配置 | 外泄与 C2：蜂窝与漫游出口可观测、可拒绝 | 内存驻留后门的存在性证明 |
| 受限功能开关 | 侧移与凭据半径：AirDrop、配对、USB、NFC、截图、AutoFill | 纯远程 JS 投递本身 |
| 托管主屏幕与 App 白名单 | 二次感染：来路 App、被劫持的内部签名分发 | 白名单内 App 自身的漏洞 |
| Safari 内容过滤与扩展部署 | 取证回溯：能否还原谁开过哪个域名 | 攻击执行本身，属事后 |

# 二、高价值策略基线：落到策略语义

键名取自 Apple 设备管理文档的 Restrictions（`com.apple.applicationaccess`）与 Web 内容过滤载荷；平台没有对应键的，语义列直说做不到。

| 策略项 | 载荷与键 | 门槛 | 语义要点 |
| --- | --- | --- | --- |
| 更新窗口与最低版本 | `forceDelayedSoftwareUpdates`、`enforcedSoftwareUpdateDelay`（天）、`forceDelayedAppSoftwareUpdates` | 监督；最长延迟 90 天；iOS 26+ 废弃、iOS 27+ 移除，迁 DDM `com.apple.configuration.softwareupdate.settings` | 延迟只管更新对用户可见，不约束远程安装命令；无强制最低版本的键，只能靠符合性判定加访问降级 |
| 后台安全改进 | `allowRapidSecurityResponseInstallation`、`allowRapidSecurityResponseRemoval`（文档现名 Background Security Improvements） | 监督，iOS 16+ | 不要下发禁止：WebKit/JSC 修复常以不重启方式落地 |
| App 安装面 | `allowAppInstallation=false`（MDM 命令可覆盖）、`allowWebDistributionAppInstallation=false`、`allowMarketplaceAppInstallation=false`、`allowEnterpriseAppTrust` | 监督 | 阻断自行安装来路 App，压制被劫持的内部签名分发 |
| 自动填充与钥匙串 | `allowPasswordAutoFill=false`、`safariAllowAutoFill=false`、`forceAuthenticationBeforeAutoFill=true`、`allowCloudKeychainSync=false`、`allowPasswordSharing=false`、`allowPasswordProximityRequests=false` | 监督，iOS 12+/13+ | 两个 AutoFill 键须成对下发，单下一个留缺口；关 iCloud 同步意味着成员无法把凭据带到个人设备，需以受管密码管理器承接，代价写进制度；收益是单台失陷的凭据半径最小化 |
| 截图与数据方向 | 截图限制键（同时禁用截图与屏幕录制）、`allowOpenFromManagedToUnmanaged=false`、`allowOpenFromUnmanagedToManaged=false`、`allowManagedAppsCloudSync=false` | 前三项可用于用户注册 | 剪贴板没有全局键，只能做托管与非托管的交换方向控制，加受管 App 自身配置 |
| Web 内容过滤 | `com.apple.webcontent-filter`：`PermittedURLs`（`FilterType=BuiltIn` 下即默认拒绝）、`DenyListURLs`（限不超过 500 条）、`FilterBrowsers`（本场景必须为真）、`FilterSockets`、`ContentFilterUUID`、`SafariHistoryRetentionEnabled`（iOS 26+） | 允许清单仅限 BuiltIn | 默认拒绝可实现，代价是全量例外管理，只推荐高风险人群；全员宜拒绝清单加新域名限速 |
| 配件与 AirDrop | `allowAirDrop=false`、`allowHostPairing=false`（无监督主机证书时关闭全部配对）、`allowUSBRestrictedMode`、`allowNFC=false` | 监督 | 呼应收割 Wi-Fi 配置与配件路径 |
| 消息附件与例外名单 | 附件预览无 Apple 全局键；例外名单属组织治理 | — | 附件面靠锁定模式（非图片附件与链接预览被阻断）加企业邮件 App 关闭预览；例外按组授予、绑到期日与替代控制、人数进 KPI |

# 三、锁定模式与 MDM：冲突边界比传闻窄，也更可治

Apple 说明锁定模式下无法安装新描述文件、也无法注册 MDM 或监督；但开启前已注册的设备仍受管理，仍可继续装卸描述文件。锁定模式也不是可由管理员配置的 MDM 选项——没有键强制开启，也没有键强制关闭。真实冲突因此是**注册时机问题**（新机、换机、自助注册需临时关闭再开），不是管控权丧失。

| 岗位画像 | 建议 | 要接受的代价 |
| --- | --- | --- |
| CXO、法务、并购、财务终审 | 先 ADE 注册下发基线，再开锁定模式；只留必要托管 App；保留企业 DNS 与代理出口 | 换机与重装需临时关闭锁定模式，应设两人见证流程 |
| 外勤记者、频繁口岸通行人员 | 专用高防护设备：开锁定模式、不注册 MDM、不装企业 App、不启 iCloud 备份，只用浏览器加 FIDO2 访问 SaaS | 失去设备遥测，必须由无管理声明即降权的条件访问承接 |
| 一般员工 | 不开锁定模式，靠第二章基线加条件访问；敏感期可临时开启并登记例外 | 依赖 JIT 的业务 Web 需进例外站点，风险随例外回升 |

不能把强制关闭锁定模式写成合规条款：**技术上做不到**，没有策略键；**风险方向错**，这是用户侧唯一能关掉 JIT 的开关，恰是纯 JS 链最依赖的执行条件；**治理上不成立**，让特定人群因管理便利承担更高风险，过不了评审。替代条款：高风险岗默认推荐锁定模式，其余岗位开或不开绑定访问等级——开即降权、不开即承担基线，本人书面确认。

# 四、MTD 能力边界：把宣传语逐条对到平台事实

| 宣传能力 | iOS 实际允许到什么程度 | 采购判定 |
| --- | --- | --- |
| 实时扫描系统内存、检测进程注入、发现 powerd 或 SpringBoard 后门 | 不允许：第三方 App 无 root、无跨进程内存读取、访问不了其他 App 沙箱，也无内核完整性接口 | 判为夸大：常见实现只是网络侧推断或重启后的配置比对 |
| 设备完整性与篡改评估 | 部分允许：越狱残留特征、已装描述文件与证书清单、监督状态、系统版本、DDM 回执 | 只作低置信度信号与配置漂移告警，不能写成判定失陷 |
| 网络与 DNS 侧检测 | 允许且有实值：内容过滤数据源与包过滤、NE 隧道与过滤扩展、受管 Wi-Fi 强制代理；再加私有根证书才有完整可见性 | MTD 在本类威胁下最有价值的部分 |
| 受管 App 内行为风控 | 允许：SDK 在自身 App 内可做会话、截屏与复制约束；管不到 Safari | 按 MAM 场景签真值，不签全设备行为检测 |
| 证明未被国家级监控植入 | 不可能：Apple 不提供此类证明，第三方也没有 | 改为给出信号源与置信度，否则不可验收 |
| SOAR 自动处置 | 部分允许：装卸描述文件、强制装卸托管 App、远程擦除与触发更新；无远程杀进程，断网只能靠 Wi-Fi 与代理配置近似 | 剧本按平台允许动作重写，不照搬 PC EDR |

POC 只验三件事：水坑页能否在域名层阻断并留下可归属到用户的日志；关掉 TLS 检查还剩多少检测力；供应商声称的 MDM 联动动作在自有 MDM 上是否可达。

# 五、补丁 SLA：按暴露面分档，不搞全员一个时限

DarkSword 影响 iOS 18.4–18.7、在 26.3 收口，Coruna 覆盖 iOS 13.0–17.2.1，这个跨度就是 SLA 前提：升到最新覆盖不了所有人。治理时序与 KEV 机制第 80 篇已述，此处只借时限口径。

| 人群 | 可见性延迟 | 常规动作 | 超期处置 |
| --- | --- | --- | --- |
| 面向公网的高价值人群 | 0–3 天，后台安全改进保持允许 | 维护窗口内远程触发安装；出差前版本核查 | 72h 未合规降权至只读，切换专用高防护设备 |
| 普通员工设备 | ≤30 天，够测关键 App 又不留跨月已知利用窗口 | 以最低允许版本判定符合性，版本分布按月出表 | 超两个周期降级访问并关闭附件下载 |
| 业务受限设备 | ≤90 天（Apple 上限），且必须与公网隔离 | 受管 Wi-Fi 加允许清单，关闭企业证书信任 | 清单外出口一律拒绝，否则退出企业数据路径 |

事件驱动时限（以 Apple 公告或确认在野为 T0）：4h 出资产盘点（版本、机型、受管、例外四维）；24h 高价值人群完成补丁或切换替代设备，72h 该人群全覆盖；7d 全员最低允许版本生效；30d 长尾进入隔离或替换。

长尾只有三种处置，必须明码选一种：**隔离**——网络白名单、停托管 App 配置、收回企业证书信任；**降级权限**——只读、禁附件、短会话、关 AutoFill，设备留在通信面但移出数据面；**替换硬件**——老机型先试 Apple 补发的 iOS 15.8.7 与 16.7.15，仍无法覆盖的退出企业路径。

指标固定五个：MDM 注册率、版本达标率、例外到期未回收次数、MDM 命令成功率、公告日到全覆盖的中位天数。

# 六、网络、身份与开发/CI 侧补偿控制

**网络侧。** 常被忽略的边界：DNS 设置载荷中禁止用户关闭 DNS 的键仅监督可用，且文档写明从 MDM 安装时只作用于受管 Wi-Fi、手工安装才额外覆盖蜂窝——出差中的蜂窝流量要走企业 DNS 只能靠 Network Extension 隧道。私有根证书让 TLS 检查可行，SCEP 下发的 mTLS 设备证书把设备身份绑进会话。

**身份侧。** 首选 FIDO2、安全密钥或平台 passkey，并把 SMS 与语音验证码从企业应用认证路径移除——短信二次验证对高风险人群本身就是钓鱼面。条件访问把受管、监督、版本达标、例外名单、网络风险作为分级声明下发；高风险账号用小时级会话上限加敏感操作再认证。App 层可用 DeviceCheck 与 App Attest 确认请求来自真机上的正版 App 实例（SEP 绑定密钥），但它证明 App 实例与密钥合法性，不证明 OS 未被内核级篡改。

**开发与 CI 侧。** 公开报道显示 Coruna 曾借被投毒的 npm 包投递：`art-template` 的 4.13.5 与 4.13.6 于 2026-05-20 遭劫持，把开发者访问的预览与文档站点变成水坑——开发机与 CI 出口也在攻击路径上。要做：lockfile 强制与可重现安装；私有镜像加白名单审批与新版本观察期；发布来源校验（provenance、签名、发布者账号变更告警）；CI 出网白名单，构建机不访问内部预览站；内部预览站默认需认证、不暴露公网。事件响应：比对全量 lockfile、吊销匿名访问、核查访问过预览站的设备版本并临时收紧 AutoFill 与钥匙串策略；开发设备关主机配对、启 USB 受限模式。

# 七、结论：三档建设路线

| 档位 | 优先动作 | 预算敏感项 | 明确不做 |
| --- | --- | --- | --- |
| A：只有一套 MDM 的小团队 | 新设备一律 ADE 监督；更新延迟 7 天并推行最低允许版本；关自动填充与 iCloud 钥匙串同步；内容过滤以 ≤500 条拒绝清单起步 | 监督换机成本、代理与 DNS 日志留存周期 | 不买 MTD：同价位优先换掉升不到已收口版本的老旧机型 |
| B：有 EDR 加 MDM 的中企业 | 分人群 SLA 与例外名单到期回收；受管 Wi-Fi、DNS、mTLS 全覆盖；先注册再开锁定模式写进换机 SOP | SSL 检查的证书生命周期运维、MDM 与 SOAR 联动开发 | 不接受实时内存扫描或证明无植入的验收条款 |
| C：有 MTD、SOAR 与高危人群计划的大企业 | 高危岗双设备（受管主力加锁定模式专用）；高价值人群 Web 出口进允许清单；CI 供应链治理常态化；以公告日为 T0 的 24/72 小时演练；SLA 指标进 KPI | 专用高防护设备的发放回收运维、跨法务的员工告知与合规评审 | 不把 MTD 当补丁替代品，不把未注册 BYOD 计入覆盖率分子 |

三条不可约事实：版本收敛是唯一能同时打断 Coruna 5 条链与 DarkSword 6 个 CVE 的动作，其余控制只降低投递概率或外泄收益；可用控制面在 Apple 文档里是明码标价的，超出平台允许能力的承诺不可验收；锁定模式与 MDM 的冲突是注册时机问题，写成禁令只会把入口抑制手段从最需要它的人手里拿走。

> **合规声明**：本篇策略与工具仅面向企业自有或经员工明确授权管理的设备。涉及内容过滤、流量审计、设备遥测的策略配置，须遵循所在司法辖区的劳动、隐私与数据保护法律，事先履行书面告知与协商程序，并按最小必要原则限定采集范围与留存周期；BYOD 场景应严格区分企业数据分区与个人区，不得对个人内容进行监控。本文不提供任何针对个人设备的监控建议或规避系统防护的方法。

# 参考来源

- Apple Developer Documentation — Restrictions（键定义与监督、版本门槛）：https://developer.apple.com/documentation/devicemanagement/restrictions
- Apple Developer Documentation — Web Content Filter 载荷：https://developer.apple.com/documentation/devicemanagement/webcontentfilter
- Apple Developer Documentation — DNS Settings 载荷：https://developer.apple.com/documentation/devicemanagement/dnssettings
- Apple 平台部署 — 为 Apple 设备安装和执行软件更新（延迟上限 90 天）：https://support.apple.com/zh-cn/101906
- 关于"锁定模式"（含已注册设备仍受管理、非 MDM 可配置项）：https://support.apple.com/HT212650
- Apple DeviceCheck 与 App Attest：https://developer.apple.com/documentation/devicecheck
- iVerify — Coruna 利用套件分析：https://iverify.io/blog/coruna-inside-the-nation-state-grade-ios-exploit-kit-we-ve-been-tracking
- iVerify — DarkSword 利用套件分析：https://iverify.io/blog/darksword-ios-exploit-kit-explained
- Socket — art-template npm 包遭劫持用于投递 iOS 利用套件：https://socket.dev/blog/coruna-respawned-compromised-art-template-npm-package

*报告结束。本报告基于截至 2026-10-28 的公开资料编写，MDM 可用策略键与第三方能力随版本与厂商变化，落地前需在自有设备上实测。*

**咨询iOS系统安全相关问题，请联系 Telegram：https://t.me/one00190**
