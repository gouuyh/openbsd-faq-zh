## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 防火墙冗余 (CARP 和 pfsync)</span>

---

- <a href="#carpintro">CARP 简介</a>
- <a href="#carpop">CARP 操作</a>
- <a href="#carpconfig">配置 CARP</a>
- <a href="#carpex">CARP 示例</a>
- <a href="#pfsyncintro">pfsync 简介</a>
- <a href="#pfsyncop">pfsync 操作</a>
- <a href="#pfsyncconfig">配置 pfsync</a>
- <a href="#pfsyncex">pfsync 示例</a>
- <a href="#failover">结合 CARP 和 pfsync 实现故障转移和冗余</a>
- <a href="#ops">操作问题</a>
    - <a href="#bootconfig">在引导期间配置 CARP 和 pfsync</a>
    - <a href="#forcefail">强制主节点故障转移</a>
    - <a href="#RulesetTips">规则集提示</a>
- <a href="#ref">其他参考资料</a>

---

<h2 id="carpintro">CARP 简介</h2>

CARP 是通用地址冗余协议 (Common Address Redundancy Protocol)。其主要目的是允许同一网段上的多台主机共享一个 IP 地址。CARP 是<a href="https://www.ietf.org/rfc/rfc3768.txt">虚拟路由器冗余协议</a> (VRRP) 和<a href="https://www.ietf.org/rfc/rfc2281.txt">热备份路由器协议</a> (HSRP) 的一种安全、免费的替代方案。

CARP 的工作原理是允许同一网段上的一组主机共享一个 IP 地址。这组主机被称为“冗余组”。冗余组被分配一个在组成员之间共享的 IP 地址。在组内，一台主机被指定为“主 (master)”，其余主机被指定为“备份 (backups)”。主主机是当前“持有”共享 IP 的主机；它响应针对该地址的任何流量或 ARP 请求。每台主机可以同时属于多个冗余组。

CARP 的一个常见用途是创建一组冗余防火墙。分配给冗余组的虚拟 IP 在客户端机器上配置为默认网关。如果主防火墙发生故障或离线，IP 将移动到其中一个备份防火墙，服务将不受影响地继续。

CARP 支持 IPv4 和 IPv6。

<h2 id="carpop">CARP 操作</h2>

组中的主主机定期向本地网络发送通告，以便备份主机知道它仍然活着。如果备份主机在设定的时间内没有听到来自主主机的通告，它们中的一个将接管主主机的职责（具有最低配置 `advbase` 和 `advskew` 值的备份主机）。

同一网段上可能存在多个 CARP 组。CARP 通告包含虚拟主机 ID，允许组成员识别通告属于哪个冗余组。

为了防止网段上的恶意用户欺骗 CARP 通告，每个组都可以配置密码。发送到该组的每个 CARP 数据包都受到 SHA1 HMAC 的保护。

由于 CARP 是其自己的协议，因此在过滤规则集中应该有一个显式的通过规则：

```text
pass out on $carp_dev proto carp
```

`$carp_dev` 应该是 CARP 通信所经过的物理接口。

<h2 id="carpconfig">配置 CARP</h2>

每个冗余组都由一个 <a href="https://man.openbsd.org/carp">carp(4)</a> 虚拟网络接口表示。因此，CARP 是使用 <a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a> 配置的。

```shell
# ifconfig carpN create
# ifconfig carpN vhid vhid [pass password] [carpdev carpdev] \
   [advbase advbase] [advskew advskew] [state state] [group|-group group] \
   ipaddress netmask mask
```

- `carpN`
  <br>carp(4) 虚拟接口的名称，其中 *N* 是表示接口编号的整数（例如 carp10）。

- `vhid`
  <br>虚拟主机 ID。这是一个唯一的数字，用于向组中的其他节点标识冗余组，并区分同一网络上的组。可接受的值为 1 到 255。这必须在组的所有成员上相同。

- `password`
  <br>与此冗余组中的其他启用 CARP 的主机通信时使用的身份验证密码。这必须在组的所有成员上相同。

- `carpdev`
  <br>此可选参数指定属于此冗余组的物理网络接口。默认情况下，CARP 将尝试通过查找与 carp(4) 接口的 *ipaddress* 和 *mask* 组合在同一子网中的物理接口来确定使用哪个接口。

- `advbase`
  <br>此可选参数指定通告我们是冗余组成员的频率（以秒为单位）。默认值为 1 秒。可接受的值为 1 到 255。

- `advskew`
  <br>此可选参数指定在发送 CARP 通告时对 `advbase` 进行多少倾斜。通过操作 `advskew`，可以选择主 CARP 主机。数字越大，主机在选择主节点时就越 *不* 被优先考虑。默认值为 0。可接受的值为 0 到 254。

