## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 构建一个路由</span>

---

- <a href="#overview">概览</a>
- <a href="#net">网络</a>
- <a href="#dhcp">DHCP</a>
- <a href="#pf">防火墙</a>
- <a href="#dns">DNS</a>

---

<h2 id="overview">概览</h2>

本例将演示如何将 OpenBSD 系统转变为执行以下职责的路由器：

- 网络地址转换 (NAT)
- 通过 DHCP 向客户端分发 IP 地址
- 允许传入连接到本地 Web 服务器
- 为 LAN 执行 DNS 缓存
- 提供无线连接（需要<a href="../001_faq/07_networking.md#Wireless">受支持的网卡</a>）

将使用两个有线 <a href="https://man.openbsd.org/em">em(4)</a> 网卡和一个 <a href="https://man.openbsd.org/athn">athn(4)</a> 无线网卡，最终目标如下所示：

```text
 [ comp1 ] ---+
              |
 [ comp2 ] ---+--- [ 交换机 ] --- em1 [ OpenBSD ] em0 --- [ 互联网 ]
              |                  athn0
 [ comp3 ] ---+                        )))))
                                       ((((( [ comp4 ]
```

请根据实际情况替换 `em0`、`em1` 和 `athn0` 接口名称。

<h2 id="net">网络</h2>

<a href="https://man.openbsd.org/hostname.if">网络配置</a>将为有线客户端使用 192.168.1.0/24 子网，为无线客户端使用 192.168.2.0/24 子网。

```shell
# echo 'net.inet.ip.forwarding=1' >> /etc/sysctl.conf
# echo 'inet autoconf' > /etc/hostname.em0 # 或者使用静态 IP
# echo 'inet 192.168.1.1 255.255.255.0 192.168.1.255' > /etc/hostname.em1
# vi /etc/hostname.athn0
```

添加以下内容，如果需要，更改模式/频道：

```text
media autoselect mode 11n mediaopt hostap chan 1
nwid AccessPointName wpakey VeryLongPassword
inet 192.168.2.1 255.255.255.0
```

OpenBSD 默认在 HostAP 模式下仅允许 WPA2-CCMP 连接。如果需要支持较旧的（不安全的）协议，必须<a href="https://man.openbsd.org/ifconfig#IEEE_802.11_(WIRELESS_DEVICES)">显式启用</a>它们。

<h2 id="dhcp">DHCP</h2>

应在引导时启动 <a href="https://man.openbsd.org/dhcpd">dhcpd(8)</a> 守护进程，以便为客户端机器提供 IP 地址。

```shell
# rcctl enable dhcpd
# rcctl set dhcpd flags em1 athn0
# vi /etc/dhcpd.conf
```

以下 <a href="https://man.openbsd.org/dhcpd.conf">dhcpd.conf(5)</a> 示例还根据 MAC 地址为笔记本电脑和服务器提供静态 IP 预留。

```text
subnet 192.168.1.0 netmask 255.255.255.0 {
	option routers 192.168.1.1;
	option domain-name-servers 192.168.1.1;
	range 192.168.1.4 192.168.1.254;
	host myserver {
		fixed-address 192.168.1.2;
		hardware ethernet 00:00:00:00:00:00;
	}
	host mylaptop {
		fixed-address 192.168.1.3;
		hardware ethernet 11:11:11:11:11:11;
	}
}

subnet 192.168.2.0 netmask 255.255.255.0 {
	option routers 192.168.2.1;
	option domain-name-servers 192.168.2.1;
	range 192.168.2.2 192.168.2.254;
}
```

任何 <a href="https://tools.ietf.org/html/rfc1918">RFC 1918</a> 地址空间都可以在此处指定。本例中的 `domain-name-servers` 行指定了一个将在稍后部分配置的本地 DNS 服务器。

<h2 id="pf">防火墙</h2>

OpenBSD 的 PF 防火墙通过 <a href="https://man.openbsd.org/pf.conf">pf.conf(5)</a> 文件进行配置。强烈建议在复制此示例之前熟悉它以及 PF 的一般知识。

```shell
# vi /etc/pf.conf
```

网关配置可能如下所示：

```text
wired = "em1"
wifi  = "athn0"
table <martians> { 0.0.0.0/8 10.0.0.0/8 127.0.0.0/8 169.254.0.0/16     \
	 	   172.16.0.0/12 192.0.0.0/24 192.0.2.0/24 224.0.0.0/3 \
	 	   192.168.0.0/16 198.18.0.0/15 198.51.100.0/24        \
	 	   203.0.113.0/24 }
set block-policy drop
set loginterface egress
set skip on lo0
match in all scrub (no-df random-id max-mss 1440)
match out on egress inet from !(egress:network) to any nat-to (egress:0)
antispoof quick for { egress $wired $wifi }
block in quick on egress from <martians> to any
block return out quick on egress from any to <martians>
block all
pass out quick inet
pass in on { $wired $wifi } inet
pass in on egress inet proto tcp from any to (egress) port { 80 443 } rdr-to 192.168.1.2
```

