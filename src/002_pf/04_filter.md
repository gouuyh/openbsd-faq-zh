## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 包过滤</span>

---

- <a href="#intro">简介</a>
- <a href="#syntax">规则语法</a>
- <a href="#defdeny">默认拒绝</a>
- <a href="#pass">通过流量</a>
- <a href="#quick"><code>quick</code> 关键字</a>
- <a href="#state">保持状态</a>
- <a href="#udpstate">保持 UDP 状态</a>
- <a href="#stateopts">状态跟踪选项</a>
- <a href="#tcpflags">TCP 标志</a>
- <a href="#synproxy">TCP SYN 代理</a>
- <a href="#antispoof">阻止欺骗数据包</a>
- <a href="#urpf">单播反向路径转发 (uRPF)</a>
- <a href="#osfp">被动操作系统指纹识别</a>
- <a href="#ipopts">IP 选项</a>
- <a href="#example">过滤规则集示例</a>

---

<h2 id="intro">简介</h2>

包过滤是指在数据包通过网络接口时对其进行选择性的通过或阻止。<a href="https://man.openbsd.org/pf">pf(4)</a> 在检查数据包时使用的标准基于第 3 层（<a href="https://man.openbsd.org/ip">IPv4</a> 和 <a href="https://man.openbsd.org/ip6">IPv6</a>）和第 4 层（<a href="https://man.openbsd.org/tcp">TCP</a>、<a href="https://man.openbsd.org/udp">UDP</a>、<a href="https://man.openbsd.org/icmp">ICMP</a> 和 <a href="https://man.openbsd.org/icmp6">ICMPv6</a>）报头。最常用的标准是源地址和目的地址、源端口和目的端口以及协议。

过滤规则指定数据包必须匹配的标准，以及在找到匹配项时采取的操作（阻止或通过）。过滤规则按顺序从头到尾进行评估。除非数据包匹配包含 `quick` 关键字的规则，否则在采取最终操作之前，将根据 *所有* 过滤规则评估数据包。最后匹配的规则是“赢家”，并将决定对数据包采取什么操作。在过滤规则集的开头有一个隐含的 `pass all`，这意味着如果数据包不匹配任何过滤规则，则结果操作将是 `pass`。

<h2 id="syntax">规则语法</h2>

过滤规则的一般、*高度简化* 的语法如下：

```text
action [direction] [log] [quick] [on interface] [af] [proto protocol]
       [from src_addr [port src_port]] [to dst_addr [port dst_port]]
       [flags tcp_flags] [state]
```

- `action`
  <br>对匹配数据包采取的操作，`pass` 或 `block`。`pass` 操作将数据包传回内核进行进一步处理，而 `block` 操作将根据 <a href="./08_options.md#block-policy">`block-policy`</a> 选项的设置进行反应。可以通过指定 `block drop` 或 `block return` 来覆盖默认反应。

- `direction`
  <br>数据包在接口上移动的方向，`in` 或 `out`。

- `log`
  <br>指定应通过 <a href="https://man.openbsd.org/pflogd">pflogd(8)</a> 记录数据包。如果规则创建状态，则仅记录建立状态的数据包。要记录所有数据包，请使用 `log (all)`。

- `quick`
  <br>如果数据包匹配指定 `quick` 的规则，则该规则被视为最后匹配的规则，并采取指定的 `action`。

- `interface`
  <br>数据包通过的网络接口的名称或组。可以使用 <a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a> 命令将接口添加到任意组。内核也会自动创建几个组：
    - `egress` 组，包含持有默认路由的接口。
    - 克隆接口的接口族组。例如：`ppp` 或 `carp`。
  <br>这将导致规则分别匹配遍历任何 `ppp` 或 `carp` 接口的任何数据包。

- `af`
  <br>数据包的地址族，IPv4 为 `inet`，IPv6 为 `inet6`。PF 通常能够根据源和/或目标地址确定此参数。