- `state`
  <br>强制 carp(4) 接口进入特定状态。有效状态为 `init`、`backup` 和 `master`。

- `group, -group`
  <br>将 carp(4) 接口添加或移除到某个接口组。默认情况下，所有 carp 接口都添加到 `carp` 组。每个组都有一个 `carpdemote` 计数器，影响属于该组的所有 carp 接口。如果一个物理启用 CARP 的接口出现故障，CARP 将使该 carp(4) 接口所属的接口组的降级计数器增加 1，实际上导致所有组成员一起进行故障转移。

- `ipaddress`
  <br>这是分配给冗余组的共享 IP 地址。此地址不必与物理接口上的 IP 地址（如果存在）在同一子网中。但是，此地址需要在组中的所有主机上相同。

- `mask`
  <br>共享 IP 的子网掩码。

此外，CARP 行为可以通过 <a href="https://man.openbsd.org/sysctl">sysctl(8)</a> 控制。

- `net.inet.carp.allow`
  <br>是否接受传入的 CARP 数据包。默认为 1 (是)。

- `net.inet.carp.preempt`
  <br>允许冗余组内具有更好 `advbase` 和 `advskew` 的主机抢占主节点。`net.inet.carp.preempt` 默认为 0 (禁用)。

- `net.inet.carp.log`
  <br>记录状态更改、坏数据包和其他错误。此值可以在 0 到 7 之间，对应于 syslog(3) 优先级。默认为 2 (仅状态更改)。

<h2 id="carpex">CARP 示例</h2>

这是一个 CARP 配置示例：

```shell
# sysctl net.inet.carp.allow=1
# echo 'net.inet.carp.allow=1' >> /etc/sysctl.conf
# ifconfig carp1 create
# ifconfig carp1 vhid 1 pass mekmitasdigoat carpdev em0 advskew 100 10.0.0.1 netmask 255.255.255.0
```

这设置了以下内容：

- 启用 CARP 数据包的接收（这是默认设置）
- 创建一个 carp(4) 接口 `carp1`
- 为虚拟主机 #1 配置 `carp1`，启用密码，将 `em0` 设置为属于该组的接口，并由于 `advskew` 为 `100` 而使此主机成为备份（假设主节点设置的 `advskew` 小于 100）

分配给该组的共享 IP 是 10.0.0.1/255.255.255.0。

在 `carp1` 上运行 `ifconfig` 显示接口的状态。

```shell
# ifconfig carp1
carp1: flags=8802<UP,BROADCAST,SIMPLEX,MULTICAST> mtu 1500
     carp: BACKUP carpdev em0 vhid 1 advbase 1 advskew 100
     groups: carp
     inet 10.0.0.1 netmask 0xffffff00 broadcast 10.0.0.255
```

<h2 id="pfsyncintro">pfsync 简介</h2>

<a href="https://man.openbsd.org/pfsync">pfsync(4)</a> 网络接口暴露了对 <a href="https://man.openbsd.org/pf">pf(4)</a> 状态表所做的某些更改。通过使用 <a href="https://man.openbsd.org/tcpdump">tcpdump(8)</a> 监视此设备，可以实时观察状态表更改。此外，pfsync 接口可以在网络上发送这些状态更改消息，以便运行 PF 的其他节点可以将更改合并到它们自己的状态表中。同样，pfsync 也可以在网络上监听传入的消息。

<h2 id="pfsyncop">pfsync 操作</h2>

默认情况下，pfsync 不会在网络上发送或接收状态表更新；但是，仍然可以在本地机器上使用 tcpdump 或类似工具监视更新。

当 pfsync 设置为在网络上发送和接收更新时，默认行为是在本地网络上多播更新。所有更新均不经身份验证发送。最佳通用做法是：

1.  使用交叉电缆背对背连接将交换更新的两个节点，并使用该接口作为 `syncdev`（见<a href="#pfsyncconfig">下文</a>）。
2.  使用 ifconfig `syncpeer` 选项（见<a href="#pfsyncconfig">下文</a>），以便更新直接单播到对等方，然后在主机之间配置 <a href="https://man.openbsd.org/ipsec">ipsec(4)</a> 以保护 pfsync 流量。

当在网络上发送和接收更新时，应在过滤规则集中通过 pfsync 数据包：

```text
pass on $sync_if proto pfsync
```

`$sync_if` 应该是 pfsync 通信所经过的物理接口。

<h2 id="pfsyncconfig">配置 pfsync</h2>

由于 pfsync 是一个虚拟网络接口，因此使用 <a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a> 进行配置。

```shell
# ifconfig pfsyncN syncdev syncdev [syncpeer syncpeer] [defer|-defer]
```

- `pfsyncN`
  <br>pfsync 接口的名称。

