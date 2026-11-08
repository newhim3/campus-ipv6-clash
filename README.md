# Windows 校园网 IPv6 分流：Clash Verge 配置与验证

适用场景：校园网已能访问 IPv6，手里已有支持 IPv6 入站的代理节点。目标是让国内和教育网域名经校园网 IPv6 直连，其他外网通过 IPv6 接入代理节点，同时把不遵循系统代理的应用交给 Clash。

本教程只配置 Clash。Windows 无线网卡的 IPv4 保持启用，认证页面仍可按学校的正常方式使用。IPv6 连通、代理接管和校园网计费是三项独立验证。

## 1. 先确认校园网和节点

1. 正常连接、认证校园网。PowerShell 运行 `Get-NetRoute -AddressFamily IPv6 -DestinationPrefix '::/0'` 检查 IPv6 默认路由。
2. 在自己的 Clash 配置中查看节点的 `server`。保留使用 IPv6 地址的节点，以及入口域名确实有 AAAA 记录的节点。域名可用 `Resolve-DnsName <节点域名> -Type AAAA` 检查。
3. 给这些节点加 `ip-version: ipv6`，保留节点原有协议、端口、认证和 TLS 参数。网站支持 IPv6 不代表节点的入口支持 IPv6；AAAA 记录存在也还要测试连接。
4. 把这些节点放入一个只含 IPv6 节点的组，不放 `DIRECT` 或 IPv4 节点作为回退。保留原订阅，另建一个本地配置测试。若采用本地快照，节点更新后需重新同步。

## 2. 配置直连和代理分流

以下是合并进自己的 YAML 的结构。保留原有的 IPv6 节点定义，将示例节点名称替换为配置里真实存在的名称；它不是一份可直接导入的完整订阅。

```yaml
mode: rule
ipv6: true

proxies:
  - name: 校园IPv6直连
    type: direct
    ip-version: ipv6
    udp: true
  # 在此保留自己的节点定义，每个节点设置 ip-version: ipv6。

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
      - 替换为你的IPv6节点名称

rules:
  - IP-CIDR,0.0.0.0/0,REJECT,no-resolve
  - DOMAIN-SUFFIX,edu.cn,校园IPv6直连
  - GEOSITE,cn,校园IPv6直连
  - MATCH,校园IPv6
```

- `GEOSITE,cn` 依赖 Mihomo 的 GeoSite 数据。它是规则集合，不代表国内的每个网站都支持 IPv6，也不一定涵盖所有国内域名。
- 第一条规则拒绝进入 Clash、且此时已知目标为 IPv4 的连接；`no-resolve` 避免为这条规则主动查询域名。部分写死 IPv4 地址的软件或下载线路会失败。国内直连域名只有 A 记录时，也无法按严格 IPv6 规则直连。
- 外网一跳“电脑 → 代理节点”使用 IPv6；节点到目标网站的协议由节点决定。
- DNS 应开启 `ipv6: true`。如需自定义国内和教育网 DNS，可在 `dns.nameserver-policy` 中把 `'geosite:cn'` 与 `'+.edu.cn'` 指向可达的 IPv6 DNS，例如 `'2400:3200::1'`，并按需用 `direct-nameserver` 指定 IPv6 解析器。先测解析器在自己的校园网可达，再应用；合并配置后检查实际生效的 DNS。TUN 劫持后的虚拟 IPv4 地址不等于无线网卡对外发出了 IPv4 流量。

## 3. 启用方式与实测

在 Clash Verge 中选中新配置，保持「规则」模式，在「校园IPv6」组选择「IPv6自动选择」。

| 系统代理 | 虚拟网卡（TUN） | 本次 Windows／百度网盘环境观察 |
| --- | --- | --- |
| 开 | 关 | 网页可用，但部分下载流量仍走 IPv4；曾观察到网盘进程有非本地 IPv4 连接。 |
| 开 | 开 | 测试中观察到的相关流量全部走 IPv6，国内网页可用。这是本机实测可用的组合。 |
| 关 | 开 | 此环境下国内网页打不开，未确认具体原因。若要单独使用 TUN，检查 DNS、TUN 路由和直连规则，再做对照。 |

系统代理只接管遵循它的程序；TUN 接管更多应用。两者同时开启在本机有效，但不能据此保证所有机器都需要同时开，也不能保证所有流量都会被接管。

## 4. 怎么确认走 IPv6、有没有免流

1. 打开 Clash「连接」页，用国内网站、外网网站和实际下载器各做一次小测试，查看连接匹配规则和代理链。国内预期是「校园IPv6直连」，外网预期是「校园IPv6 → IPv6 节点」。若下载器没出现在连接页，不能算已接管。
2. 要验证真正出网的协议，在自己的电脑上抓取**物理 WLAN 网卡**的测试流量，核对直连目的地址和到代理节点的目的地址。仅看系统 IPv6 开关、浏览器出口 IP，或 TUN 内部地址都不足以确认。
3. 在同一校园网会话中记录用量基线，暂停其他下载，用同一个客户端下载几十 MB 的文件，等待统计更新，再记录用量差值。IPv6 可以连通不代表一定免流；机场订阅也有独立用量。不要把一次大文件下载与校园网增加的用量直接相减，当作已免流字节数。
4. 未登录时若节点全部超时，即使电脑仍有 IPv6 地址和默认路由，也无法用这套配置证明免登录。该案例在未登录时测试的 TCP、UDP 代理节点均未连通。

网页异常时，先关闭 TUN、切回原配置恢复访问。本文参考 [NEU-campus-free](https://github.com/SkYFly2233/NEU-campus-free) 的 IPv6 代理接入思路；这里复用已有节点并采用国内 IPv6 直连规则，不需要运行其 VPS 安装脚本。字段参见 [Mihomo 出站配置](https://wiki.metacubex.one/config/proxies/#ip-version) 和 [DIRECT 出站](https://wiki.metacubex.one/config/proxies/direct/)。

