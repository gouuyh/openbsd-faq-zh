## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 网络地址转换 (NAT)</span>

---

- <a href="#intro">简介</a>
- <a href="#works">NAT 如何工作</a>
- <a href="#ipfwd">IP 转发</a>
- <a href="#config">配置 NAT</a>
- <a href="#binat">双向映射 (1:1 映射)</a>
- <a href="#except">转换规则例外</a>
- <a href="#status">检查 NAT 状态</a>

---

<h2 id="intro">简介</h2>

网络地址转换 (NAT) 是一种将整个网络（或多个网络）映射到单个 IP 地址的方法。例如，当互联网服务提供商分配给客户的 IP 地址数量少于该家庭中需要互联网访问的计算机总数时，就需要使用 NAT。NAT 在 <a href="https://tools.ietf.org/html/rfc1631">RFC 1631</a> 中描述。

NAT 允许管理员利用 <a href="https://tools.ietf.org/html/rfc1918">RFC 1918</a> 中描述的保留地址块。通常，内部网络将设置为使用这些网络块中的一个或多个：

```text
10.0.0.0/8       (10.0.0.0 - 10.255.255.255)
172.16.0.0/12    (172.16.0.0 - 172.31.255.255)
192.168.0.0/16   (192.168.0.0 - 192.168.255.255)
```

执行 NAT 的 OpenBSD 系统将至少有两个网络接口：一个连接到互联网，另一个连接到内部网络。NAT 将转换来自内部网络的请求，使它们看起来都像是来自 OpenBSD NAT 系统本身。

<h2 id="works">NAT 如何工作</h2>

当内部网络上的客户端联系互联网上的机器时，它会发送发往该机器的 IP 数据包。这些数据包包含将其传送到目的地所需的所有寻址信息。NAT 关注这些信息片段：源 IP 地址和源 TCP 或 UDP 端口。

当数据包通过 NAT 网关时，它们将被修改，以便看起来像是来自 NAT 网关本身。NAT 网关将在其状态表中记录所做的更改，以便它可以反转返回数据包上的更改，并确保返回数据包通过防火墙而不被阻止。例如，可能会进行以下更改：

- 源 IP：替换为网关的外部地址
- 源端口：替换为网关上随机选择的未使用端口

内部机器和互联网主机都不知道这些转换步骤。对于内部机器，NAT 系统只是一个互联网网关。对于互联网主机，数据包似乎直接来自 NAT 系统。它完全不知道内部工作站的存在。

当互联网主机回复内部机器的数据包时，它们将被寻址到 NAT 网关外部 IP 的转换端口。然后，NAT 网关将搜索状态表以确定回复数据包是否匹配已建立的连接。将基于 IP/端口组合找到唯一的匹配项，这告诉 PF 数据包属于内部机器发起的连接。然后，PF 将进行与对传出数据包所做的相反的更改，并将回复数据包转发给内部机器。

ICMP 数据包的转换以类似的方式发生，但不修改源端口。

<h2 id="ipfwd">IP 转发</h2>

由于 NAT 几乎总是用于路由器和网络网关，因此可能需要启用 IP 转发，以便数据包可以在 OpenBSD 机器上的网络接口之间传输。使用 <a href="https://man.openbsd.org/sysctl.2">sysctl(2)</a> 机制启用 IP 转发：

```shell
# sysctl net.inet.ip.forwarding=1
# echo 'net.inet.ip.forwarding=1' >> /etc/sysctl.conf
```

或者，对于 IPv6：

```shell
# sysctl net.inet6.ip6.forwarding=1
# echo 'net.inet6.ip6.forwarding=1' >> /etc/sysctl.conf
```

<h2 id="config">配置 NAT</h2>

NAT 被指定为出站 `pass` 规则的可选 `nat-to` 参数。通常，不是直接在 `pass` 规则上设置，而是使用 `match` 规则。当数据包被 `match` 规则选中时，该规则中的参数（例如 `nat-to`）会被记住，并在达到匹配该数据包的 `pass` 规则时应用于该数据包。这允许通过单个 `match` 规则处理整类数据包，然后使用 `block` 和 `pass` 规则就是否允许流量做出具体决定。

`pf.conf` 中的一般格式如下所示：

```text
match out on interface [af] \
   from src_addr to dst_addr \
   nat-to ext_addr [pool_type] [static-port]
[...]
pass out [log] on interface [af] [proto protocol] \
   from ext_addr [port src_port] \
   to dst_addr [port dst_port]
```

- `match`
  <br>当数据包遍历规则集并匹配 `match` 规则时，该规则中指定的任何可选参数都会被记住以供将来使用（使其具有“粘性”）。