- `protocol`
  <br>数据包的第 4 层协议：
    - `tcp`
    - `udp`
    - `icmp`
    - `icmp6`
    - 来自 <a href="https://man.openbsd.org/protocols">`/etc/protocols`</a> 的有效协议名称
    - 0 到 255 之间的协议号
    - 使用<a href="./02_macros.md#lists">列表</a>的一组协议。

- `src_addr`, `dst_addr`
  <br>IP 报头中的源/目标地址。地址可以指定为：
    - 单个 IPv4 或 IPv6 地址。
    - <a href="https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html">CIDR</a> 网络块。
    - 完全限定域名 (FQDN)，将在加载规则集时通过 DNS 解析。所有生成的 IP 地址都将替换到规则中。
    - 网络接口或组的名称。分配给接口的任何 IP 地址都将替换到规则中。
    - 网络接口名称后跟 `/netmask`（即 `/24`）。接口上的每个 IP 地址都与子网掩码组合形成 CIDR 网络块，该网络块将替换到规则中。
    - 括号 `( )` 中的网络接口或组的名称。这告诉 PF 如果命名接口上的 IP 地址发生变化，则更新规则。这对于通过 DHCP 或拨号获取 IP 地址的接口非常有用，因为每次地址更改时都不必重新加载规则集。
    - 网络接口名称后跟以下任何修饰符：
        - `:network` - 替换 CIDR 网络块（例如，192.168.0.0/24）
        - `:broadcast` - 替换网络广播地址（例如，192.168.0.255）
        - `:peer` - 替换点对点链路上的对等方 IP 地址
        <br>此外，`:0` 修饰符可以附加到接口名称或上述任何修饰符，以指示 PF 不应在替换中包含别名 IP 地址。当接口包含在括号中时，也可以使用这些修饰符。示例：`fxp0:network:0`
    - <a href="./03_tables.md">表 (table)</a>。
    - 关键字 `urpf-failed` 可用于源地址，以指示应通过 <a href="#urpf">uRPF 检查</a>运行它。
    - 上述任何一项，但使用 `!`（“非”）修饰符进行否定。
    - 使用<a href="./02_macros.md#lists">列表</a>的一组地址。
    - 关键字 `any` 表示所有地址。
    - 关键字 `all` 是 `from any to any` 的简写。

- `src_port`, `dst_port`
  <br>第 4 层数据包报头中的源/目标端口。端口可以指定为：
    - 1 到 65535 之间的数字
    - 来自 <a href="https://man.openbsd.org/services">`/etc/services`</a> 的有效服务名称
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

- `tcp_flags`
  <br>指定使用 `proto tcp` 时必须在 TCP 报头中设置的标志。标志指定为 `flags check/mask`。例如：`flags S/SA` - 这指示 PF 仅查看 S 和 A（SYN 和 ACK）标志，并且仅当 SYN 标志为“开”时才匹配（默认情况下应用于所有 TCP 规则）。`flags any` 指示 PF 不检查标志。

- `state`
  <br>指定是否对匹配此规则的数据包保留状态信息。
    - `no state` - 适用于 TCP、UDP 和 ICMP。PF 不会对此连接进行状态跟踪。对于 TCP 连接，通常还需要 `flags any`。
    - `keep state` - 适用于 TCP、UDP 和 ICMP。此选项是所有过滤规则的默认选项。
    - `modulate state` - 仅适用于 TCP。PF 将为匹配此规则的数据包生成强初始序列号 (ISN)。
    - `synproxy state` - 代理传入的 TCP 连接，以帮助保护服务器免受欺骗性 TCP SYN 洪水攻击。此选项包括 `keep state` 和 `modulate state` 的功能。

<h2 id="defdeny">默认拒绝</h2>

设置防火墙时的推荐做法是采取“默认拒绝”的方法。即拒绝 *一切*，然后选择性地允许某些流量通过防火墙。推荐这种方法是因为它宁可错杀也不放过，而且使编写规则集更容易。

要创建默认拒绝过滤策略，第一条过滤规则应该是：