- `syncdev`
  <br>用于发送 pfsync 更新的物理接口的名称。

- `syncpeer`
  <br>此可选参数指定与之交换 pfsync 更新的主机的 IP 地址。默认情况下，pfsync 更新在本地网络上多播。此选项覆盖该行为，而是将更新单播到指定的 `syncpeer`。

- `defer`
  <br>如果使用 `defer` 标志，通过防火墙的新连接的初始数据包将不会传输，直到另一个 pfsync 系统确认状态表添加，或者超时过期。这增加了一些小的延迟，但允许多个防火墙可能主动处理数据包（“双活/active-active”）时的流量流动，例如使用某些 <a href="https://man.openbsd.org/ospfd">ospfd(8)</a>、<a href="https://man.openbsd.org/bgpd">bgpd(8)</a> 或 <a href="https://man.openbsd.org/carp">carp(4)</a> 配置。

<h2 id="pfsyncex">pfsync 示例</h2>

这是一个 pfsync 配置示例：

```shell
# ifconfig pfsync0 syncdev em1 up
```

这在 `em1` 接口上启用了 pfsync。传出更新将在网络上多播，允许任何其他运行 pfsync 的主机接收它们。

<h2 id="failover">结合 CARP 和 pfsync 实现故障转移</h2>

通过结合 CARP 和 pfsync 的功能，可以使用一组两个或多个防火墙来创建高可用性、完全冗余的防火墙集群。CARP 处理从一个防火墙到另一个防火墙的自动故障转移，而 pfsync 同步所有防火墙之间的状态表。在发生故障转移时，流量可以不间断地流经新的主防火墙。

示例场景：两个防火墙，`fw1` 和 `fw2`。

```text
         +----| WAN/internet |----+
         |                        |
      em2|                        |em2
      +-----+                  +-----+
      | fw1 |-em1----------em1-| fw2 |
      +-----+                  +-----+
      em0|                        |em0
         |                        |
      ---+-------Shared LAN-------+---
```

防火墙使用交叉电缆在 `em1` 上背对背连接。两者都在 `em0` 上连接到 LAN，在 `em2` 上连接到 WAN/互联网。IP 地址如下：

- fw1 em0: 172.16.0.1
- fw1 em1: 10.10.10.1
- fw1 em2: 192.0.2.1
- fw2 em0: 172.16.0.2
- fw2 em1: 10.10.10.2
- fw2 em2: 192.0.2.2
- LAN 共享 IP: 172.16.0.100
- WAN/互联网共享 IP: 192.0.2.100

网络策略是 `fw1` 将作为首选主节点。

要配置 fw1，首先启用抢占和组接口故障转移：

```shell
# sysctl net.inet.carp.preempt=1
# echo 'net.inet.carp.preempt=1' >> /etc/sysctl.conf
```

配置 pfsync：

```shell
# ifconfig em1 10.10.10.1 netmask 255.255.255.0
# ifconfig pfsync0 syncdev em1
# ifconfig pfsync0 up
```

在 LAN 侧配置 CARP：

```shell
# ifconfig carp1 create
# ifconfig carp1 vhid 1 carpdev em0 pass lanpasswd \
     172.16.0.100 netmask 255.255.255.0
```

在 WAN/互联网侧配置 CARP：

```shell
# ifconfig carp2 create
# ifconfig carp2 vhid 2 carpdev em2 pass netpasswd \
     192.0.2.100 netmask 255.255.255.0
```

然后相应地配置 fw2：

```shell
# sysctl net.inet.carp.preempt=1
# echo 'net.inet.carp.preempt=1' >> /etc/sysctl.conf
# ifconfig em1 10.10.10.2 netmask 255.255.255.0
# ifconfig pfsync0 syncdev em1
# ifconfig pfsync0 up
# ifconfig carp1 create
# ifconfig carp1 vhid 1 carpdev em0 pass lanpasswd \
     advskew 128 172.16.0.100 netmask 255.255.255.0
# ifconfig carp2 create
# ifconfig carp2 vhid 2 carpdev em2 pass netpasswd \
     advskew 128 192.0.2.100 netmask 255.255.255.0
```

<h2 id="ops">操作问题</h2>

<h3 id="bootconfig">在引导期间配置 CARP 和 pfsync</h3>

由于 carp 和 pfsync 都是网络接口类型，因此可以通过创建 <a href="https://man.openbsd.org/hostname.if">hostname.if(5)</a> 文件在引导时进行配置。<a href="https://man.openbsd.org/netstart">netstart</a> 启动脚本将负责创建接口并对其进行配置。

示例：

`/etc/hostname.carp1`
```text
inet 172.16.0.100 255.255.255.0 172.16.0.255 vhid 1 carpdev em0 pass lanpasswd
```

