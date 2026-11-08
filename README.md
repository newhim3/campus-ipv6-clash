# Windows 校园网 IPv6 分流：Clash Verge 配置与验证

适用场景：校园网已能访问 IPv6，手里已有支持 IPv6 入站的代理节点。目标是让国内和教育网域名经校园网 IPv6 直连，其他外网通过 IPv6 接入代理节点，同时把不遵循系统代理的应用交给 Clash。

本教程只配置 Clash。Windows 无线网卡的 IPv4 保持启用，认证页面仍可按学校的正常方式使用。IPv6 连通、代理接管和校园网计费是三项独立验证。

## 1. 先确认校园网和节点

1. 正常连接、认证校园网。PowerShell 运行 `Get-NetRoute -AddressFamily IPv6 -DestinationPrefix '::/0'` 检查 IPv6 默认路由。
2. 在自己的 Clash 配置中查看节点的 `server`。保留使用 IPv6 地址的节点，以及入口域名确实有 AAAA 记录的节点。域名可用 `Resolve-DnsName <节点域名> -Type AAAA` 检查。
3. 给这些节点加 `ip-version: ipv6`，保留节点原有协议、端口、认证和 TLS 参数。网站支持 IPv6 不代表节点的入口支持 IPv6；AAAA 记录存在也还要测试连接。
4. 把这些节点放入一个只含 IPv6 节点的组，不放 `DIRECT` 或 IPv4 节点作为回退。保留原订阅，另建一个本地配置测试。若采用本地快照，节点更新后需重新同步。

## 2. 配置直连和代理分流

### 2.1 从原订阅复制一个节点

1. 打开 Clash Verge 的「订阅」页，右键原订阅，找「编辑文件」或「查看配置」。如果版本没有该入口，用服务商提供的 Clash/Mihomo 格式 YAML 在本地打开。**保留原订阅，不在原文件上修改。**
2. 搜索行首的 `proxies:`。从上一步确认可用的节点里选一个，复制它的**完整条目**：从 `- name:` 到下一个节点之前。若节点写成 `- {name: ..., type: ..., ...}`，就复制整行。地址、端口、协议、密码和 TLS 等字段都照原样保留。
3. 如果源文件只有 `proxy-providers:`，这只是节点集合的引用；需要在本地取得该集合的实际节点定义，不能把供应商名称或代理组名称当节点粘贴。如果节点引用了 YAML 锚点或链式代理，先选一个不依赖其他配置的节点完成首测。

### 2.2 创建本地 YAML，填入真实节点

用记事本新建 `campus-ipv6.yaml`，把下面模板全部复制进去。保存时选「所有文件」、UTF-8，并确认扩展名是 `.yaml`，不是 `.yaml.txt`。

```yaml
mode: rule
ipv6: true
allow-lan: false

dns:
  enable: true
  ipv6: true
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  fake-ip-filter:
    - '*.lan'
    - '*.local'
  default-nameserver:
    - '2400:3200::1'
    - '2606:4700:4700::1111'
  proxy-server-nameserver:
    - '2400:3200::1'
    - '2606:4700:4700::1111'
  direct-nameserver:
    - '2400:3200::1'
    - '2606:4700:4700::1111'
  nameserver-policy:
    'geosite:cn':
      - '2400:3200::1'
    '+.edu.cn':
      - '2400:3200::1'
  nameserver:
    - 'https://1.1.1.1/dns-query#校园IPv6'

proxies:
  - name: 校园IPv6直连
    type: direct
    ip-version: ipv6
    udp: true

  # 用从原订阅复制的完整节点替换以下 5 行。
  - name: 示例节点
    type: socks5
    server: '2001:db8::1'
    port: 1080
    ip-version: ipv6

proxy-groups:
  - name: 校园IPv6
    type: select
    proxies:
      - IPv6自动选择
  - name: IPv6自动选择
    type: url-test
    url: https://www.gstatic.com/generate_204
    interval: 600
    tolerance: 80
    proxies:
      - 示例节点

rules:
  - IP-CIDR,0.0.0.0/0,REJECT,no-resolve
  - DOMAIN-SUFFIX,edu.cn,校园IPv6直连
  - GEOSITE,cn,校园IPv6直连
  - MATCH,校园IPv6
```

只改两处：