```text
block all
```

这将阻止所有接口上从任何地方到任何地方的双向流量。

<h2 id="pass">通过流量</h2>

现在必须明确允许流量通过防火墙，否则它将被默认拒绝策略丢弃。这就是源/目标端口、源/目标地址和协议等数据包标准发挥作用的地方。每当允许流量通过防火墙时，规则应尽可能严格。这是为了确保预期的流量，且仅有预期的流量被允许通过。

一些例子：

```text
# 允许流量从本地网络 192.168.0.0/24 通过 dc0 进入到 OpenBSD 机器的 IP 地址 192.168.0.1。
# 同时，允许返回流量通过 dc0 输出。
pass in  on dc0 from 192.168.0.0/24 to 192.168.0.1
pass out on dc0 from 192.168.0.1    to 192.168.0.0/24

# 允许 TCP 流量通过 egress 接口进入运行在 OpenBSD 机器上的 Web 服务器。
pass in on egress proto tcp from any to egress port www
```

<h2 id="quick"><code>quick</code> 关键字</h2>

如前所述，每个数据包都从上到下根据过滤规则集进行评估。默认情况下，数据包被标记为通过，这可以被任何规则更改，并且在过滤规则结束之前可能会来回更改多次。**最后匹配的规则获胜**，但有一个例外：过滤规则上的 `quick` 选项具有取消任何进一步规则处理的效果，并导致采取指定的操作。让我们看几个例子：

错误：

```text
block in on egress proto tcp to port ssh
pass  in all
```

在这种情况下，`block` 行可能会被评估，但永远不会有任何效果，因为它后面紧跟一行通过所有内容的规则。

更好：

```text
block in quick on egress proto tcp to port ssh
pass  in all
```

这些规则的评估方式略有不同。如果由于 `quick` 选项而匹配了 `block` 行，则数据包将被阻止，并且规则集的其余部分将被忽略。

<h2 id="state">保持状态</h2>

PF 的重要能力之一是“保持状态”或“状态检测”。状态检测是指 PF 跟踪网络连接状态或进度的能力。通过在状态表中存储有关每个连接的信息，PF 能够快速确定通过防火墙的数据包是否属于已建立的连接。如果是，它将通过防火墙而不经过规则集评估。

保持状态有许多优点，包括更简单的规则集和更好的包过滤性能。PF 能够将 *任一* 方向移动的数据包匹配到状态表条目，这意味着不需要编写通过返回流量的过滤规则。由于匹配状态连接的数据包不经过规则集评估，PF 处理这些数据包所花费的时间可以大大减少。

当规则创建状态时，匹配规则的第一个数据包会在发送方和接收方之间创建一个“状态”。现在，不仅从发送方到接收方的数据包匹配状态条目并绕过规则集评估，而且从接收方到发送方的回复数据包也是如此。

所有 *pass* 规则在数据包匹配规则时自动创建状态条目。可以使用 `no state` 选项显式禁用此功能。

```text
pass out on egress proto tcp from any to any
```

此规则允许 egress 接口上的任何出站 TCP 流量，并允许回复流量通过防火墙传回。保持状态显着提高了性能，因为状态查找比通过过滤规则运行数据包要快得多。

`modulate state` 选项的工作方式类似于 `keep state`，但它仅适用于 TCP 数据包。使用 `modulate state`，出站连接的初始序列号 (ISN) 是随机的。这对于保护由选择 ISN 较差的某些操作系统发起的连接非常有用。为了允许更简单的规则集，`modulate state` 选项可用于指定 TCP 以外协议的规则中。在这些情况下，它被视为 `keep state`。

保持传出 TCP、UDP 和 ICMP 数据包的状态并调制 TCP ISN：

```text
pass out on egress proto { tcp, udp, icmp } from any to any modulate state
```

保持状态的另一个优点是相应的 ICMP 流量将通过防火墙。例如，如果通过防火墙的 TCP 连接正在被状态跟踪，并且到达引用此 TCP 连接的 ICMP 源抑制消息，它将匹配到适当的状态条目并通过防火墙。

