## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 数据包标记（策略过滤）</span>

---

- <a href="#intro">简介</a>
- <a href="#assign">为数据包分配标记</a>
- <a href="#check">检查已应用的标记</a>
- <a href="#policy">策略过滤</a>
- <a href="#ethernet">标记以太网帧</a>

---

<h2 id="intro">简介</h2>

数据包标记是一种使用内部标识符标记数据包的方法，以后可以在过滤和转换规则标准中使用该标识符。通过标记，可以执行诸如在接口之间创建“信任”以及确定数据包是否已由转换规则处理等操作。也可以从基于规则的过滤转向基于策略的过滤。

<h2 id="assign">为数据包分配标记</h2>

要向数据包添加标记，请使用 `tag` 关键字：

```text
pass in on $int_if all tag INTERNAL_NET
```

标记 `INTERNAL_NET` 将添加到任何匹配上述规则的数据包。

也可以使用<a href="./02_macros.md#macros">宏</a>分配标记。例如：

```text
name = "INTERNAL_NET"
pass in on $int_if all tag $name
```

还有一组预定义的宏也可以使用。

- `$if` - 接口
- `$srcaddr` - 源 IP 地址
- `$dstaddr` - 目标 IP 地址
- `$srcport` - 源端口规范
- `$dstport` - 目标端口规范
- `$proto` - 协议
- `$nr` - 规则编号

这些宏在规则集加载时扩展，而不是在运行时扩展。

标记遵循以下规则：

- 标记是“粘性”的。一旦匹配规则将标记应用于数据包，它就不会被删除。但是，可以用不同的标记替换它。
- 由于标记的“粘性”，即使最后匹配的规则没有使用 `tag` 关键字，数据包也可以有标记。
- 一个数据包一次最多只能分配一个标记。
- 标记是 *内部* 标识符。标记不会通过网络发送。
- 标记名称最长可达 63 个字符。

以以下规则集为例。

```text
pass in on $int_if tag INT_NET
pass in quick on $int_if proto tcp to port 80 tag INT_NET_HTTP
pass in quick on $int_if from 192.168.1.5
```

- 在 `$int_if` 上进入的数据包将被规则 #1 分配标记 `INT_NET`。
- 在 `$int_if` 上进入并前往端口 80 的 TCP 数据包将首先由规则 #1 分配标记 `INT_NET`。然后该标记将被规则 #2 替换为 `INT_NET_HTTP` 标记。
- 从 192.168.1.5 在 `$int_if` 上进入的数据包将以两种方式之一进行标记。如果数据包前往 TCP 端口 80，它将匹配规则 #2 并被标记为 `INT_NET_HTTP`。否则，数据包将匹配规则 #3，但将被标记为 `INT_NET`。因为数据包匹配规则 #1，所以应用了 `INT_NET` 标记，并且除非后续匹配规则指定了标记，否则不会将其删除（这就是标记的“粘性”）。

<h2 id="check">检查已应用的标记</h2>

要检查以前应用的标记，请使用 `tagged` 关键字：

```text
pass out on egress tagged INT_NET
```

外部接口上的传出数据包必须带有 `INT_NET` 标记才能匹配上述规则。也可以使用 `!` 运算符进行反向匹配：

```text
pass out on egress ! tagged WIFI_NET
```

<h2 id="policy">策略过滤</h2>

策略过滤采用不同的方法来编写过滤规则集。定义了一个策略，该策略设置了通过哪些类型的流量以及阻止哪些类型的流量的规则。然后根据源/目标 IP 地址/端口、协议等传统标准将数据包分类到策略中。例如，检查以下防火墙策略：

- 允许从内部 LAN 到互联网的流量 (LAN_INET)，并且必须进行转换 (LAN_INET_NAT)。
- 允许从内部 LAN 到 DMZ 的流量 (LAN_DMZ)。
- 允许从互联网到 DMZ 中服务器的流量 (INET_DMZ)。
- 允许从互联网重定向到 <a href="https://man.openbsd.org/spamd">spamd(8)</a> 的流量 (SPAMD)。
- 所有其他流量都被阻止。

请注意策略如何涵盖 *所有* 将通过防火墙的流量。括号中的项目表示将用于该策略项目的标记。

现在需要编写规则将数据包分类到策略中。

```text
block all
pass out on egress inet tag LAN_INET_NAT tagged LAN_INET nat-to ($ext_if)
pass in  on $int_if from $int_net tag LAN_INET
pass in  on $int_if from $int_net to $dmz_net tag LAN_DMZ
pass in  on egress proto tcp to $www_server port 80 tag INET_DMZ
pass in  on egress proto tcp from <spamd> to port smtp tag SPAMD rdr-to 127.0.0.1 port 8025
```

现在设置定义策略的规则。

```text
pass in  quick on egress  tagged SPAMD
pass out quick on egress  tagged LAN_INET_NAT
pass out quick on $dmz_if tagged LAN_DMZ
pass out quick on $dmz_if tagged INET_DMZ
```

既然整个规则集已设置好，更改就是修改分类规则的问题。例如，如果将 POP3/SMTP 服务器添加到 DMZ，则需要为 POP3 和 SMTP 流量添加分类规则，如下所示：

```text
mail_server = "192.168.0.10"
[...]
pass in on egress proto tcp to $mail_server port { smtp, pop3 } tag INET_DMZ
```

电子邮件流量现在将作为 INET_DMZ 策略条目的一部分通过。

完整的规则集：

```text
int_if      = "dc0"
dmz_if      = "dc1"
int_net     = "10.0.0.0/24"
dmz_net     = "192.168.0.0/24"
www_server  = "192.168.0.5"
mail_server = "192.168.0.10"

table <spamd> persist file "/etc/spammers"
# 分类 -- 根据定义的防火墙策略对数据包进行分类。
block all
pass out on egress inet tag LAN_INET_NAT tagged LAN_INET nat-to (egress)
pass in on $int_if from $int_net tag LAN_INET
pass in on $int_if from $int_net to $dmz_net tag LAN_DMZ
pass in on egress proto tcp to $www_server port 80 tag INET_DMZ
pass in on egress proto tcp from <spamd> to port smtp tag SPAMD rdr-to 127.0.0.1 port 8025

# 策略执行 -- 根据定义的防火墙策略通过/阻止。
pass in  quick on egress  tagged SPAMD
pass out quick on egress  tagged LAN_INET_NAT
pass out quick on $dmz_if tagged LAN_DMZ
pass out quick on $dmz_if tagged INET_DMZ
```

<h2 id="ethernet">标记以太网帧</h2>

如果执行标记/过滤的机器也充当 <a href="https://man.openbsd.org/bridge.4">bridge(4)</a>，则可以在以太网级别执行标记。通过创建使用 `tag` 关键字的网桥过滤规则，可以使 PF 基于源或目标 MAC 地址进行过滤。网桥规则是使用 <a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a> 命令创建的：

```shell
# ifconfig bridge0 rule pass in on fxp0 src 0:de:ad:be:ef:0 tag USER1
```

然后在 `pf.conf` 中：

```text
pass in on fxp0 tagged USER1
```