1. 把 `proxies:` 下的「示例节点」整个条目换成你复制的真实节点。保留「校园IPv6直连」。确认真实节点有且只有一个 `ip-version: ipv6`：多行写法把它和 `type:` 对齐；单行花括号写法在 `}` 前加 `, ip-version: ipv6`，已有该字段就改值。缩进用空格，不用 Tab。
2. 在「IPv6自动选择」组的 `proxies:` 下，把 `示例节点` 替换为真实节点的 `name`，包含空格、符号和表情在内都要一致。第一轮只放一个节点；成功后可在顶层 `proxies:` 下继续加入节点，并在组内每行列一个名称。不要把 `DIRECT` 加进该组。示例里的 `2001:db8::1` 是文档占位地址，保留它不会连通。

### 2.3 导入并启用

1. 在 Clash Verge「订阅」页把 `campus-ipv6.yaml` 拖入窗口，或点击「新建」，选择「本地」并选中该文件。参见 [Clash Verge Rev 本地配置导入](https://www.clashverge.dev/guide/profile.html#本地配置)。
2. 点选新配置，回首页把代理模式设为「规则」，开启「系统代理」。到「代理」页，在「校园IPv6」组选择「IPv6自动选择」。先用浏览器确认能访问国内网站和外网，再按下一节开启 TUN 测试网盘等客户端。
3. 如果提示节点不存在，核对组内名称和节点的 `name`；如果提示 YAML 解析失败，检查缩进、重复字段和文件扩展名。若导入成功但新规则没生效，检查全局或订阅覆写是否替换了规则或 DNS。网页异常时可切回原订阅。

### 2.4 看懂分流和 DNS

| 进入 Clash 的请求 | 处理方式 |
| --- | --- |
| 目标已知为 IPv4 地址 | 拒绝；该规则不为了匹配而主动解析域名。依赖 IPv4 地址的下载线路可能失败。 |
| `edu.cn` 或 `GEOSITE,cn` 匹配的域名 | 使用「校园IPv6直连」。只有 IPv4 的站点不会回退直连。 |
| 其余请求 | 走「校园IPv6」组。电脑到代理节点使用 IPv6；节点到网站的协议由节点决定。 |

模板的节点域名和国内 DNS 查询使用 IPv6 解析器，其他域名的 DoH 查询通过「校园IPv6」代理。DoH URL 里的 `1.1.1.1` 是代理另一端访问的解析器，并不是本机从校园网直接拨 IPv4。若国内网站打不开，可在 PowerShell 运行 `Resolve-DnsName www.baidu.com -Type AAAA -Server 2400:3200::1` 检查解析器，并将相应条目替换为校园网可达的 IPv6 DNS。若报 `GEOSITE,cn` 加载错误，先检查 Mihomo 的 GeoSite 数据是否可用。字段说明见 [Mihomo DNS 文档](https://wiki.metacubex.one/config/dns/)。

## 3. 启用方式与实测

在 Clash Verge 中选中新配置，保持「规则」模式，在「校园IPv6」组选择「IPv6自动选择」。

| 系统代理 | 虚拟网卡（TUN） | 本次 Windows／百度网盘环境观察 |
| --- | --- | --- |
| 开 | 关 | 网页可用，但部分下载流量仍走 IPv4；曾观察到网盘进程有非本地 IPv4 连接。 |
| 开 | 开 | 测试中观察到的相关流量全部走 IPv6，国内网页可用。这是本机实测可用的组合。 |
| 关 | 开 | 此环境下国内网页打不开，未确认具体原因。若要单独使用 TUN，检查 DNS、TUN 路由和直连规则，再做对照。 |

系统代理只接管遵循它的程序；TUN 接管更多应用。两者同时开启在本机有效，但不能据此保证所有机器都需要同时开，也不能保证所有流量都会被接管。

## 4. 怎么确认走 IPv6、有没有免流

1. 在 Clash「连接」页确认：国内网站走「校园IPv6直连」，外网走「校园IPv6 → IPv6 节点」。
2. 在同一次校园网会话里下载一个小文件，等统计更新后比较前后用量。走 IPv6 不等于免流，以校园网用量为准。

网页异常时，先关闭 TUN、切回原配置恢复访问。本文参考 [NEU-campus-free](https://github.com/SkYFly2233/NEU-campus-free) 的 IPv6 代理接入思路；这里复用已有节点并采用国内 IPv6 直连规则，不需要运行其 VPS 安装脚本。字段参见 [Mihomo 出站配置](https://wiki.metacubex.one/config/proxies/#ip-version) 和 [DIRECT 出站](https://wiki.metacubex.one/config/proxies/direct/)。