状态条目的范围由 <a href="./08_options.md#state-policy">state-policy</a> 运行时选项全局控制，并通过 `if-bound` 和 `floating` 状态选项关键字在每个规则的基础上进行控制。这些每个规则的关键字与用于 `state-policy` 选项时的含义相同。例如：

```text
pass out on egress proto { tcp, udp, icmp } from any to any modulate state (if-bound)
```

此规则将规定，为了使数据包匹配状态条目，它们必须通过 egress 接口。

<h2 id="udpstate">保持 UDP 状态</h2>

虽然 UDP 通信会话确实没有任何状态概念（显式的通信开始和停止），但这不会对 PF 为 UDP 会话创建状态的能力产生任何影响。对于没有“开始”和“结束”数据包的协议，PF 只是跟踪自匹配数据包通过以来已经过了多长时间。如果达到超时，则清除状态。超时值可以在 `pf.conf` 文件的 <a href="./08_options.md">options</a> 部分中设置。

<h2 id="stateopts">状态跟踪选项</h2>

创建状态条目的过滤规则可以指定各种选项来控制结果状态条目的行为。可以使用以下选项：

- `max number`
  <br>将规则可以创建的最大状态条目数限制为 *number*。如果达到最大值，通常会创建状态的数据包将无法匹配此规则，直到现有状态的数量降至限制以下。

- `no state`
  <br>防止规则自动创建状态条目。

- `source-track`
  <br>此选项启用跟踪每个源 IP 地址创建的状态数。此选项有两种格式：
    - `source-track rule` - 此规则创建的最大状态数受规则的 `max-src-nodes` 和 `max-src-states` 选项限制。只有由此特定规则创建的状态条目才计入规则的限制。
    - `source-track global` - 所有使用此选项的规则创建的状态数受到限制。每个规则可以指定不同的 `max-src-nodes` 和 `max-src-states` 选项，但是由任何参与规则创建的状态条目都计入每个单独规则的限制。
  <br>全局跟踪的源 IP 地址总数可以通过 <a href="./08_options.md#limit">`src-nodes` 运行时选项</a>控制。

- `max-src-nodes number`
  <br>当使用 `source-track` 选项时，`max-src-nodes` 将限制可以同时创建状态的源 IP 地址的数量。此选项只能与 `source-track rule` 一起使用。

- `max-src-states number`
  <br>当使用 `source-track` 选项时，`max-src-states` 将限制每个源 IP 地址可以创建的同时状态条目的数量。此限制的范围（即仅由此规则创建的状态或由所有使用 `source-track` 的规则创建的状态）取决于指定的 `source-track` 选项。

选项在括号内指定，并紧跟在状态关键字之一（`keep state`、`modulate state` 或 `synproxy state`）之后。多个选项用逗号分隔。`keep state` 选项是所有过滤规则的隐含默认值。尽管如此，在指定状态选项时，仍必须在选项前面使用其中一个状态关键字。

示例规则：

```text
pass in on egress proto tcp to $web_server port www keep state   \
                  (max 200, source-track rule, max-src-nodes 100, \
                   max-src-states 3)
```

上面的规则定义了以下行为：

- 将此规则可以创建的绝对最大状态数限制为 200
- 启用源跟踪：仅根据此规则创建的状态限制状态创建
- 将可以同时创建状态的最大节点数限制为 100
- 将每个源 IP 的最大同时状态数限制为 3

可以对已完成三次握手的有状态 TCP 连接施加一组单独的限制。

- `max-src-conn number`
  <br>限制单个主机可以发起的已完成三次握手的最大同时 TCP 连接数。

- `max-src-conn-rate number / interval`
  <br>将新连接的速率限制为每段时间间隔一定数量。

这两个选项都会自动调用 `source-track rule` 选项，并且与 `source-track global` 不兼容。