- `pass`
  <br>此规则允许传输数据包。如果数据包之前被指定了参数的 `match` 规则匹配，这些参数将应用于此数据包。`pass` 规则可能有自己的参数；这些参数优先于 `match` 规则中指定的参数。

- `out`
  <br>指定此规则适用的数据包流向。`nat-to` 只能为出站数据包指定。

- `log`
  <br>通过 <a href="https://man.openbsd.org/pflogd">pflogd(8)</a> 记录匹配的数据包。通常，只记录第一个匹配的数据包。要记录所有匹配的数据包，请使用 `log (all)`。

- `interface`
  <br>传输数据包的网络接口的名称或组。

- `af`
  <br>地址族，`inet` 用于 IPv4 或 `inet6` 用于 IPv6。PF 通常能够根据源/目标地址确定此参数。

- `protocol`
  <br>允许的数据包协议（例如 tcp, udp, icmp）。如果指定了 *src_port* 或 *dst_port*，则 *必须* 同时给出协议。

- `src_addr`
  <br>将被转换的数据包的源（内部）地址。源地址可以指定为：
    - 单个 IPv4 或 IPv6 地址。
    - <a href="https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html">CIDR</a> 网络块。
    - 完全限定域名 (FQDN)，将在加载规则集时通过 DNS 解析。所有生成的 IP 地址都将替换到规则中。
    - 网络接口的名称或组。分配给接口的任何 IPv4 和 IPv6 地址将在加载时替换到规则中。
    - 网络接口名称后跟 `/netmask`（例如 `/24`）。接口上的每个 IP 地址都与子网掩码组合形成 CIDR 网络块，该网络块将替换到规则中。
    - 网络接口的名称或组后跟以下任何一个修饰符：
        - `:network` - 替换 CIDR 网络块（例如，192.168.0.0/24）
        - `:broadcast` - 替换网络广播地址（例如，192.168.0.255）
        - `:peer` - 替换点对点链路上的对等方 IP 地址
        <br>此外，`:0` 修饰符可以附加到接口名称/组或上述任何修饰符，以指示 PF 不应在替换中包含别名 IP 地址。当接口包含在括号中时，也可以使用这些修饰符。示例：`fxp0:network:0`
    - <a href="./03_tables.md">表 (table)</a>。
    - 上述任何一项，但使用 `!`（“非”）修饰符进行否定。
    - 使用<a href="./02_macros.md#lists">列表</a>的一组地址。
    - 关键字 `any` 表示所有地址。

- `src_port`
  <br>第 4 层数据包报头中的源端口。端口可以指定为：
    - 1 到 65535 之间的数字
    - 来自 <a href="https://man.openbsd.org/services">services(5)</a> 的有效服务名称
    - 使用<a href="./02_macros.md#lists">列表</a>的一组端口
    - 范围：
        - `!=`（不等于）
        - `<`（小于）
        - `>`（大于）
        - `<=`（小于或等于）
        - `>=`（大于或等于）
        - `><`（范围）
        - `<>`（反向范围）
        <br>最后两个是二元运算符（它们接受两个参数），并且不包括范围内的参数。
        - `:`（包含范围）
        <br>包含范围运算符也是二元运算符，并且包括范围内的参数。

`port` 选项通常不用于 `nat` 规则，因为目标通常是对所有流量进行 NAT，而不管使用的是哪个端口。

- `dst_addr`
  <br>要转换的数据包的目标地址。目标地址的指定方式与源地址相同。

- `dst_port`
  <br>第 4 层数据包报头中的目标端口。此端口的指定方式与源端口相同。

- `ext_addr`
  <br>NAT 网关上的外部（转换）地址，数据包将被转换到该地址。外部地址可以指定为：
    - 单个 IPv4 或 IPv6 地址。
    - <a href="https://web.archive.org/web/20150611113606/http://public.swbell.net/dedicated/cidr.html">CIDR</a> 网络块。
    - 完全限定域名 (FQDN)，将在加载规则集时通过 DNS 解析。
    - 外部网络接口的名称或组。分配给接口的任何 IP 地址将在加载时替换到规则中。
    - 括号 `( )` 中的外部网络接口的名称或组。这告诉 PF 如果命名接口上的 IP 地址发生变化，则更新规则。当外部接口通过 DHCP 或拨号获取其 IP 地址时，这非常有用，因为每次地址更改时都不必重新加载规则集。
    - 网络接口的名称或组后跟以下任一修饰符：
        - `:network` - 替换 CIDR 网络块（例如，192.168.0.0/24）
        - `:peer` - 替换点对点链路上的对等方 IP 地址
        <br>此外，`:0` 修饰符可以附加到接口名称/组或上述任何修饰符，以指示 PF 不应在替换中包含别名 IP 地址。当接口包含在括号中时，也可以使用这些修饰符。示例：`fxp0:network:0`
    - 使用<a href="./02_macros.md#lists">列表</a>的一组地址。

