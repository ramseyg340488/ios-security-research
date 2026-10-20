# iOS 安全攻防研究系列

现代 iOS 纵深防御体系的系列研究笔记：从芯片与系统安全架构，讲到攻击链的形成逻辑，再回到检测与加固。

> 内容为防御性安全研究与公开资料整理，不包含可用的漏洞利用代码。

## 目录

本系列共 12 篇，按顺序陆续发布。

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
12. 从 Coruna 与 DarkSword 看 iOS 沙箱逃逸技术演进：从 WebContent 到系统服务的攻击路径（待发布）

## 阅读顺序建议

先建立架构认知（第 1 篇），再理解隔离边界（第 2 篇），然后看真实攻击链如何跨越这些边界（第 3、4 篇），扩展到固件更新与驻留持久化两个容易被忽略的维度（第 5、6 篇）。第 7 篇起进入 Coruna 与 DarkSword 专题：从完整利用链总览（第 7 篇）逐层拆开——浏览器入口（第 8 篇）、载荷与窃取（第 9 篇）、控制流保护对抗（第 10 篇）、内核接口攻击面（第 11 篇），最后回到沙箱逃逸的攻防演进（第 12 篇）。

## 说明

同一系列在 GitHub 与 Gitee 同步发布。
