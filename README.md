# ShadowRocket-DNS
在iOS/MacOS中ShadowRocket设置DNS最佳实践

## 问题的提出
在一个充满着拦截、注入、DPI（深入包检测）的网络环境里，DNS域名解析是一个被审计的重要风险点。
当DNS解析包被审计时，可能会有：
### 1. 明文DNS（UDP/53、TCP/53）被监听和劫持
	•	查询内容明文传输，任何中间节点（如运营商）都可实时看到你访问的域名。
	•	可被用于审计、行为建模、封锁预处理。
	•	可被运营商或墙系统返回错误地址（劫持）或注入 NXDOMAIN 拒绝解析。
	•	是最容易被主动污染的链路层。
📌 风险等级：极高
⸻
### 2. DoH 被审计、记录与阻断
	•	如使用 dns.alidns.com, doh.pub, dns.ustc.edu.cn 等 DoH 服务器：
	•	查询内容虽然加密，但元数据泄露严重（你在用 DoH，访问了哪个 DoH 域，发了多少请求，频率如何）。
	•	DoH 提供商可能配合审计系统保留访问日志，甚至实时上传。
	•	你访问 doh.pub 就等于告诉系统你用了代理，并试图隐藏行为。
	•	在关键时期可能直接封锁 DoH 服务或返回污染 IP。
📌 风险等级：中-高（取决于服务商与时期）
⸻
### 3. DoH/IP目标路径上被运营商识别与限速、丢包、阻断
	•	如使用 https://dns.google/dns-query、cloudflare-dns.com：
	•	虽然内容是 TLS 加密的，但运营商可识别出特征行为和连接目标（SNI、IP、路径、流量频率）。
	•	某些网络环境中会对这些 IP 地址限速、阻断（尤其 Cloudflare）。
	•	有能力的 DPI 可对流量频率与 TLS ClientHello 特征做行为分析，判定你是特定网络行为。
📌 风险等级：中，但在特定环境下仍然危险
⸻
### 4. DoH/DoT 的 SNI 明文暴露（除非使用 ECH）
	•	大多数 DoH 服务仍使用 TLS 1.2/1.3 明文 SNI（dns.google、cloudflare-dns.com 等）。
	•	DPI 设备可以直接通过 SNI 得知你在访问哪个 DoH 服务。
	•	当前ECH（Encrypted Client Hello）支持极差，无法指望。
📌 风险等级：中
⸻
### 5. DNS over HTTPS 并非真正匿名（中继流量可被元数据分析）
	•	虽然 DoH 加密了内容，但你的请求目标是公开的（如 1.1.1.1、8.8.8.8）。
	•	即使不解密内容，元数据（谁访问、访问频率、时间特征）可被用于“访问图谱重建”。
	•	典型攻击：“关联分析 + 数据指纹 + 目标预测”。不需要知道你访问了什么，只要知道“你在做规避行为”。
📌 风险等级：中
⸻
### 6. DNS缓存污染、冷启动时陷阱
	•	首次启动时解析代理域名，若用系统 DNS 或公共 DNS，极易被提前拦截或污染。
	•	某些设备或客户端无完整控制 DNS 层，会偷偷发出系统 DNS 查询（如 Android 系统服务、低层 stub resolver）。
	•	部分配置失误（如 Apple 的 Happy Eyeballs 双连接策略）可能导致 DNS 泄露。
📌 风险等级：中偏高（冷启动极易中招）

总结起来，想要完全避免DNS导致的信息泄露、被审计日志记录、被污染的DNS解析干扰是几乎没有可能的。
但是我们可以研究一下有什么思路和卡点可以帮助我们缓和此类风险

### 第一件事情：不要用Android系统
### 第二件事情：不要用受限地的Apple ID

以上两条做不到，就别往下看了，没有任何办法防止你的访问被审计、被劫持、被识别

## 思路与卡点
### ⛔️ 卡点一：系统冷启动时无法解析代理服务器域名
📌 原因：
	•	代理配置依赖域名（如 us1.myproxy.net）
	•	启动初期还没建立代理通道，只能依赖本地 DNS 或系统 resolver
	•	明文 DNS 被墙污染或封锁，DoH 尚未生效