- `pool_type`
  <br>指定用于转换的<a href="./10_pools.md">地址池</a>类型。

- `static-port`
  <br>告诉 PF 不要转换 TCP 和 UDP 数据包中的源端口。

这会导致这些行的最基本形式类似于：

```text
match out on tl0 from 192.168.1.0/24 to any nat-to 198.51.100.1
pass on tl0 from 192.168.1.0/24 to any
```

或者可以使用以下内容：

```text
pass out on tl0 from 192.168.1.0/24 to any nat-to 198.51.100.1
```

此规则表示对来自 192.168.1.0/24 的任何数据包在 `tl0` 接口上执行 NAT，并将源 IP 地址替换为 198.51.100.1。

虽然上述规则是正确的，但不推荐这种形式。维护可能会很困难，因为任何外部或内部网络号码的更改都需要更改该行。相反，与此更易于维护的行进行比较（`tl0` 是外部，`dc0` 是内部）：

```text
pass out on tl0 inet from dc0:network to any nat-to tl0
```

优点应该相当明显：可以更改任一接口的 IP 地址而无需更改此规则。请注意，在这种情况下应指定 `inet` 以确保仅使用 IPv4 地址，避免意外惊喜。

如上所述指定用于转换地址的接口名称时，IP 地址是在 pf.conf *加载* 时确定的，而不是动态确定的。如果使用 DHCP 配置外部接口，这可能是一个问题。如果分配的 IP 地址发生变化，NAT 将继续使用旧 IP 地址转换传出数据包。这将导致传出连接停止工作。为了解决这个问题，PF 可以通过在接口名称周围加上括号来自动更新转换地址：

```text
pass out on tl0 inet from dc0:network to any nat-to (tl0)
```

此方法适用于转换为 IPv4 和 IPv6 地址。

<h2 id="binat">双向映射 (1:1 映射)</h2>

可以使用 `binat-to` 参数建立双向映射。`binat-to` 规则在内部 IP 地址和外部地址之间建立一对一映射。例如，这对于为内部网络上的 Web 服务器提供其自己的外部 IP 地址很有用。从互联网到外部地址的连接将被转换为内部地址，来自 Web 服务器的连接（如 DNS 请求）将被转换为外部地址。TCP 和 UDP 端口永远不会像 `nat` 规则那样被 `binat-to` 规则修改。

示例：

```text
web_serv_int = "192.168.1.100"
web_serv_ext = "198.51.100.6"

pass on tl0 from $web_serv_int to any binat-to $web_serv_ext
```

<h2 id="except">转换规则例外</h2>

如果在某些情况下需要提供 NAT 规则的例外，请确保例外由不包含 `nat-to` 参数的过滤规则处理。例如，如果上面的 NAT 示例修改为如下所示：

```text
pass out on tl0 from 192.168.1.0/24 to any nat-to 198.51.100.79
pass out on tl0 from 192.168.1.208  to any
```

那么整个 192.168.1.0/24 网络的数据包都将被转换为外部地址 198.51.100.79，但 192.168.1.208 除外。

<h2 id="status">检查 NAT 状态</h2>

要查看活动的 NAT 转换，使用带有 `-s state` 选项的 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a>。此选项将列出所有当前的 NAT 会话：

```shell
# pfctl -s state
fxp0 tcp 192.168.1.35:2132 (198.51.100.1:53136) -> 198.51.100.10:22 TIME_WAIT:TIME_WAIT
fxp0 udp 192.168.1.35:2491 (198.51.100.1:60527) -> 198.51.100.33:53   MULTIPLE:SINGLE
```

解释（仅第一行）：

- **fxp0**
  <br>指示状态绑定的接口。如果状态是 <a href="./08_options.md#state-policy">`floating`</a>，则会出现单词 `self`。

- **TCP**
  <br>连接使用的协议。

- **192.168.1.35:2132**
  <br>内部网络上机器的 IP 地址 (192.168.1.35)。源端口 (2132) 显示在地址之后。这也是在 IP 报头中被替换的地址。

- **198.51.100.1:53136**
  <br>网关上的 IP 地址 (198.51.100.1) 和端口 (53136)，数据包被转换到该地址。

- **198.51.100.10:22**
  <br>内部机器正在连接的 IP 地址 (198.51.100.10) 和端口 (22)。

- **TIME_WAIT:TIME_WAIT**
  <br>这表示 PF 认为 TCP 连接处于什么状态。
