## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 日志</span>

---

- <a href="#intro">简介</a>
- <a href="#log">记录数据包</a>
- <a href="#logfile">读取日志文件</a>
- <a href="#filter">过滤日志输出</a>

---

<h2 id="intro">简介</h2>

当 PF 记录数据包时，数据包报头的副本会连同一些附加数据（如数据包经过的接口、PF 采取的操作（通过或阻止）等）一起发送到 <a href="https://man.openbsd.org/pflog">pflog(4)</a> 接口。`pflog` 接口允许用户空间应用程序从内核接收 PF 的日志数据。如果在系统启动时启用了 PF，则会启动 <a href="https://man.openbsd.org/pflogd">pflogd(8)</a> 守护进程。默认情况下，pflogd 监听 `pflog0` 接口并将所有日志数据写入 `/var/log/pflog` 文件。

<h2 id="log">记录数据包</h2>

为了记录通过 PF 的数据包，必须使用 `log` 关键字。`log` 关键字会导致所有匹配规则的数据包被记录。在规则<a href="./04_filter.md#state">创建状态</a>的情况下，只有看到的第一个数据包（导致状态创建的那个）会被记录。

可以提供给 `log` 关键字的选项有：

- `all`
  <br>导致所有匹配的数据包被记录，而不只是初始数据包。对创建状态的规则很有用。

- `to pflogN`
  <br>导致所有匹配的数据包被记录到指定的 pflog(4) 接口。例如，当使用 <a href="https://man.openbsd.org/spamlogd">spamlogd(8)</a> 时，PF 可以将所有 SMTP 流量记录到专用的 pflog 接口。然后可以告诉 spamlogd 守护进程在该接口上监听。这可以保持主 PF 日志文件不包含除此之外不需要记录的 SMTP 流量。使用 <a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a> 创建 pflog 接口。默认日志接口 `pflog0` 是自动创建的。

- `user`
  <br>导致拥有数据包源/目的 socket（以本地 socket 为准）的用户 id 和组 id 与标准日志信息一起被记录。

选项在 `log` 关键字后的括号中给出；多个选项可以用逗号或空格分隔。

```text
pass in log (all, to pflog1) on egress inet proto tcp to egress port 22
```

<h2 id="logfile">读取日志文件</h2>

pflogd 写入的日志文件是二进制格式，不能使用文本编辑器读取。必须使用 <a href="https://man.openbsd.org/tcpdump">tcpdump(8)</a>。

要查看日志文件：

```shell
# tcpdump -n -e -ttt -r /var/log/pflog
```

请注意，使用 tcpdump 观察 pflog 文件 *不会* 提供实时显示。要实时显示记录的数据包，需使用 `pflog0` 接口：

```shell
# tcpdump -n -e -ttt -i pflog0
```

在检查日志时，应特别注意 tcpdump 的详细协议解码（通过 `-v` 命令行选项激活）。tcpdump 的协议解码器并没有完美的安全记录。至少在理论上，通过日志设备记录的部分数据包有效载荷可能会引发延迟攻击。建议的做法是在以这种方式检查日志文件之前，将它们移出防火墙机器。

还应采取额外的谨慎措施来保护对日志的访问。默认情况下，pflogd 将在日志文件中记录数据包的 160 个字节。访问日志可能会提供对敏感数据包有效载荷的部分访问权限。

<h2 id="filter">过滤日志输出</h2>

因为 pflogd 以 tcpdump 二进制格式记录日志，所以在审查日志时可以使用 tcpdump 的全部功能。例如，只查看匹配特定端口的数据包：

```shell
# tcpdump -n -e -ttt -r /var/log/pflog port 80
```

这可以通过将数据包的显示限制为特定的主机和端口组合来进一步细化：

```shell
# tcpdump -n -e -ttt -r /var/log/pflog port 80 and host 192.168.1.3
```

同样的思路也可以应用于从 `pflog0` 接口读取时：

```shell
# tcpdump -n -e -ttt -i pflog0 host 192.168.4.2
```

请注意，这对哪些数据包被记录到 pflogd 日志文件没有影响；上述命令仅显示正在记录的数据包。

除了使用标准的 <a href="https://man.openbsd.org/tcpdump.8">tcpdump(8)</a> 过滤规则外，tcpdump 过滤语言已针对读取 pflogd 输出进行了扩展：

- `ip` - 地址族为 IPv4。
- `ip6` - 地址族为 IPv6。
- `on int` - 数据包通过接口 *int*。
- `ifname int` - 与 `on int` 相同。
- `ruleset name` - 数据包匹配的<a href="./09_anchors.md">规则集/锚</a>。
- `rulenum num` - 数据包匹配的过滤规则是规则编号 *num*。
- `action act` - 对数据包采取的操作。可能的操作是 `pass`（通过）和 `block`（阻止）。
- `reason res` - 采取 `action` 的原因。可能的原因包括 `match`, `bad-offset`, `fragment`, `short`, `normalize`, `memory`, `bad-timestamp`, `congestion`, `ip-option`, `proto-cksum`, `state-mismatch`, `state-insert`, `state-limit`, `src-limit` 和 `synproxy`。
- `inbound` - 数据包是入站的。
- `outbound` - 数据包是出站的。

示例：

```shell
# tcpdump -n -e -ttt -i pflog0 inbound and action block and on wi0
```

这将实时显示在 wi0 接口上被阻止的入站数据包的日志。