✅ 解决思路：
	•	为 DoH 服务添加静态 IP 绑定（例如 dns.alidns.com = 223.5.5.5）
	•	通过 host: 为代理域名配置独立的 DNS 路由策略
	•	避免初始解析走系统 DNS，确保 DoH 可用时即能解析代理域名
⸻
### ⛔️ 卡点二：DoH 请求本身被识别、限速或阻断（如 dns.google）
📌 原因：
	•	常见的 DoH 域名（如 dns.google）在中国境内早已被 DPI 特征识别
	•	HTTPS 封包即使加密，TLS SNI 暴露了目标
✅ 解决思路：
	•	使用 DoH over Proxy（配置中 https://dns.google/dns-query#proxy）
	•	使用备用或小众 DoH，如：
	•	https://security.cloudflare-dns.com/dns-query
	•	为 dns.google 加入多个 SNI/Fake TLS Fingerprint 对抗特征识别（限 Surge、Clash Meta）
⸻
### ⛔️ 卡点三：系统/APP偷偷发出明文DNS请求
📌 原因：
	•	iOS/macOS 中部分系统进程绕过代理层（如 Captive Portal 检测）
	•	某些 APP（QQ、微信）使用 native DNS API，无视代理设置
✅ 解决思路：
	•	使用 DST-PORT,53,REJECT-DROP 明文 DNS 全局封杀
	•	启用 DoH-only 模式，不配置 system DNS fallback
	•	如果是 Clash Meta，使用 dns.fake-ip 全局接管 DNS 查询
⸻
### ⛔️ 卡点四：代理节点 IP 被墙，DNS 正确但无法连通
📌 原因：
	•	虽然 DNS 解析成功（通过 DoH），但返回的 IP 已被墙封锁或 reset
	•	例如 us1.myproxy.net 的解析结果 198.x.x.x 已被封
✅ 解决思路：
	•	启用 proxy-providers 配合 health-check 和 failover 策略自动换节点
	•	尽可能使用 CDN 前置（如 Cloudflare Workers + RealIP fallback）
	•	为所有代理节点域名加多个备选 IP，通过 round-robin 或域名负载均衡
⸻
### ⛔️ 卡点五：DoH 请求未走代理，导致节点可用但解析失败
📌 原因：
	•	忘记为主 DNS 加 #proxy
	•	DNS 请求直接出境，被拦截或返回错误结果
✅ 解决思路：
	•	强制所有主 DNS 与 fallback DNS 加 #proxy
	•	特殊域名如 *.myproxy.net 使用 server: 分流至更贴近出口的 DoH 服务
	•	使用 Shadowrocket 的 DNS 日志功能确认 DoH 请求路径是否走代理
⸻
### ⛔️ 卡点六：规则冲突导致 DNS 请求泄露或代理失效
📌 原因：
	•	DNS请求走了不该走的规则（如常见误用 FINAL,DIRECT）
	•	部分域名未被正确分流至 server: 或 proxy
✅ 解决思路：
	•	使用 DOMAIN-SUFFIX,xxx,USE-DNS 精细绑定 DNS 使用路径
	•	使用 HOST 精确覆盖关键域名，如：
        dns.alidns.com = 223.5.5.5
        cloudflare-dns.com = 104.16.249.249
### ⛔️ 卡点七：测速失败但节点实际可用
📌 原因：
	•	测试 URL 对代理节点返回不友好（301/403/timeout）
	•	某些轻量接口对无 UA 或异常来源返回非 200
✅ 解决思路：
	•	使用已验证的 轻量测速 URL：
	•	https://res.wx.qq.com/zh_CN/htmledition/images/icon/favicon.ico
	•	https://dns.google/favicon.ico
	•	https://cloudflare.com/favicon.ico
	•	或自建测速端点（小 HTML/静态文件）托管于 CDN 上
⸻
### ⛔️ 卡点八：非典型 DNS流量被行为识别（主动审计）
#### 📌 原因：
	•	请求频率、目标集中、TLS 指纹特征都可以被标记
	•	即使走了代理，但某些行为（如持续请求 dns.google）仍可能触发行为黑名单

#### ✅ 解决思路：
	•	多 DoH 服务混用，分布式负载（避免单一流量特征）
	•	使用带有 utls 模拟 TLS 指纹的客户端（Clash Meta、Hysteria2）
	•	不在 General 配置中默认 fallback 至本地 DNS，始终显式 DoH 绑定

 