由于这些限制仅针对已完成三次握手的 TCP 连接，因此可以对违规 IP 地址采取更积极的措施。

- `overload <table>`
  <br>将违规主机的 IP 地址放入命名表中。

- `flush [global]`
  <br>终止由此源 IP 创建的与此规则匹配的任何其他状态。当指定 `global` 时，终止匹配此源 IP 的所有状态，无论哪个规则创建了该状态。

示例：

```text
table <abusive_hosts> persist
block in quick from <abusive_hosts>

pass in on egress proto tcp to $web_server port www flags S/SA keep state \
                                (max-src-conn 100, max-src-conn-rate 15/5, \
                                 overload <abusive_hosts> flush)
```

这执行以下操作：

- 将每个源的最大连接数限制为 100
- 将连接速率限制为 5 秒内 15 个
- 将任何违反这些限制的主机的 IP 地址放入 `<abusive_hosts>` 表中
- 对于任何违规 IP 地址，刷新由此规则创建的任何状态

<h2 id="tcpflags">TCP 标志</h2>

基于标志匹配 TCP 数据包最常用于过滤试图打开新连接的 TCP 数据包。TCP 标志及其含义如下：

- **F** : FIN - 结束；会话结束
- **S** : SYN - 同步；表示开始会话的请求
- **R** : RST - 重置；断开连接
- **P** : PUSH - 推送；数据包立即发送
- **A** : ACK - 确认
- **U** : URG - 紧急
- **E** : ECE - 显式拥塞通知回显
- **W** : CWR - 拥塞窗口减少

要让 PF 在评估规则期间检查 TCP 标志，使用 `flags` 关键字，语法如下：

```text
flags check/mask
flags any
```

`mask` 部分告诉 PF 仅检查指定的标志，`check` 部分指定报头中必须为“开”的标志才能发生匹配。使用 `any` 关键字允许在报头中设置任何标志组合。

```text
pass in on egress proto tcp from any to any port ssh flags S/SA
pass in on egress proto tcp from any to any port ssh
```

由于默认设置了 `flags S/SA`，上述规则是等效的。这些规则中的每一个都通过设置了 SYN 标志的 TCP 流量，同时仅查看 SYN 和 ACK 标志。带有 SYN 和 ECE 标志的数据包将匹配上述规则，而带有 SYN 和 ACK 或仅 ACK 的数据包则不会。

可以使用上述 `flags` 选项覆盖默认标志。

使用标志时要小心——了解正在做什么以及为什么。有些人建议“仅当设置了 SYN 标志且没有其他标志时”才创建状态。这样的规则将以以下内容结尾：

```text
[...] flags S/FSRPAUEW  bad idea!!
```

理论是在 TCP 会话开始时创建状态，并且会话应该以 SYN 标志开始，没有其他标志。问题是某些站点使用 ECN 标志，任何使用 ECN 尝试连接到客户端机器的站点都会被此类规则拒绝。更好的准则是根本不指定任何标志，让 PF 应用默认标志。如果确实需要指定标志，此组合应该是安全的：

```text
[...] flags S/SAFR
```

虽然这既实用又安全，但如果流量也被清理 (scrub)，则没有必要检查 FIN 和 RST 标志。清理过程将导致 PF 丢弃任何具有非法 TCP 标志组合（如 SYN 和 RST）的传入数据包，并规范化潜在的歧义组合（如 SYN 和 FIN）。

<h2 id="synproxy">TCP SYN 代理</h2>

通常，当客户端向服务器发起 TCP 连接时，PF 会在握手数据包到达时在两个端点之间传递它们。然而，PF 有能力 *代理* 握手。在代理握手的情况下，PF 本身将与客户端完成握手，与服务器发起握手，然后在两者之间传递数据包。在 TCP SYN 洪水攻击的情况下，攻击者永远不会完成三次握手，因此攻击者的数据包永远不会到达受保护的服务器，但合法客户端将完成握手并被通过。这最大限度地减少了欺骗性 TCP SYN 洪水对受保护服务的影响，而是在 PF 中处理它。但是，不建议常规使用此选项，因为当服务器无法处理请求以及涉及负载平衡器时，它会破坏预期的 TCP 协议行为。

