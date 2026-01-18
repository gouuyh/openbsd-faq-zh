## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 创建规则集的捷径</span>

---

- <a href="#intro">简介</a>
- <a href="#macros">使用宏</a>
- <a href="#lists">使用列表</a>
- <a href="#grammar">PF 语法</a>
    - <a href="#elim">关键字消除</a>
    - <a href="#return"><code>Return</code> 简化</a>
    - <a href="#order">关键字顺序</a>

---

<h2 id="intro">简介</h2>

PF 提供了许多简化规则集的方法。一些很好的例子是使用<a href="./02_macros.md#macros">宏</a>和<a href="./02_macros.md#lists">列表</a>。此外，规则集语言或语法也提供了一些使规则集更简单的捷径。作为一般经验法则，规则集越简单，就越容易理解和维护。

<h2 id="macros">使用宏</h2>

宏很有用，因为它们提供了一种替代方法，可以将地址、端口号、接口名称等硬编码到规则集中。服务器的 IP 地址更改了吗？没问题，只需更新宏；无需弄乱过滤规则。

PF 规则集中的一个常见约定是为每个网络接口定义一个宏。如果网卡需要更换为使用不同驱动程序的网卡，可以更新宏，过滤规则将像以前一样运行。另一个好处是在多台机器上安装相同的规则集时。某些机器可能有不同的网卡，使用宏定义网络接口允许以最少的编辑安装规则集。建议使用宏来定义规则集中可能更改的信息，例如端口号、IP 地址和接口名称。

```text
# 为每个网络接口定义宏
IntIF = "dc0"
ExtIF = "fxp0"
DmzIF = "fxp1"
```

另一个常见的约定是使用宏来定义 IP 地址和网络块。当 IP 地址更改时，这可以大大减少规则集的维护。

```text
# 定义我们的网络
IntNet = "192.168.0.0/24"
ExtAdd = "192.0.2.4"
DmzNet = "10.0.0.0/24"
```

如果内部网络扩展或重新编号为不同的 IP 块，可以更新宏：

```text
IntNet = "{ 192.168.0.0/24, 192.168.1.0/24 }"
```

重新加载规则集后，一切将像以前一样工作。

<h2 id="lists">使用列表</h2>

以下示例是一组用于处理 <a href="https://tools.ietf.org/html/rfc1918">RFC 1918</a> 地址的规则，这些地址不应在互联网上浮动。当它们出现时，通常是试图制造麻烦。

```text
block in  quick on egress inet from 127.0.0.0/8 to any
block in  quick on egress inet from 192.168.0.0/16 to any
block in  quick on egress inet from 172.16.0.0/12 to any
block in  quick on egress inet from 10.0.0.0/8 to any
block out quick on egress inet from any to 127.0.0.0/8
block out quick on egress inet from any to 192.168.0.0/16
block out quick on egress inet from any to 172.16.0.0/12
block out quick on egress inet from any to 10.0.0.0/8
```

现在看看下面的简化：

```text
block in quick  on egress inet from { 127.0.0.0/8, 192.168.0.0/16, \
   172.16.0.0/12, 10.0.0.0/8 } to any
block out quick on egress inet from any to { 127.0.0.0/8, \
   192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 }
```

规则集已从八行减少到两行。当宏与列表结合使用时，情况会变得更好：

```text
NoRouteIPs = "{ 127.0.0.0/8, 192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 }"

block in  quick on egress from $NoRouteIPs to any
block out quick on egress from any to $NoRouteIPs
```

宏和列表简化了 `pf.conf` 文件，但这些行实际上被 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 扩展为多条规则。因此，上面的示例实际上扩展为以下规则：

```text
block in  quick on egress inet from 127.0.0.0/8 to any
block in  quick on egress inet from 192.168.0.0/16 to any
block in  quick on egress inet from 172.16.0.0/12 to any
block in  quick on egress inet from 10.0.0.0/8 to any
block out quick on egress inet from any to 10.0.0.0/8
block out quick on egress inet from any to 172.16.0.0/12
block out quick on egress inet from any to 192.168.0.0/16
block out quick on egress inet from any to 127.0.0.0/8
```

PF 扩展纯粹是为了方便 `pf.conf` 文件的编写者和维护者，而不是对 <a href="https://man.openbsd.org/pf">pf(4)</a> 处理的规则的实际简化。

宏不仅仅可以用来定义地址和端口；它们可以在 PF 规则文件的任何地方使用。

```text
pre  = "pass in quick on ep0 inet proto tcp from "
post = "to any port { 80, 6667 }"

$pre 198.51.100.80 $post
$pre 203.0.113.79  $post
$pre 203.0.113.178 $post
```

扩展为：

```text
pass in quick on ep0 inet proto tcp from 198.51.100.80 to any port = 80
pass in quick on ep0 inet proto tcp from 198.51.100.80 to any port = 6667
pass in quick on ep0 inet proto tcp from 203.0.113.79  to any port = 80
pass in quick on ep0 inet proto tcp from 203.0.113.79  to any port = 6667
pass in quick on ep0 inet proto tcp from 203.0.113.178 to any port = 80
pass in quick on ep0 inet proto tcp from 203.0.113.178 to any port = 6667
```

<h2 id="grammar">PF 语法</h2>

PF 的语法非常灵活，这反过来允许规则集具有很大的灵活性。PF 能够推断某些关键字，这意味着不必在规则中明确说明它们，并且关键字顺序是宽松的，因此不必记住严格的语法。

<h3 id="elim">关键字消除</h3>

要定义“默认拒绝”策略，使用两条规则：

```text
block in  all
block out all
```

这现在可以简化为：

```text
block
```

当未指定方向时，PF 将假定规则适用于双向移动的数据包。

同样，规则中可以省略 "`from any to any`" 和 "`all`" 子句：

```text
block in on rl0 all
pass  in quick log on rl0 proto tcp from any to any port 22
```

这可以简化为：

```text
block in on rl0
pass  in quick log on rl0 proto tcp to port 22
```

第一条规则阻止 `rl0` 上从任何地方到任何地方的所有传入数据包，而第二条规则通过 `rl0` 上到端口 22 的传入 TCP 流量。

<h3 id="return"><code>Return</code> 简化</h3>

用于阻止数据包并回复 TCP RST 或 ICMP Unreachable 响应的规则集可能如下所示：

```text
block in all
block return-rst  in  proto tcp all
block return-icmp in  proto udp all
block out all
block return-rst  out proto tcp all
block return-icmp out proto udp all
```

这可以简化为：

```text
block return
```

当 PF 看到 `return` 关键字时，它足够聪明，可以根据被阻止数据包的协议发送适当的响应，或者根本不发送响应。

<h3 id="order">关键字顺序</h3>

在大多数情况下，指定关键字的顺序是灵活的。例如，写成这样的规则：

```text
pass in log quick on rl0 proto tcp to port 22 flags S/SA queue ssh label ssh
```

也可以写成：

```text
pass in quick log on rl0 proto tcp to port 22 queue ssh label ssh flags S/SA
```

其他类似的变体也可以工作。