`/etc/hostname.pfsync0`
```text
syncdev em1
up
```

<h3 id="forcefail">强制主节点故障转移</h3>

有时可能有必要故意对主节点进行故障转移或降级。示例可能包括将主节点关闭以进行维护或在排除问题时。这里的目标是将流量优雅地故障转移到其中一个备份主机，以便用户不会注意到任何影响。

要对特定 CARP 组进行故障转移，请关闭主节点上的 carp 接口。这将导致主节点以“无限”的 `advbase` 和 `advskew` 通告自己。备份主机将看到这一点并立即接管主节点的角色。

```shell
# ifconfig carp1 down
```

另一种方法是将 `advskew` 增加到比备份主机上的 `advskew` 更高的值。这将导致故障转移，但仍允许主节点参与 CARP 组。

另一种故障转移方法是调整 CARP 降级计数器。降级计数器是衡量主机成为 CARP 组主节点的“准备程度”的指标。例如，当主机正在启动时，在其所有接口配置完毕、所有网络守护进程启动之前，让它成为 CARP 主节点是一个坏主意。通告高降级值的主机将不太可能被选为主节点。

降级计数器存储在 CARP 接口所属的每个接口组中。默认情况下，所有 CARP 接口都是 "carp" 接口组的成员。可以使用 `ifconfig` 查看降级计数器的当前值：

```shell
# ifconfig -g carp
carp: carp demote count 0
```

在此示例中，显示了与 "carp" 接口组关联的计数器。当 CARP 主机在网络上通告自己时，它会获取 carp 接口所属的每个接口组的降级计数器之和，并将该值作为其降级值进行通告。

现在假设以下示例。两个运行 CARP 的防火墙具有以下 CARP 接口：

- carp1 -- 会计部门
- carp2 -- 普通员工
- carp3 -- 互联网
- carp4 -- DMZ

目标是仅将 carp1 和 carp2 组故障转移到辅助防火墙。

首先，将每个分配给一个新的接口组，在本例中名为 "internal"：

```shell
# ifconfig carp1 group internal
# ifconfig carp2 group internal
# ifconfig internal
carp1: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
     carp: MASTER carpdev em0 vhid 1 advbase 1 advskew 100
     groups: carp internal
     inet 10.0.0.1 netmask 0xffffff00 broadcast 10.0.0.255
carp2: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
     carp: MASTER carpdev em1 vhid 2 advbase 1 advskew 100
     groups: carp internal
     inet 10.0.1.1 netmask 0xffffff00 broadcast 10.0.1.255
```

现在增加 "internal" 组的降级计数器：

```shell
# ifconfig -g internal
internal: carp demote count 0
# ifconfig -g internal carpdemote 50
# ifconfig -g internal
internal: carp demote count 50
```

防火墙现在将优雅地将 carp1 和 carp2 组故障转移到集群中的另一个防火墙，同时仍保持 carp3 和 carp4 的主节点身份。如果另一个防火墙开始以高于 50 的降级值通告自己，或者如果另一个防火墙完全停止通告，则此防火墙将再次接管 carp1 和 carp2 的主节点身份。

要回切到主防火墙，请反转更改：

```shell
# ifconfig -g internal -carpdemote 50
# ifconfig -g internal
internal: carp demote count 0
```

网络守护进程如 <a href="https://man.openbsd.org/bgpd">bgpd(8)</a> 和 <a href="https://man.openbsd.org/sasyncd">sasyncd(8)</a> 利用降级计数器来确保防火墙在 BGP 会话建立和 IPsec SA 同步之前不会成为主节点。

<h3 id="RulesetTips">规则集提示</h3>

**过滤物理接口。**
就 PF 而言，网络流量来自物理接口，而不是 CARP 虚拟接口（即 `carp0`）。因此，请相应地编写规则集。不要忘记 PF 规则中的接口名称可以是物理接口的名称或与该接口关联的地址。例如，此规则可能是正确的：

```text
pass in on fxp0 inet proto tcp from any to carp0 port 22
```

但是将 `fxp0` 替换为 `carp0` 将无法按预期工作。

**不要忘记** 通过 `proto carp` 和 `proto pfsync`！

<h2 id="ref">其他参考资料</h2>

请参阅这些其他来源以获取更多信息：

- <a href="https://man.openbsd.org/carp">carp(4)</a>
- <a href="https://man.openbsd.org/pfsync">pfsync(4)</a>
- <a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a>
- <a href="https://man.openbsd.org/hostname.if">hostname.if(5)</a>
- <a href="https://man.openbsd.org/pf.conf">pf.conf(5)</a>
- <a href="https://man.openbsd.org/ifstated">ifstated(8)</a>
- <a href="https://man.openbsd.org/ifstated.conf">ifstated.conf(5)</a>
