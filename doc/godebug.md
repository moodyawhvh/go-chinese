---
title: "Go、向后兼容性与 GODEBUG"
layout: article
---

> 🌐 本文档由 [golang/go](https://github.com/golang/go) 翻译,英文原版见原项目。

<!--
本文档保存在 Go 仓库(而非 x/website)中,
因为它记录了已知 GODEBUG 设置的完整列表,
而这些设置与特定发行版绑定。
-->

> 📝 注:原文超过 10000 字符,本译文完整翻译核心章节(简介、默认 GODEBUG 值);「GODEBUG 历史」一节逐条明细见英文原版,此处保留各版本核心变更的中文摘要。

## 简介 {#intro}

对向后兼容的重视是 Go 的关键优势之一。然而,有些时候我们无法保持完全的兼容。如果代码依赖有缺陷(包括不安全)的行为,那么修复该缺陷就会破坏这段代码。新特性也可能造成类似影响:让 HTTP 客户端默认启用 HTTP/2,就曾破坏过连接到 HTTP/2 实现有缺陷的服务器的程序。这类变更不可避免,且[为 Go 1 兼容性规则所允许](/doc/go1compat)。即便如此,Go 仍提供了一种名为 GODEBUG 的机制,以减轻这类变更对"用新工具链编译旧代码"的 Go 开发者的冲击。

GODEBUG 设置是控制 Go 程序某些部分执行行为的 `key=value` 键值对。环境变量 `GODEBUG` 可以容纳以逗号分隔的多个此类设置。例如,如果 Go 程序运行在包含以下内容的环境中:

	GODEBUG=http2client=0,http2server=0

那么该程序将默认在 HTTP 客户端和 HTTP 服务器中都禁用 HTTP/2。`GODEBUG` 环境变量中无法识别的设置会被忽略。也可以为某个程序设置默认的 `GODEBUG`(下文讨论)。

在准备任何"虽为 Go 1 兼容性规则所允许、但可能破坏某些现有程序"的变更时,我们会先对变更进行工程化设计,尽可能让更多现有程序继续正常工作。对于剩余的程序,我们定义一个新的 GODEBUG 设置,允许个别程序选择回到旧行为。若确实不可行,可以不添加 GODEBUG 设置,但这应当极其罕见。

为兼容性而添加的 GODEBUG 设置至少会维护两年(四个 Go 发行版)。有些设置,比如 `http2client` 和 `http2server`,会维护更久,甚至无限期维护。

在可能的情况下,每个 GODEBUG 设置都有一个关联的 [runtime/metrics](/pkg/runtime/metrics/) 计数器,名为 `/godebug/non-default-behavior/<name>:events`,用于统计某程序的行为因该设置取非默认值而发生改变的次数。例如,设置 `GODEBUG=http2client=0` 时,`/godebug/non-default-behavior/http2client:events` 会统计程序配置了多少个不支持 HTTP/2 的 HTTP 传输。

## 默认 GODEBUG 值 {#default}

当某个 GODEBUG 设置未在环境变量中列出时,其值来自三个来源:构建程序所用 Go 工具链的默认值,再根据 `go.mod` 中列出的 Go 版本进行修正,最后被程序中显式的 `//go:debug` 行覆盖。

[GODEBUG 历史](#history)给出了每个 Go 工具链版本的确切默认值。例如,Go 1.21 引入了 `panicnil` 设置,控制是否允许 `panic(nil)`;其默认值为 `panicnil=0`,使 `panic(nil)` 成为运行时错误。使用 `panicnil=1` 可恢复 Go 1.20 及更早版本的行为。

当编译声明了较旧 Go 版本的工作模块或工作区时,Go 工具链会修正其默认值,使其尽可能匹配那个较旧的 Go 版本。例如,当 Go 1.21 工具链编译某个程序时,如果工作模块的 `go.mod` 或工作区的 `go.work` 写着 `go` `1.20`,那么该程序默认采用 `panicnil=1`,即匹配 Go 1.20 而非 Go 1.21。

作为例外,为安全补丁版本引入的 GODEBUG 会把新行为应用于所有版本。

由于这种设置 GODEBUG 默认值的方法直到 Go 1.21 才引入,声明了早于 Go 1.20 版本的程序会被配置为匹配 Go 1.20,而不是更旧的版本。

要覆盖这些默认值,从 Go 1.23 起,工作模块的 `go.mod` 或工作区的 `go.work` 可以列出一个或多个 `godebug` 行:

	godebug (
		default=go1.21
		panicnil=1
		asynctimerchan=0
	)

特殊的键 `default` 表示一个 Go 版本,未指定的设置将从该版本取默认值。这使得 GODEBUG 默认值可以与模块中的 Go 语言版本分开设置。在本例中,程序要求 Go 1.21 语义,然后要求 Go 1.21 之前的 `panic(nil)` 旧行为,以及 Go 1.23 的 `asynctimerchan=0` 新行为。

只有工作模块的 `go.mod` 会被查询 `godebug` 指令;被依赖模块中的任何指令都会被忽略。列出无法识别的 `godebug` 设置是错误。(早于 Go 1.23 的工具链会拒绝所有 `godebug` 行,因为它们完全不理解 `godebug`。)使用工作区时,`go.mod` 文件中的 `godebug` 指令会被忽略,转而查询 `go.work` 中的 `godebug` 指令。

`go` 行与 `godebug` 行产生的默认值适用于所有被构建的主包。若要更细粒度的控制,从 Go 1.21 起,主包的源文件可以在文件顶部(`package` 语句之前)包含一个或多个 `//go:debug` 指令。上例中的 `godebug` 行可以写成:

	//go:debug default=go1.21
	//go:debug panicnil=1
	//go:debug asynctimerchan=0

从 Go 1.21 起,Go 工具链会把带有无法识别的 GODEBUG 设置的 `//go:debug` 指令视为无效程序;对同一设置写有多条 `//go:debug` 行的程序同样视为无效。(更早的工具链会完全忽略 `//go:debug` 指令。)

编译进某个主包的默认值可通过以下命令查看:

	go list -f '{{.DefaultGODEBUG}}' my/main/package

只报告与基础 Go 工具链默认值不同的部分。

测试包时,`*_test.go` 文件中的 `//go:debug` 行会被当作测试主包的指令。在其他任何上下文中,`//go:debug` 行都会被工具链忽略;`go` `vet` 会把此类行报告为位置不当。

## GODEBUG 历史 {#history}

本节记录每个 Go 大版本中出于兼容性原因新增和移除的 GODEBUG 设置。包或程序也可能为内部调试目的定义额外设置;例如参见 [runtime 文档](/pkg/runtime#hdr-Environment_Variables)与 [go 命令文档](/cmd/go#hdr-Build_and_test_caching)。

> 📝 以下为各版本核心变更中文摘要,逐条完整明细请参阅英文原版。

### Go 1.28

新增 `netmarshal` 设置,控制 `net.IPMask`、`net.IPNet`、`net.HardwareAddr` 类型的文本编组是否可读:`netmarshal=0` 以不可读的 base64 编码做 JSON 编码,`netmarshal=1` 使用可读形式(如 IP 地址文本)。Go 1.28 与 1.29 默认保持 `netmarshal=0`,确保这些版本生成的 JSON 等编组数据能被旧版本 Go 读取;预计 Go 1.30 将默认改为 `netmarshal=1`。读取数据时新旧两种编码始终都支持。该设置最早可在 Go 1.34 移除。

### Go 1.27

移除了 `gotypesalias`、`tlsunsafeekm`、`tlsrsakex`、`tls10server`、`asynctimerchan`、`x509keypairleaf`、`tls3des` 等设置(各自缘由见英文原版对应条目)。新增 `htmlmetacontenturlescape` 设置,控制 html/template 是否转义 HTML meta 标签 content 属性 `url=` 部分中的 URL,默认转义;为防内容注入攻击,该设置与默认值已回移至 Go 1.25.8 和 Go 1.26.1。`tracebacklabels`(Go 1.26 引入)默认值改为 `1`,该退出开关预计无限期保留。新增 `x509sslcertoverrideplatform` 设置,控制 Windows/Darwin 上设置 `SSL_CERT_FILE`/`SSL_CERT_DIR` 时是否从磁盘加载根证书(默认加载;计划 Go 1.31 移除)。新增 `fips140ems` 设置,置 `0` 时在 FIPS 140-3 模式下不再强制 Extended Master Secret(已回移至 Go 1.26.6 和 Go 1.25.13;计划 Go 1.31 移除)。

### Go 1.26

新增 `httpcookiemaxnum` 设置(默认 3000),限制 net/http 解析 HTTP 头时接受的 cookie 最大数量,超限即提前失败;置 `0` 表示不限制。为防拒绝服务攻击,已回移至 Go 1.25.2 和 Go 1.24.8。新增 `urlmaxqueryparams` 设置(默认 10000),限制 net/url 解码查询串时接受的参数最大数量;置 `0` 关闭限制;已回移至 Go 1.25.6 和 Go 1.24.12。新增 `urlstrictcolons` 设置,`net/url.Parse` 默认拒绝 `http://localhost:1:2`、`http://::1/` 等含非法冒号的主机名(方括号 IPv6 内的冒号仍允许)。默认启用后量子密钥交换机制 SecP256r1MLKEM768 与 SecP384r1MLKEM1024,可用 [`tlssecpmlkem` 设置](/pkg/crypto/tls/#Config.CurvePreferences)回退。新增 `tracebacklabels` 设置,控制 runtime traceback 与 debug=2 的 pprof 栈转储中是否包含 runtime/pprof 设置的 goroutine 标签。新增 `cryptocustomrand` 设置,Go 1.26 默认 `0` 表示大多数 crypto/... API 忽略自定义随机 `io.Reader` 参数,置 `1` 恢复 Go 1.26 之前的行为。

### Go 1.25

新增 `decoratemappings` 设置,控制运行时是否在 /proc/self/maps、/proc/self/smaps 中为匿名内存映射标注用途信息(形如 "[anon: Go: ...]"),仅限 Linux,默认 `1` 开启,且在程序启动时固定。新增 `embedfollowsymlinks` 设置,控制 go 命令嵌入文件时是否跟随指向常规文件的符号链接,默认 `0` 不跟随。新增 `containermaxprocs` 设置(仅 Linux),控制设置默认 GOMAXPROCS 时是否考虑 cgroup CPU 限制,默认 `1`。新增 `updatemaxprocs` 设置,控制运行时是否周期性根据新的 CPU 亲和性或 cgroup 限制更新 GOMAXPROCS,默认 `1`。按 RFC 9155 在 TLS 1.2 中禁用 SHA-1 签名算法,可用 `tlssha1=1` 回退。crypto/x509.CreateCertificate 改用 SHA-256 填充缺失的 SubjectKeyId,可用 `x509sha256skid=0` 回退。修正运行时内部锁争用报告的语义,并移除 [`runtimecontentionstacks` 设置](/pkg/runtime#hdr-Environment_Variables)。自 Go 1.25 RC 2 起,检测到多个 VCS 时因 VCS 注入攻击风险而禁用构建信息打标,已回移至 Go 1.24.5 和 Go 1.23.11,可用 `allowmultiplevcs=1` 重新启用。

### Go 1.24

新增 `fips140` 设置,控制 Go 密码模块是否运行于 FIPS 140-3 模式,取值:"off"(默认,无特殊支持)、"on"(启用)、"only"(启用,且未经 FIPS 140-3 批准的密码算法返回错误或 panic)。详见 [FIPS 140-3 合规](/doc/security/fips140)。该设置在程序启动时固定,启动后改 `GODEBUG` 无效。全局 [`math/rand.Seed`](/pkg/math/rand/#Seed) 变为空操作,由 `randseednop` 控制,Go 1.24 默认 `randseednop=1`,置 `0` 恢复旧行为。`multipathtcp` 新增取值:"0"(拨号方与监听方均默认禁用)、"1"(均默认启用)、"2"(仅监听方默认启用)、"3"(仅拨号方默认启用);Go 1.24 默认 "2",置 "0" 恢复旧行为。`go test -json` 改为以 JSON 输出构建错误(新 `Action` 值区分),可能影响不够健壮的 CI 系统,由 `gotestjsonbuildtext` 控制,置 `1` 恢复 1.23 行为,该设置最早 Go 1.28 移除。[crypto/rsa](/pkg/crypto/rsa) 要求 RSA 密钥至少 1024 位,由 `rsa1024min` 控制,置 `0` 恢复 Go 1.23 行为。在 [`crypto/subtle`](/pkg/crypto/subtle) 引入启用平台相关数据独立计时(DIT)模式的机制,可用 `dataindependenttiming` 设置对整个程序启用,Go 1.24 默认 `0`;启用后从 Go 调入 C 时也保持 DIT,从 C 调入 Go 时临时启用并在返回前恢复;目前仅影响 arm64 程序。移除 `x509sha1` 设置,crypto/x509 不再支持验证基于 SHA-1 签名算法的证书。[`x509usepolicies` 设置](/pkg/crypto/x509/#CreateCertificate)默认值从 `0` 改为 `1`:编组证书时默认取 [`Certificate.Policies`](/pkg/crypto/x509/#Certificate.Policies) 字段而非 [`Certificate.PolicyIdentifiers`](/pkg/crypto/x509/#Certificate.PolicyIdentifiers)。默认启用后量子密钥交换机制 X25519MLKEM768,可用 [`tlsmlkem` 设置](/pkg/crypto/tls/#Config.CurvePreferences)回退——这对无法正确处理大记录、握手超时的有缺陷 TLS 服务器有用(见 [TLS post-quantum TL;DR fail](https://tldr.fail/));同时移除 X25519Kyber768Draft00 与 Go 1.23 的 `tlskyber` 设置。[ParsePKCS1PrivateKey](/pkg/crypto/x509/#ParsePKCS1PrivateKey) 现在使用并校验私钥编码中的 CRT 参数,由 `x509rsacrt` 控制,置 `0` 恢复 Go 1.23 行为。

### Go 1.23

time 包创建的通道改为无缓冲(同步),使 [`Timer.Stop`](/pkg/time/#Timer.Stop) 与 [`Timer.Reset`](/pkg/time/#Timer.Reset) 返回值更易正确使用;[`asynctimerchan` 设置](/pkg/time/#NewTimer)可禁用此变更;无对应的运行时指标;该设置将于 Go 1.27 移除。Windows 上 reparse point 的模式位报告改变,由 `winsymlink` 控制:Go 1.23 起(`winsymlink=1`)挂载点不再置 [`os.ModeSymlink`](/pkg/os#ModeSymlink),非符号链接/Unix 套接字/去重文件的 reparse point 一律置 [`os.ModeIrregular`](/pkg/os#ModeIrregular);[`filepath.EvalSymlinks`](/pkg/path/filepath#EvalSymlinks) 不再展开挂载点(这曾是大量不一致与 bug 的来源)。旧版本(`winsymlink=0`)把挂载点当符号链接,且其他带非默认 [`os.ModeType`](/pkg/os#ModeType) 位(如 [`os.ModeDir`](/pkg/os#ModeDir))的 reparse point 不置 `ModeIrregular` 位。[os.Readlink](/pkg/os#Readlink) 与 [`filepath.EvalSymlinks`](/pkg/path/filepath#EvalSymlinks) 不再尝试把卷名规范化为盘符(这本来也并非总可行),由 `winreadlinkvolume` 控制,Go 1.23 默认 `1`,旧版本默认 `0`。默认启用实验性后量子密钥交换机制 X25519Kyber768Draft00,可用 [`tlskyber` 设置](/pkg/crypto/tls/#Config.CurvePreferences)回退。[crypto/x509.ParseCertificate](/pkg/crypto/x509/#ParseCertificate) 改为拒绝负序列号,可用 [`x509negativeserial` 设置](/pkg/crypto/x509/#ParseCertificate)回退。html/template 默认重新支持 ECMAScript 6 模板字面量,[`jstmpllitinterp` 设置](/pkg/html/template#hdr-Security_Model)不再有任何效果。未显式配置时,客户端与服务器的默认 TLS 密码套件移除 3DES,可用 [`tls3des` 设置](/pkg/crypto/tls/#Config.CipherSuites)回退,该设置将于 Go 1.27 移除。[tls.X509KeyPair](/pkg/crypto/tls#X509KeyPair) 与 [tls.LoadX509KeyPair](/pkg/crypto/tls#LoadX509KeyPair) 会填充返回的 [tls.Certificate](/pkg/crypto/tls#Certificate) 的 Leaf 字段,由 `x509keypairleaf` 控制,Go 1.23 默认 `1`,旧版本默认 `0`,该设置将于 Go 1.27 移除。[net/http.ServeContent](/pkg/net/http#ServeContent)、[net/http.ServeFile](/pkg/net/http#ServeFile)、[net/http.ServeFS](/pkg/net/http#ServeFS) 出错响应时移除 Cache-Control、Content-Encoding、Etag、Last-Modified 头,由 [`httpservecontentkeepheaders` 设置](/pkg/net/http#ServeContent)控制,置 `1` 恢复 Go 1.23 之前的行为。

### Go 1.22

新增可配置的 [`tlsmaxrsasize` 设置](/pkg/crypto/tls#Conn.Handshake),限制 TLS 握手中可接受的 RSA 密钥最大长度,默认 tlsmaxrsasize=8192;为防拒绝服务攻击,已回移至 Go 1.19.13、Go 1.20.8 与 Go 1.21.1。net/http 客户端或服务器读到空 Content-Length 头的请求或响应时报错,由 `httplaxcontentlength` 控制。ServeMux 改为接受扩展模式并按段反转义模式与请求路径,由 [`httpmuxgo121` 设置](/pkg/net/http/#ServeMux)控制。[go/types](/pkg/go/types) 新增用于显式表示[类型别名](/ref/spec#Type_declarations)的 [Alias 类型](/pkg/go/types#Alias),类型检查器是否产出 `Alias` 类型由 [`gotypesalias` 设置](/pkg/go/types#Alias)控制,Go 1.22 默认 `0`,Go 1.23 起默认 `1`,该设置将于 Go 1.27 移除。服务器与客户端默认支持的最低 TLS 版本改为 TLS 1.2,可用 [`tls10server` 设置](/pkg/crypto/tls/#Config)回退到 TLS 1.0,该设置将于 Go 1.27 移除。未显式配置时,默认 TLS 密码套件移除基于 RSA 密钥交换的套件,可用 [`tlsrsakex` 设置](/pkg/crypto/tls/#Config)回退,该设置将于 Go 1.27 移除。当连接既不支持 TLS 1.3 也不支持扩展主密钥(Go 1.21 实现)时,禁用 [`ConnectionState.ExportKeyingMaterial`](/pkg/crypto/tls/#ConnectionState.ExportKeyingMaterial),可用 [`tlsunsafeekm` 设置](/pkg/crypto/tls/#ConnectionState.ExportKeyingMaterial)重新启用,该设置将于 Go 1.27 移除。改变运行时与 Linux 透明大页的交互方式:常见的 Linux 内核默认配置可能导致显著内存开销,Go 1.22 不再绕过该默认;可用 [`disablethp` 设置](/pkg/runtime#hdr-Environment_Variables)对 Go 内存禁用透明大页,该行为已回移至 Go 1.21.1,但设置自 Go 1.21.6 起可用;受影响的用户应按 [GC 指南](/doc/gc-guide#Linux_transparent_huge_pages)的建议调整 Linux 配置,或改用完全禁用透明大页的发行版。运行时内部锁的争用被纳入 [`mutex` profile](/pkg/runtime/pprof#Profile),这些锁的争用一律记在 `runtime._LostContendedRuntimeLock`,可用 [`runtimecontentionstacks` 设置](/pkg/runtime#hdr-Environment_Variables)启用完整栈回溯(其语义非标准,详见设置文档)。[crypto/x509.Certificate](/pkg/crypto/x509/#Certificate) 新增 [`Policies`](/pkg/crypto/x509/#Certificate.Policies) 字段,支持分量大于 31 位的证书策略 OID;默认仅在解析时填充,编组时不使用;可通过 [`x509usepolicies` 设置](/pkg/crypto/x509/#CreateCertificate)在编组时用它替代现有的 PolicyIdentifiers 字段。

### Go 1.21

`panic` 传入 nil 接口值成为运行时错误,由 [`panicnil` 设置](/pkg/builtin/#panic)控制。html/template 的 action 出现在 ECMAScript 6 模板字面量内时报错,由 [`jstmpllitinterp` 设置](/pkg/html/template#hdr-Security_Model)控制,该行为已回移至 Go 1.19.8+ 与 Go 1.20.3+。新增 MIME 头与 multipart 表单的最大数量限制,分别由 [`multipartmaxheaders` 与 `multipartmaxparts` 设置](/pkg/mime/multipart#hdr-Limits)控制,已回移至 Go 1.19.8+ 与 Go 1.20.3+。新增 Multipath TCP 支持,但仅在应用显式要求时使用,由 [`multipathtcp` 设置](/pkg/net#Dialer.SetMultipathTCP)控制。暂无移除这些设置的计划。

### Go 1.20

新增拒绝 tar 与 zip 归档中不安全路径的支持,分别由 [`tarinsecurepath` 设置](/pkg/archive/tar/#Reader.Next)与 [`zipinsecurepath` 设置](/pkg/archive/zip/#NewReader)控制;默认均为 `1`,保持早期版本行为;未来版本可能将默认改为 `0`。新增 [`math/rand`](/pkg/math/rand) 全局随机数生成器的自动播种,由 [`randautoseed` 设置](/pkg/math/rand/#Seed)控制。新增证书验证用的回退根证书概念,由 [`x509usefallbackroots` 设置](/pkg/crypto/x509/#SetFallbackRoots)控制。Go 发行版不再预装标准库 `.a` 文件,现在与其他模块的包一样构建并缓存标准库;[`installgoroot` 设置](/cmd/go#hdr-Compile_and_install_packages_and_dependencies)可恢复预装 `.a` 文件的安装与使用。暂无移除这些设置的计划。

### Go 1.19

路径查找解析到当前目录中的可执行文件时报错,由 [`execerrdot` 设置](/pkg/os/exec#hdr-Executables_in_the_current_directory)控制,暂无移除计划。DNS 请求开始发送 EDNS0 附加头,据报道会破坏某些路由器(如 CenturyLink Zyxel C3000Z)提供的 DNS 服务器,可由 [`netedns0` 设置](/pkg/net#hdr-Name_Resolution)更改;该设置在 Go 1.21.12、Go 1.22.5、Go 1.23 及以后可用,暂无移除计划。

### Go 1.18

大多数 X.509 证书不再支持 SHA1,由 [`x509sha1` 设置](/pkg/crypto/x509#InsecureAlgorithmError)控制;该设置已于 Go 1.24 移除。

### Go 1.10

改变构建缓存工作方式并新增测试缓存,引入 [`gocacheverify`、`gocachehash`、`gocachetest` 设置](/cmd/go/#hdr-Build_and_test_caching),暂无移除计划。

### Go 1.6

引入对 HTTP/2 的透明支持,引入 [`http2client`、`http2server`、`http2debug` 设置](/pkg/net/http/#hdr-HTTP_2),暂无移除计划。

### Go 1.5

引入纯 Go DNS 解析器,引入 [`netdns` 设置](/pkg/net/#hdr-Name_Resolution),暂无移除计划。