TCP SYN 代理使用过滤规则中的 `synproxy state` 关键字启用。例如：

```text
pass in on egress proto tcp to $web_server port www synproxy state
```

在这里，到 Web 服务器的连接将由 PF 进行 TCP 代理。

由于 `synproxy state` 的工作方式，它还包括与 `keep state` 和 `modulate state` 相同的功能。

如果 PF 在 <a href="https://man.openbsd.org/bridge">bridge(4)</a> 上运行，SYN 代理将不起作用。

<h2 id="antispoof">阻止欺骗数据包</h2>

地址欺骗是指恶意用户在传输的数据包中伪造源 IP 地址，以隐藏真实地址或冒充网络上的另一个节点。一旦地址被欺骗，就可以在不泄露攻击真实来源的情况下发起网络攻击。攻击者还可以尝试获得对仅限于某些 IP 地址的网络服务的访问权限。

PF 通过 `antispoof` 关键字提供一些针对地址欺骗的保护：

```text
antispoof [log] [quick] for interface [af]
```

- `log`
  <br>指定应通过 <a href="https://man.openbsd.org/pflogd">pflogd(8)</a> 记录匹配的数据包。

- `quick`
  <br>如果数据包匹配此规则，则将其视为“获胜”规则，并且规则集评估将停止。

- `interface`
  <br>要激活欺骗保护的网络接口。这也可以是接口的<a href="./02_macros.md#lists">列表</a>。

- `af`
  <br>要激活欺骗保护的地址族，IPv4 为 `inet`，IPv6 为 `inet6`。

示例：

```text
antispoof for fxp0 inet
```

加载规则集时，任何出现的 `antispoof` 关键字都会扩展为两个过滤规则。假设 egress 接口的 IP 地址为 10.0.0.1，子网掩码为 255.255.255.0（即 /24），上述 `antispoof` 规则将扩展为：

```text
block in on ! fxp0 inet from 10.0.0.0/24 to any
block in inet from 10.0.0.1 to any
```

这些规则完成两件事：

- 阻止所有来自 10.0.0.0/24 网络但 *不* 通过 `fxp0` 接口进入的流量。由于 10.0.0.0/24 网络位于 `fxp0` 接口上，因此具有该网络块中源地址的数据包绝不应该出现在任何其他接口上。
- 阻止所有来自 10.0.0.1（`fxp0` 上的 IP 地址）的传入流量。主机绝不应该通过外部接口向自身发送数据包，因此任何具有属于机器的源地址的传入数据包都可以被视为恶意的。

**注意**：`antispoof` 规则扩展成的过滤规则也会阻止通过环回接口发送到本地地址的数据包。无论如何，最佳做法是跳过环回接口上的过滤，但这在使用 antispoof 规则时变得必要：

```text
set skip on lo0
antispoof for fxp0 inet
```

`antispoof` 的使用应仅限于已分配 IP 地址的接口。在没有 IP 地址的接口上使用 `antispoof` 将导致如下过滤规则：

```text
block drop in on ! fxp0 inet all
block drop in inet all
```

使用这些规则，存在阻止 *所有* 接口上 *所有* 入站流量的风险。

<h2 id="urpf">单播反向路径转发 (uRPF)</h2>

PF 提供单播反向路径转发 (uRPF) 功能。当通过 uRPF 检查运行数据包时，会在路由表中查找数据包的源 IP 地址。如果在路由表条目中找到的出站接口与数据包刚进入的接口相同，则 uRPF 检查通过。如果接口不匹配，则数据包的源地址可能已被欺骗。

可以通过在过滤规则中使用 `urpf-failed` 关键字对数据包执行 uRPF 检查：

```text
block in quick from urpf-failed label uRPF
```

请注意，uRPF 检查仅在路由对称的环境中有意义。

