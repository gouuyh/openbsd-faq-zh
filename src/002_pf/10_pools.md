## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 地址池和负载平衡</span>

---

- <a href="#intro">简介</a>
- <a href="#nat">NAT 地址池</a>
- <a href="#incoming">传入连接的负载平衡</a>
- <a href="#outgoing">传出流量的负载平衡</a>
    - <a href="#outexample">规则集示例</a>

---

<h2 id="intro">简介</h2>

地址池是两个或多个地址的供应，其使用在一组用户之间共享。它可以指定为 `nat-to`、`rdr-to`、`route-to`、`reply-to` 和 `dup-to` <a href="./04_filter.md">过滤</a>选项中的目标地址。

有四种使用地址池的方法：

- `bitmask` - 将池地址的网络部分移植到正在修改的地址（`nat-to` 规则的源地址，`rdr-to` 规则的目标地址）之上。例如：如果地址池是 192.0.2.1/24，正在修改的地址是 10.0.0.50，则结果地址将是 192.0.2.50。如果地址池是 192.0.2.1/25，正在修改的地址是 10.0.0.130，则结果地址将是 192.0.2.2。
- `random` - 从池中随机选择一个地址。
- `source-hash` - 使用源地址的哈希值来确定从池中使用哪个地址。此方法确保给定的源地址始终映射到相同的池地址。可以在 `source-hash` 关键字之后以十六进制格式或字符串形式指定提供给哈希算法的密钥。默认情况下，<a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 将在每次加载规则集时生成一个随机密钥。
- `round-robin` - 按顺序循环遍历地址池。这是默认方法，也是使用<a href="./03_tables.md">表</a>指定地址池时允许的唯一方法。

除了 `round-robin` 方法外，地址池必须表示为 <a href="https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html">CIDR</a> (无类别域间路由) 网络块。`round-robin` 方法将接受使用<a href="./02_macros.md#lists">列表</a>或<a href="./03_tables.md">表</a>指定的多个单独地址。

`sticky-address` 选项可以与 `random` 和 `round-robin` 池类型一起使用，以确保特定源地址始终映射到相同的重定向地址。

<h2 id="nat">NAT 地址池</h2>

地址池可以用作 <a href="./05_nat.md">`nat-to`</a> 规则中的转换地址。连接将根据所选方法将其源地址转换为池中的地址。这在 PF 为非常大的网络执行 NAT 的情况下很有用。由于每个转换地址的 NAT 连接数是有限的，添加额外的转换地址将允许 NAT 网关扩展以服务更多的用户。

在此示例中，两个地址的池用于转换传出数据包。对于每个传出连接，PF 将以轮询方式轮换地址。

```text
match out on egress inet nat-to { 192.0.2.5, 192.0.2.10 }
```

此方法的一个缺点是，来自同一内部地址的连续连接并不总是转换为相同的转换地址。这可能会造成干扰，例如，在浏览根据 IP 地址跟踪用户登录的网站时。另一种方法是使用 `source-hash` 方法，以便每个内部地址始终转换为相同的转换地址。为此，地址池必须是 <a href="https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html">CIDR</a> 网络块。

```text
match out on egress inet nat-to 192.0.2.4/31 source-hash
```

此规则使用地址池 192.0.2.4/31 (192.0.2.4 - 192.0.2.5) 作为传出数据包的转换地址。由于 `source-hash` 关键字，每个内部地址将始终转换为相同的转换地址。

<h2 id="incoming">传入连接的负载平衡</h2>

地址池也可用于对传入连接进行负载平衡。例如，传入的 Web 服务器连接可以分布在 Web 服务器场中：

```text
web_servers = "{ 10.0.0.10, 10.0.0.11, 10.0.0.13 }"

match in on egress proto tcp to port 80 rdr-to $web_servers \
    round-robin sticky-address
```

连续的连接将以轮询方式重定向到 Web 服务器，来自同一源的连接将被发送到同一台 Web 服务器。只要存在引用此连接的状态，这种“粘性连接”就会存在。一旦状态过期，粘性连接也会过期。来自该主机的进一步连接将被重定向到轮询中的下一台 Web 服务器。

<h2 id="outgoing">传出流量的负载平衡</h2>

当适当的多路径路由协议（如 <a href="https://www.rfc-editor.org/rfc/rfc1771.txt">BGP4</a>）不可用时，地址池可以与 `route-to` 过滤选项结合使用，以对两个或多个互联网连接进行负载平衡。通过将 `route-to` 与 `round-robin` 地址池一起使用，传出连接可以在多个出站路径之间均匀分布。

为此还需要的一条信息是每个互联网连接上相邻路由器的 IP 地址。这被馈送到 `route-to` 选项以控制传出数据包的目的地。

以下示例在两个互联网连接之间平衡传出流量：

```text
lan_net = "192.168.0.0/24"
int_if  = "dc0"
ext_if1 = "fxp0"
ext_if2 = "fxp1"
ext_gw1 = "198.51.100.100"
ext_gw2 = "203.0.113.200"

pass in on $int_if from $lan_net route-to \
   { $ext_gw1 $ext_gw2 } round-robin
```

`route-to` 选项用于 *内部* 接口上 *传入* 的流量，以指定流量将在其间平衡的传出网络网关。请注意，`route-to` 选项必须出现在要平衡流量的 *每个* 过滤规则上（它不能与 `match` 规则一起使用）。

为了确保源地址属于 `$ext_if1` 的数据包始终路由到 `$ext_gw1`（对于 `$ext_if2` 和 `$ext_gw2` 也是如此），规则集中应包含以下两行：

```text
pass out on $ext_if1 from $ext_if2 route-to $ext_gw2
pass out on $ext_if2 from $ext_if1 route-to $ext_gw1
```

最后，也可以在每个传出接口上使用 NAT：

```text
match out on $ext_if1 from $lan_net nat-to ($ext_if1)
match out on $ext_if2 from $lan_net nat-to ($ext_if2)
```

<h3 id="outexample">规则集示例</h3>

一个平衡传出流量的完整示例如下所示：

```text
lan_net = "192.168.0.0/24"
int_if  = "dc0"
ext_if1 = "fxp0"
ext_if2 = "fxp1"
ext_gw1 = "198.51.100.100"
ext_gw2 = "203.0.113.200"

# 在每个互联网接口上对传出连接进行 nat
match out on $ext_if1 from $lan_net nat-to ($ext_if1)
match out on $ext_if2 from $lan_net nat-to ($ext_if2)

# 默认拒绝
block in
block out

# 允许内部接口上的所有传出数据包
pass out on $int_if to $lan_net
# 快速允许任何发往网关本身的数据包
pass in quick on $int_if from $lan_net to $int_if
# 对来自内部网络的传出流量进行负载平衡
pass in on $int_if from $lan_net \
    route-to { $ext_gw1 $ext_gw2 } round-robin
# 将 https 流量保持在单个连接上；某些 Web 应用程序，
# 特别是“安全”应用程序，不允许在会话中途更改
pass in on $int_if proto tcp from $lan_net to port https \
    route-to $ext_gw1

# 外部接口的一般 "pass out" 规则
pass out on $ext_if1
pass out on $ext_if2

# 将来自 $ext_if1 上任何 IP 的数据包路由到 $ext_gw1，
# 对 $ext_if2 和 $ext_gw2 同理
pass out on $ext_if1 from $ext_if2 route-to $ext_gw2
pass out on $ext_if2 from $ext_if1 route-to $ext_gw1
```