现在将解释规则集的各个部分：

```text
wired = "em1"
wifi  = "athn0"
```

LAN 的有线和无线接口名称使用<a href="./02_macros.md">宏</a>定义，用于使整体维护更容易。定义后，可以在整个规则集中引用宏。

```text
table <martians> { 0.0.0.0/8 10.0.0.0/8 127.0.0.0/8 169.254.0.0/16     \
	 	   172.16.0.0/12 192.0.0.0/24 192.0.2.0/24 224.0.0.0/3 \
	 	   192.168.0.0/16 198.18.0.0/15 198.51.100.0/24        \
	 	   203.0.113.0/24 }
```

这是一个稍后将使用的不可路由私有地址的<a href="./03_tables.md">表</a>。

```text
set block-policy drop
set loginterface egress
set skip on lo0
```

PF 允许在运行时设置某些<a href="./08_options.md">选项</a>。`block-policy` 决定被拒绝的数据包是应该返回 TCP RST 还是被静默丢弃。`loginterface` 指定哪个接口应启用数据包和字节统计信息收集。可以使用 `pfctl -si` 命令查看这些统计信息。

在这种情况下，使用的是 `egress` <a href="https://man.openbsd.org/ifconfig#group">组</a>而不是特定的接口名称。通过这样做，将自动选择持有默认路由的接口（`em0`）。最后，`skip` 允许从数据包处理中省略给定的接口。防火墙将忽略 <a href="https://man.openbsd.org/lo">lo(4)</a> 环回接口上的流量。

```text
match in all scrub (no-df random-id max-mss 1440)
match out on egress inet from !(egress:network) to any nat-to (egress:0)
```

这里使用的 `match` 规则完成了两件事：规范化传入数据包，并在 LAN 和公共互联网之间的 `egress` 接口上执行网络地址转换。有关 `match` 规则及其不同选项的更详细解释，请参阅 <a href="https://man.openbsd.org/pf.conf#match">pf.conf(5)</a> 手册。

```text
antispoof quick for { egress $wired $wifi }
block in quick on egress from <martians> to any
block return out quick on egress from any to <martians>
```

<a href="./04_filter.md#antispoof">antispoof</a> 关键字提供了一些针对具有欺骗源地址的数据包的保护。如果传入数据包看起来来自前面定义的不可路由地址列表，则应将其丢弃。此类数据包可能是由于配置错误发送的，或者可能是欺骗攻击的一部分。同样，客户端不应尝试连接到此类地址。指定 "return" 操作是为了防止用户遇到烦人的超时。请注意，如果路由器本身也在 NAT 后面，这可能会导致问题。

```text
block all
```

防火墙将为所有流量设置“默认拒绝”策略，这意味着只允许规则集中明确放入的传入和传出连接。

```text
pass out quick inet
```

允许来自网关本身和 LAN 客户端的传出 IPv4 流量。

```text
pass in on { $wired $wifi } inet
```

允许内部 LAN 流量。

```text
pass in on egress inet proto tcp from any to (egress) port { 80 443 } rdr-to 192.168.1.2
```

将传入连接（在 TCP 端口 80 和 443 上，用于 Web 服务器）转发到 192.168.1.2 处的机器。这仅仅是端口转发的一个例子。

<h2 id="dns">DNS</h2>

虽然对于网关系统来说 DNS 缓存不是必需的，但它是常见的附加功能。当客户端发出 DNS 查询时，它们首先会命中 <a href="https://man.openbsd.org/unbound">unbound(8)</a> 缓存。如果没有答案，它会转到上游解析器。然后将结果反馈给客户端并缓存一段时间，从而加快同一地址的未来查找速度。

```shell
# rcctl enable unbound
# vi /var/unbound/etc/unbound.conf
```

类似这样的配置应该适用于大多数设置：

```text
server:
	interface: 192.168.1.1
	interface: 192.168.2.1
	interface: 127.0.0.1
	access-control: 192.168.1.0/24 allow
	access-control: 192.168.2.0/24 allow
	do-not-query-localhost: no
	hide-identity: yes
	hide-version: yes
	prefetch: yes

forward-zone:
        name: "."
        forward-addr: X.X.X.X  # 首选上游解析器的 IP
```

更多配置选项可以在 <a href="https://man.openbsd.org/unbound.conf">unbound.conf(5)</a> 手册中找到。传出的 DNS 查找也可以使用 dnscrypt-proxy 软件包或 unbound 内置的 DNS over TLS 支持进行加密。

如果路由器也应该使用缓存解析器，其 `/etc/resolv.conf` 文件应包含：

```text
nameserver 127.0.0.1
```

更改到位后，重启系统。