uRPF 提供与 <a href="#antispoof">antispoof</a> 规则相同的功能。

<h2 id="osfp">被动操作系统指纹识别</h2>

被动操作系统指纹识别 (OSFP) 是一种基于远程主机 TCP SYN 数据包中的某些特征被动检测远程主机操作系统的方法。此信息随后可用作过滤规则中的标准。

PF 通过将 TCP SYN 数据包的特征与<a href="./08_options.md#fingerprints">指纹文件</a>（默认为 <a href="https://man.openbsd.org/pf.os">pf.os(5)</a>）进行比较来确定远程操作系统。启用 PF 后，可以使用此命令查看当前指纹列表：

```shell
# pfctl -s osfp
```

在过滤规则中，可以通过 OS 类、版本或子类型/补丁级别指定指纹。这些项目中的每一个都列在上面显示的 `pfctl` 命令的输出中。要在过滤规则中指定指纹，使用 `os` 关键字：

```text
pass  in on egress proto tcp from any os OpenBSD
block in on egress proto tcp from any os "Windows 2000"
block in on egress proto tcp from any os "Linux 2.4 ts"
block in on egress proto tcp from any os unknown
```

特殊的操作系统类 `unknown` 允许在 OS 指纹未知时匹配数据包。

请注意以下几点：

- 由于欺骗和/或精心制作的数据包使其看起来像是源自特定操作系统，操作系统指纹偶尔会出错。
- 操作系统的某些修订版或补丁级别可能会更改堆栈的行为，并导致其要么不匹配指纹文件中的内容，要么完全匹配另一个条目。
- OSFP 仅适用于 TCP SYN 数据包；它不适用于其他协议或已建立的连接。

<h2 id="ipopts">IP 选项</h2>

PF 默认阻止设置了 IP 选项的数据包。这可能会使 nmap 等操作系统指纹识别实用程序的工作更加困难。如果应用程序需要传递这些数据包（例如多播或 IGMP），可以使用 `allow-opts` 指令：

```text
pass in quick on fxp0 all allow-opts
```

<h2 id="example">过滤规则集示例</h2>

下面是一个过滤规则集的示例。运行 PF 的机器充当小型内部网络和互联网之间的防火墙。只显示了过滤规则；`queueing`、<a href="./05_nat.md">`nat`</a>、<a href="./06_rdr.md">`rdr`</a> 等已在此示例中省略。

```text
int_if  = "dc0"
lan_net = "192.168.0.0/24"

# 包含分配给防火墙的所有 IP 地址的表
table <firewall> const { self }

# 不要在环回接口上过滤
set skip on lo0

# 清理传入的数据包
match in all scrub (no-df)

# 设置默认拒绝策略
block all

# 为所有接口激活欺骗保护
block in quick from urpf-failed

# 仅允许来自本地网络的 ssh 连接，如果它来自受信任的计算机 192.168.0.15。
# 使用 "block return" 以便发送 TCP RST 立即关闭被阻止的连接。
# 使用 "quick" 以便此规则不被下面的 "pass" 规则覆盖。
block return in quick on $int_if proto tcp from ! 192.168.0.15 to $int_if port ssh

# 允许所有进出本地网络的流量。
# 由于默认的 "keep state" 选项将自动应用，这些规则将创建状态条目。
pass in  on $int_if from $lan_net
pass out on $int_if to   $lan_net

# 允许 tcp、udp 和 icmp 在外部（互联网）接口上输出。
# tcp 连接将被调制，udp/icmp 将被状态跟踪。
pass out on egress proto { tcp udp icmp } all modulate state

# 允许外部接口上的 ssh 连接进入，只要它们不是发往防火墙的
# （即，它们是发往本地网络上的机器的）。
# 记录初始数据包，以便我们以后可以知道谁在尝试连接。
# 取消注释最后一部分以使用 tcp syn 代理来代理连接。
pass in log on egress proto tcp to ! <firewall> port ssh # synproxy state
```
