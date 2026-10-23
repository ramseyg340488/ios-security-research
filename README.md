# iOS 安全攻防研究系列

现代 iOS 纵深防御体系的系列研究笔记：从芯片与系统安全架构，讲到攻击链的形成逻辑，再回到检测与加固。

> 内容为防御性安全研究与公开资料整理，不包含可用的漏洞利用代码。

## 目录

本系列共 42 篇，按顺序陆续发布。

1. [iOS 安全架构演进：SPTM、TXM 与 Exclaves 技术体系解析](./01-iOS安全架构演进-SPTM-TXM与Exclaves技术体系解析.md)
2. [iOS 安全沙箱机制深度解析：从 Coruna 与 DarkSword 看沙箱逃逸的攻防演进](./02-iOS安全沙箱机制深度解析-从Coruna与DarkSword看沙箱逃逸的攻防演进.md)
3. [iOS 安全研究专题：Coruna 与 DarkSword 漏洞链深度解析](./03-iOS安全研究专题-Coruna与DarkSword漏洞链深度解析.md)
4. [iOS 高级漏洞研究：从 WebKit 到内核提权的移动安全攻防演进](./04-iOS高级漏洞研究-从WebKit到内核提权的移动安全攻防演进.md)
5. [iOS 固件更新安全分析：从 OTA 验证到降级防护的攻击面研究](./05-iOS固件更新安全分析-从OTA验证到降级防护的攻击面研究.md)
6. [iOS 后台执行与持久化机制分析：从 BackgroundTasks 到系统守护进程的驻留攻防](./06-iOS后台执行与持久化机制分析-从BackgroundTasks到系统守护进程的驻留攻防.md)
7. [iOS Coruna 与 DarkSword 漏洞利用链深度分析：攻击机制、安全风险与防御体系](./07-iOS-Coruna与DarkSword漏洞利用链深度分析-攻击机制安全风险与防御体系.md)
8. [Coruna 漏洞利用链中的 WebKit 入口点深度分析：从 CVE-2024-23222 到浏览器 RCE 机制](./08-Coruna漏洞利用链中的WebKit入口点深度分析-从CVE-2024-23222到浏览器RCE机制.md)
9. [Coruna 的 PlasmaLoader 载荷深度解析：金融信息窃取机制与加密货币威胁](./09-Coruna的PlasmaLoader载荷深度解析-金融信息窃取机制与加密货币威胁.md)
10. [ARM 指针认证码 PAC 绕过技术前沿：从 Coruna 到 DarkSword 的控制流攻击实践](./10-ARM指针认证码PAC绕过技术前沿-从Coruna到DarkSword的控制流攻击实践.md)
11. [Mach IPC 与 XNU 内核攻击面：Coruna 与 DarkSword 利用链中的系统调用深度解析](./11-MachIPC与XNU内核攻击面-Coruna与DarkSword利用链中的系统调用深度解析.md)
12. [从 Coruna 与 DarkSword 看 iOS 沙箱逃逸技术演进：从 WebContent 到系统服务的攻击路径](./12-从Coruna与DarkSword看iOS沙箱逃逸技术演进-从WebContent到系统服务的攻击路径.md)
13. [iOS 漏洞主要类型深度解析：从内存安全到逻辑攻击面的全景分类体系](./13-iOS漏洞主要类型深度解析-从内存安全到逻辑攻击面的全景分类体系.md)
14. [iOS 权限模型与攻击面映射：从隐私权限到内核能力的落差分析](./14-iOS权限模型与攻击面映射-从隐私权限到内核能力的落差分析.md)
15. [iOS 网络协议栈安全分析：从 TLS 到 DNS 的攻击面与 Coruna 级远程利用的通信基础](./15-iOS网络协议栈安全分析-从TLS到DNS的攻击面与Coruna级远程利用的通信基础.md)
16. [iOS 基带与蜂窝协议栈安全分析：从远程投递到持久化控制的通信基础设施威胁](./16-iOS基带与蜂窝协议栈安全分析-从远程投递到持久化控制的通信基础设施威胁.md)
17. [iOS 蓝牙与近场通信攻击面分析：Coruna 与 DarkSword 生态中无线链路的安全边界](./17-iOS蓝牙与近场通信攻击面分析-Coruna与DarkSword生态中无线链路的安全边界.md)
18. [iOS 生物识别与本地认证安全：Touch ID 与 Face ID 在高级攻击链中的信任边界](./18-iOS生物识别与本地认证安全-TouchID与FaceID在高级攻击链中的信任边界.md)
19. [Secure Enclave 与密钥保护边界分析：从 Coruna 与 DarkSword 看 SEP 的攻防价值](./19-SecureEnclave与密钥保护边界分析-从Coruna与DarkSword看SEP的攻防价值.md)
20. [iOS 密码学服务与钥匙串安全：从 SecItem 到 CryptoKit 的密钥生命周期威胁分析](./20-iOS密码学服务与钥匙串安全-从SecItem到CryptoKit的密钥生命周期威胁分析.md)
21. [iOS 应用数据容器与 App Group 安全：从共享文件到跨应用横向移动的攻防分析](./21-iOS应用数据容器与AppGroup安全-从共享文件到跨应用横向移动的攻防分析.md)
22. [iOS 数字取证与反取证对抗：Coruna 与 DarkSword 痕迹清理与检测取证的博弈](./22-iOS数字取证与反取证对抗-Coruna与DarkSword痕迹清理与检测取证的博弈.md)
23. [iOS 内核安全机制深度解析：从 XNU 架构到现代移动安全防御体系（上）](./23-iOS内核安全机制深度解析-从XNU架构到现代移动安全防御体系-上.md)
24. [XNU Kernel 漏洞演进史：从 iOS 10 到 iOS 18 的安全架构变化](./24-XNU-Kernel漏洞演进史-从iOS10到iOS18安全架构变化.md)
25. [XNU 内核漏洞缓解机制演进：从堆喷利用到现代内核攻防技术对抗](./25-XNU内核漏洞缓解机制演进-从堆喷利用到现代内核攻防技术对抗.md)
26. [iOS 内存分配器与堆利用缓解：从 kalloc 隔离到 MTE 时代的内核利用工程评估](./26-iOS内存分配器与堆利用缓解-kalloc隔离到MTE时代的内核利用工程评估.md)
27. [PAC 绕过技术演进：ARM64e 时代的控制流保护分析](./27-PAC绕过技术演进-ARM64e时代控制流保护分析.md)
28. [iOS 代码签名体系攻防演进：从 AMFI 到内核签名验证的突破与加固](./28-iOS代码签名体系攻防演进-从AMFI到内核签名验证的突破与加固.md)
29. [SPTM、TXM 与 Exclaves：苹果下一代安全架构解析](./29-SPTM-TXM与Exclaves苹果下一代安全架构解析.md)
30. [SPTM/TXM 深度解析：苹果如何重新定义 Kernel 安全边界](./30-SPTM-TXM深度解析-苹果如何重新定义Kernel安全边界.md)
31. [Memory Integrity Enforcement：苹果下一代内存安全战略分析](./31-Memory-Integrity-Enforcement苹果下一代内存安全战略分析.md)
32. [现代 iOS Kernel 漏洞挖掘方法论：从源码审计到安全验证](./32-现代iOS-Kernel漏洞挖掘方法论-从源码审计到安全验证.md)
33. [从 Coruna 到 DarkSword：iOS 攻击链演进时间线（2024—2026）](./33-从Coruna到DarkSword-iOS攻击链演进时间线-2024至2026.md)
34. [DarkSword 六个漏洞协同利用时序分析：攻击阶段依赖关系与关键路径](./34-DarkSword六个漏洞协同利用时序分析-攻击阶段依赖关系与关键路径.md)
35. [JavaScriptCore 类型混淆漏洞利用原理：从 Coruna 到 DarkSword 的 JIT 安全攻防](./35-JavaScriptCore类型混淆漏洞利用原理-从Coruna到DarkSword的JIT安全攻防.md)
36. [Coruna 与 DarkSword 设备指纹识别机制对比：如何选择最优攻击链](./36-Coruna与DarkSword设备指纹识别机制对比-如何选择最优攻击链.md)
37. [Coruna 与 DarkSword 对 Exclaves 安全架构的启示：从 XNU 失陷到跨域攻击边界](./37-Coruna与DarkSword对Exclaves安全架构的启示-从XNU失陷到跨域攻击边界分析.md)
38. [Coruna 与 DarkSword 对加密货币用户的定向威胁：助记词窃取与资产保护](./38-Coruna与DarkSword对加密货币用户的定向威胁分析-助记词窃取钱包攻击与资产保护.md)
39. [Coruna 与 DarkSword 威胁建模与攻击面量化：从资产分类到风险矩阵](./39-Coruna与DarkSword威胁建模与攻击面量化-从资产分类到风险矩阵的系统安全评估.md)
40. [针对 Coruna 与 DarkSword 的网络层检测：流量特征分析与 IOC 提取](./40-针对Coruna与DarkSword的网络层检测方法-流量特征分析与IOC提取.md)
41. [iOS 设备管理与配置描述文件攻击面：从 MDM 到声明式设备管理的滥用与防护](./41-iOS设备管理与配置描述文件攻击面-从MDM到声明式设备管理的滥用与防护.md)
42. iOS 漏洞研究学习路线：从入门到高级安全研究员（待发布）

## 阅读顺序建议

先建立架构认知（第 1 篇），再理解隔离边界（第 2 篇），然后看真实攻击链如何跨越这些边界（第 3、4 篇），扩展到固件更新与驻留持久化两个容易被忽略的维度（第 5、6 篇）。第 7 至 12 篇为 Coruna 与 DarkSword 专题：利用链总览（第 7 篇）、浏览器入口（第 8 篇）、载荷与窃取（第 9 篇）、控制流保护对抗（第 10 篇）、内核接口攻击面（第 11 篇）、沙箱逃逸（第 12 篇）。第 13 至 22 篇为攻击面与纵深防御全景：分类体系（第 13 篇）、权限模型（第 14 篇）、三条通信链路（第 15 至 17 篇）、认证与密钥保护（第 18 至 20 篇）、跨应用数据与取证（第 21、22 篇）。第 23 篇起进入内核与缓解机制专题：XNU 架构（第 23 篇）与漏洞演进史（第 24 篇），缓解机制（第 25 篇）、堆利用工程（第 26 篇）、PAC 控制流保护（第 27 篇）、代码签名体系（第 28 篇），再看苹果如何重画信任边界（第 29 至 31 篇的 SPTM/TXM/Exclaves 与 MIE），最后以内核漏洞挖掘方法论收尾（第 32 篇）。第 33 至 42 篇为 Coruna 与 DarkSword 二期专题：先补齐时间线与利用时序（第 33、34 篇），再深入到 JIT 类型混淆原理与设备指纹选择（第 35、36 篇），接着把攻击放回新边界上做对照（第 37 篇的 Exclaves 启示），转向被定向的目标与评估方法（第 38 篇的加密货币用户威胁、第 39 篇的威胁建模与风险矩阵），最后落到可操作的防御与能力建设（第 40 篇网络层检测、第 41 篇 MDM 与配置描述文件攻击面、第 42 篇漏洞研究学习路线）。

## 说明

同一系列在 GitHub 与 Gitee 同步发布。
