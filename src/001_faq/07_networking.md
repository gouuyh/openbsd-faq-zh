# [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">FAQ - 网络</span>

---

- <a href="#Setup">网络配置</a>
    - <a href="#Setup.if">识别和设置网络接口</a>
    - <a href="#Setup.gateway">默认主机名和网关</a>
    - <a href="#Setup.resolver">DNS 解析</a>
    - <a href="#Setup.activate">激活更改</a>
    - <a href="#Setup.chkroute">检查路由</a>
    - <a href="#Setup.aliases">在接口上设置别名</a>
- <a href="#DHCP">动态主机配置协议 (DHCP)</a>
- <a href="#Wireless">无线网络</a>
- <a href="#Bridge">设置网桥</a>
- <a href="#Multipath">等价多路径路由 (ECMP)</a>
- <a href="#NFS">使用 NFS</a>

---

<h2 id="Setup">网络配置</h2>

OpenBSD 中的网络配置是通过 `/etc` 中的文本文件完成的。通常，这些设置是在<a href="./02_installation.md">安装过程</a>中初始配置的。

<h3 id="Setup.if">识别和设置网络接口</h3>

接口是按网卡类型命名的，而不是按连接类型命名的。例如，这是一个 Intel 快速以太网卡的 <a href="https://man.openbsd.org/dmesg">dmesg(8)</a> 片段：

```text
fxp0 at pci0 dev 10 function 0 "Intel 82557" rev 0x0c: irq 5, address 00:02:b3:2b:10:f7
inphy0 at fxp0 phy 1: i82555 10/100 media interface, rev. 4
```

该设备使用 <a href="https://man.openbsd.org/fxp">fxp(4)</a> 驱动程序，并在此处分配了编号 0。

<a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a> 实用程序将显示系统上已识别的网络接口。

```shell
$ ifconfig
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> mtu 33200
        index 3 priority 0 llprio 3
        groups: lo
        inet 127.0.0.1 netmask 0xff000000
fxp0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
        lladdr 00:02:b3:2b:10:f7
        index 1 priority 0 llprio 3
        media: Ethernet autoselect (100baseTX full-duplex)
        status: active
        inet 10.0.0.38 netmask 0xffffff00 broadcast 10.0.0.255
enc0: flags=0<>
        index 2 priority 0 llprio 3
        groups: enc
        status: active
pflog0: flags=141<UP,RUNNING,PROMISC> mtu 33200
        index 4 priority 0 llprio 3
        groups: pflog
```

此示例仅显示一个物理以太网接口：`fxp0`。它上面配置了一个 IP，因此有 `inet 10.0.0.38 netmask 0xffffff00 broadcast 10.0.0.255` 这些值。`UP` 和 `RUNNING` 标志也已设置。

<a href="https://man.openbsd.org/netstart">netstart(8)</a> 脚本在引导时使用 <a href="https://man.openbsd.org/hostname.if">hostname.if(5)</a> 文件配置网络接口，其中 "if" 替换为每个接口的全名。上面的示例将使用文件 `/etc/hostname.fxp0`，其中包含以下选项：

```text
inet 10.0.0.38 255.255.255.0
```

这个 `hostname.fxp0` 文件也有一个交互式的等效命令：

```shell
# ifconfig fxp0 10.0.0.38 255.255.255.0
```

其他几个接口默认启用。这些是服务于各种功能的虚拟接口。以下手册页描述了它们：

- <a href="https://man.openbsd.org/enc">enc(4)</a> - 封装接口
- <a href="https://man.openbsd.org/lo">lo(4)</a> - 回环接口
- <a href="https://man.openbsd.org/pflog">pflog(4)</a> - 包过滤日志接口

可以使用 <a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a> 的 `create` 子命令添加其他虚拟接口。

<h3 id="Setup.gateway">默认主机名和网关</h3>

`/etc/myname` 和 `/etc/mygate` 文件由 <a href="https://man.openbsd.org/netstart">netstart(8)</a> 脚本读取。这两个文件分别由一行组成，指定系统的完全限定域名和网关主机的地址。`/etc/mygate` 文件不需要在所有系统上都存在。有关更多详细信息，请参阅 <a href="https://man.openbsd.org/myname">myname(5)</a>。

<h3 id="Setup.resolver">DNS 解析</h3>

DNS 解析由 <a href="https://man.openbsd.org/resolv.conf">resolv.conf(5)</a> 文件控制，该文件由 <a href="https://man.openbsd.org/resolvd">resolvd(8)</a> 管理。

```shell
$ cat /etc/resolv.conf
search example.com
nameserver 125.2.3.4
nameserver 125.2.3.5
lookup file bind
```

在这里，默认域名将是 `example.com`，将有两个名称服务器（`125.2.3.4` 和 `125.2.3.5`），并且在查询名称服务器之前将先查询 <a href="https://man.openbsd.org/hosts">hosts(5)</a> 文件。

<h3 id="Setup.activate">激活更改</h3>

从这里开始，重启系统或运行 <a href="https://man.openbsd.org/netstart">netstart(8)</a> 脚本：

```shell
# sh /etc/netstart
```

如果接口已经配置，运行此脚本时可能会产生一些警告。使用 <a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a> 确保接口设置正确。

尽管可以在运行中的 OpenBSD 系统上完全重新配置网络，但在进行任何重大更改后，建议重新启动。

<h3 id="Setup.chkroute">检查路由</h3>

可以通过 <a href="https://man.openbsd.org/netstat">netstat(1)</a> 或 <a href="https://man.openbsd.org/route">route(8)</a> 检查路由。

```shell
$ netstat -rn
Routing tables

Internet:
Destination        Gateway            Flags     Refs     Use    Mtu  Prio Iface
default            10.0.0.1           UGS         4       16      -    12 fxp0
224/4              127.0.0.1          URS         0        0  32768     8 lo0
127/8              127.0.0.1          UGRS        0        0  32768     8 lo0
127.0.0.1          127.0.0.1          UH          2       15  32768     1 lo0
10.0.0/24          link#1             UC          1        4      -     4 fxp0
10.0.0.1           aa:0:4:0:81:d      UHL         1       11      -     1 fxp0
10.0.0.38          127.0.0.1          UGHS        0        0      -     1 lo0

$ route show
Routing tables

Internet:
Destination        Gateway            Flags     Refs     Use    Mtu  Prio Iface
default            10.0.0.1           UGS         4       16      -    12 fxp0
base-address.mcast localhost          URS         0        0  32768     8 lo0
loopback           localhost          UGRS        0        0  32768     8 lo0
localhost          localhost          UH          2       15  32768     1 lo0
10.0.0/24          link#1             UC          1        4      -     4 fxp0
10.0.0.1           aa:0:4:0:81:d      UHL         1       11      -     1 fxp0
10.0.0.38          localhost          UGHS        0        0      -     1 lo0
```

<h3 id="Setup.aliases">在接口上设置别名</h3>

要在接口上设置 IP 别名，只需编辑其 <a href="https://man.openbsd.org/hostname.if">hostname.if(5)</a> 文件。

在此示例中，向接口 `dc0` 添加了两个别名，该接口已配置为 `192.168.0.2`，子网掩码为 `255.255.255.0`。

```shell
$ cat /etc/hostname.dc0
inet 192.168.0.2 255.255.255.0
inet alias 192.168.0.3 255.255.255.255
inet alias 192.168.0.4 255.255.255.255
```

创建此文件后，<a href="#Setup.activate">运行 netstart</a> 或重启。要查看所有别名，请使用 `ifconfig -A`。

<h2 id="DHCP">动态主机配置协议 (DHCP)</h2>

动态主机配置协议 (DHCP) 是一种自动配置网络接口的方法。OpenBSD 可以作为配置其他机器的 DHCP 服务器，也可以作为由 DHCP 服务器配置的 DHCP 客户端。

<h3 id="DHCPclient">DHCP 客户端</h3>

要使用 <a href="https://man.openbsd.org/dhcpleased">dhcpleased(8)</a>，请编辑网络接口的 <a href="https://man.openbsd.org/hostname.if">hostname.if(5)</a> 文件。<a href="#Wireless">无线网络</a>章节解释了如何设置无线接口。对于以太网接口，一行就足够了：

```text
inet autoconf
```

OpenBSD 将在启动时从 DHCP 服务器获取其 IP 地址、默认网关和 DNS 服务器。其他选项可以在 <a href="https://man.openbsd.org/dhcpleased.conf">dhcpleased.conf(5)</a> 中指定。

要从命令行通过 DHCP 获取 IP，请运行：

```shell
# ifconfig xl0 inet autoconf
```

将 `xl0` 替换为接口名称。

<h3 id="DHCPserver">DHCP 服务器</h3>

要将 OpenBSD 用作 DHCP 服务器，请在启动时启用 <a href="https://man.openbsd.org/dhcpd">dhcpd(8)</a> 守护进程：

```shell
# rcctl enable dhcpd
```

在下次引导时，dhcpd 将运行并附加到所有在 <a href="https://man.openbsd.org/dhcpd.conf">dhcpd.conf(5)</a> 中具有有效配置的网卡。也可以通过显式命名来指定单个接口：

```shell
# rcctl set dhcpd flags em1 em2
```

一个 `etc/dhcpd.conf` 文件示例可能如下所示：

```text
# Home
subnet 192.168.1.0 netmask 255.255.255.0 {
	option domain-name-servers 192.168.1.2;
	option routers 192.168.1.1;
	range 192.168.1.3 192.168.1.50;
}

# Guests
subnet 172.16.0.0 netmask 255.255.255.0 {
	option domain-name-servers 1.2.3.4, 5.6.7.8;
	option routers 172.16.0.1;
	range 172.16.0.2 172.16.0.254;
}
```

此示例中有两个子网：家庭网络和访客网络。客户端将自动获得 IP 地址，并指向其各自配置文件部分中指定的网关和 DNS 服务器。有关更多选项，请参阅 <a href="https://man.openbsd.org/dhcp-options">dhcp-options(5)</a>。

<h3 id="PXE">PXE 引导 (i386, amd64)</h3>

预引导执行环境 (PXE) 是一种仅使用网络引导系统的标准方法。客户端的支持 PXE 的网卡在<a href="./06_disk-setup.md#BootAmd64">引导过程</a>开始时广播 DHCP 请求，除了接收基本的 IP/DNS 信息外，还会获得一个用于引导的文件。在 OpenBSD 上，此文件称为 <a href="https://man.openbsd.org/pxeboot">pxeboot(8)</a>，通常由 <a href="https://man.openbsd.org/tftpd">tftpd(8)</a> 提供服务。

<h2 id="Wireless">无线网络</h2>

OpenBSD 支持<a href="https://man.openbsd.org/?query=wireless&amp;apropos=1">多种无线芯片组</a>。更多支持的设备可以在 <a href="https://man.openbsd.org/usb">usb(4)</a> 和 <a href="https://man.openbsd.org/pci">pci(4)</a> 中找到。其支持的确切范围在驱动程序手册页中有描述。

以下网卡支持基于主机的接入点 (HostAP) 模式，允许它们用作<a href="../002_pf/17_example1.md">无线接入点</a>：
<!-- XXXrelease - current list? grep -lRI IEEE80211_C_HOSTAP src/sys/dev -->

- <a href="https://man.openbsd.org/acx">acx(4)</a> - TI ACX100/ACX111
- <a href="https://man.openbsd.org/ath">ath(4)</a> - Atheros 802.11a/b/g
- <a href="https://man.openbsd.org/athn">athn(4)</a> - Atheros 802.11/a/g/n 设备
- <a href="https://man.openbsd.org/bwfm">bwfm(4)</a> - Broadcom 和 Cypress IEEE 802.11a/ac/b/g/n 无线网络设备
- <a href="https://man.openbsd.org/pgt">pgt(4)</a> - Conexant/Intersil Prism GT Full-MAC 802.11a/b/g
- <a href="https://man.openbsd.org/ral">ral(4)</a> 和 <a href="https://man.openbsd.org/ural">ural(4)</a> - Ralink Technology RT25x0 802.11a/b/g
- <a href="https://man.openbsd.org/rtw">rtw(4)</a> - Realtek 8180 802.11b
- <a href="https://man.openbsd.org/rum">rum(4)</a> - Ralink Technology RT2501USB
- <a href="https://man.openbsd.org/wi">wi(4)</a> - Prism2/2.5/3

<a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a> 的 `media` 子命令显示网络接口的介质功能。对于无线设备，它显示支持的 802.11a/b/g/n 介质模式和支持的操作模式（`hostap`、`ibss`、`monitor`）。例如，要查看接口 `ath0` 的介质功能，请运行：

```shell
$ ifconfig ath0 media
```

为了使用某些无线网卡，可能需要通过 <a href="https://man.openbsd.org/fw_update">fw_update(8)</a> 获取固件文件。一些制造商拒绝允许<a href="./01_introduction.md#ReallyFree">免费</a>分发其固件，因此无法将其包含在 OpenBSD 中。

另一个可以考虑的选项：使用传统的网卡和外部桥接无线接入点作为基于 OpenBSD 的防火墙。

<h3>配置无线适配器</h3>

基于受支持芯片的适配器可以像任何其他网络接口一样使用。要将 OpenBSD 系统连接到现有的无线网络，请使用 <a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a> 实用程序。

无线客户端的 <a href="https://man.openbsd.org/hostname.if">hostname.if(5)</a> 文件示例如下：

```text
nwid puffyuberalles wpakey passwordhere
inet autoconf
```

或者，对于多个接入点：

```text
join home-net wpakey passwordhere
join work-net wpakey passwordhere
join cafe-wifi
inet autoconf
```

请注意，`inet autoconf` 应在其他配置行之后，因为网络适配器在配置完成之前无法发送 DHCP 请求。

在运行时，可以使用 ifconfig `nwid` 命令临时覆盖自动接入点选择：

```shell
ifconfig ath0 nwid home-net
```

如果给定的网络已存在于加入列表中，其 WPA 密码将在需要时自动使用。

ifconfig `-nwid` 命令将接口移回自动接入点选择模式：

```shell
ifconfig ath0 -nwid
```

<h3>无线适配器链路聚合 (Trunking)</h3>

Trunk 是由一个或多个网络接口组成的虚拟接口。在本节中，我们的示例将是一台具有有线 <a href="https://man.openbsd.org/bge">bge0</a> 接口和无线 <a href="https://man.openbsd.org/iwn">iwn0</a> 接口的笔记本电脑。我们将使用这两个接口构建一个 <a href="https://man.openbsd.org/trunk">trunk(4)</a> 接口。有线和无线接口必须连接到同一个二层网络。

为此，我们首先激活两个物理端口，然后将它们分配给 `trunk0`。

```shell
# echo up > /etc/hostname.bge0
```

然而，无线接口需要更多的配置。它需要连接到我们受 WPA 保护的无线网络：

```shell
$ cat /etc/hostname.iwn0
nwid puffynet wpakey mysecretkey
up
```

现在，我们的 trunk 接口定义如下：

```shell
$ cat /etc/hostname.trunk0
trunkproto failover trunkport bge0
trunkport iwn0
inet autoconf
```

Trunk 设置为 `failover`（故障转移）模式，因此可以使用任一接口。如果两者都可用，它将首选 `bge0` 端口，因为那是第一个添加到 trunk 设备的端口。

<h2 id="Bridge">设置网桥</h2>

<a href="https://man.openbsd.org/bridge">bridge(4)</a> 是两个或多个独立网络之间的链路。与路由器不同，数据包透明地通过网桥：两个网段对两侧的节点来说就像是一个网段。网桥只转发必须从一个网段传递到另一个网段的数据包，因此，网桥中的接口可能有也可能没有自己的 IP 地址。如果有，该接口实际上有两种操作模式：一种作为网桥的一部分，另一种作为独立的网卡。如果两个接口都没有 IP 地址，网桥将传递网络数据，但无法从外部进行维护（这可能是一个特性）。

<h3>充当 DHCP 服务器的网桥</h3>

假设我们有一个系统，它有四个 <a href="https://man.openbsd.org/vr">vr(4)</a> 接口，`vr0` 到 `vr3`。我们要将 `vr1`、`vr2` 和 `vr3` 桥接在一起，留下 `vr0` 作为上行链路。我们还想通过桥接接口上的 DHCP 提供 IP 地址。作为 DHCP 服务器和上行路由器，该机器需要在桥接网络上拥有一个 IP 地址。

不可能直接将 IP 地址分配给网桥接口。IP 地址应添加到其中一个成员接口，但我们不能使用物理接口，因为链路可能会断开，在这种情况下地址将无法访问。幸运的是，<a href="https://man.openbsd.org/vether">vether(4)</a>（虚拟以太网）驱动程序可用于此目的。我们将把它添加到网桥，为其分配 IP 地址，并让 <a href="https://man.openbsd.org/dhcpd">dhcpd(8)</a> 在那里监听。

- <a href="#DHCPserver">DHCP 服务器配置</a>在本节中不再赘述，但这里使用的寻址方案是相同的。
- 这也将是桥接网络的上行路由器，因此我们将使用 IP 地址 192.168.1.1 以匹配 DHCP 服务器配置。
- 我们不在此处介绍上行链路、路由或防火墙配置。

首先，将 `vr1`、`vr2` 和 `vr3` 接口标记为 up：

```shell
# echo up > /etc/hostname.vr1
# echo up > /etc/hostname.vr2
# echo up > /etc/hostname.vr3
```

然后创建 `vether0` 配置：

```shell
# echo 'inet 192.168.1.1 255.255.255.0 192.168.1.255' > /etc/hostname.vether0
```

配置网桥接口以包含所有上述接口：

```shell
$ cat /etc/hostname.bridge0
add vether0
add vr1
add vr2
add vr3
up
```

最后，我们让 DHCP 守护进程在 `vether0` 接口上监听：

```shell
# rcctl set dhcpd flags vether0
```

最后，重启系统。

<h3>在网桥上过滤</h3>

<a href="../002_pf/00_index.md">PF</a> 可用于限制通过网桥的流量。请记住，由于网桥的性质，相同的数据流经两个接口，因此只需在一个接口上进行过滤。

<h3>关于桥接的提示</h3>

- 通过使用 <a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a> 的 `blocknonip` 选项或在 <a href="https://man.openbsd.org/hostname.if">hostname.bridge0</a> 中使用该选项，可以防止非 IP 流量（如 IPX 或 NETBEUI）绕过过滤器。在某些情况下这可能很重要，但请注意，网桥适用于各种流量，而不仅仅是 IP。
- 桥接要求网卡处于混杂模式。它们监听 **所有** 网络流量，而不仅仅是发往该接口的流量。这将给处理器和总线带来比预期更高的负载。

<h2 id="Multipath">等价多路径路由 (ECMP)</h2>

等价多路径路由是指在路由表中针对同一网络（例如默认路由 `0.0.0.0/0`）拥有多条路由。当内核进行路由查找以确定将发往该网络的数据包发送到何处时，它可以从任何等价路由中进行选择。在大多数情况下，多路径路由用于提供冗余上行链路连接，例如，连接到互联网的冗余连接。

<a href="https://man.openbsd.org/route">route(8)</a> 命令用于在路由表中添加/更改/删除路由。添加多路径路由时使用 `-mpath` 参数。

```shell
# route add -mpath default 10.130.128.1
# route add -mpath default 10.132.0.1
```

验证路由：

```shell
# netstat -rnf inet | grep default
default     10.130.128.1      UGS       2      134      -     fxp1
default     10.132.0.1        UGS       0      172      -     fxp2
```

在这个例子中，我们可以看到一条默认路由指向 `10.130.128.1`，可通过 `fxp1` 接口访问，另一条指向 `10.132.0.1`，可通过 `fxp2` 访问。

由于 <a href="https://man.openbsd.org/mygate">mygate(5)</a> 文件尚不支持多路径默认路由，因此上述命令应添加到 `fxp1` 和 `fxp2` 接口的 <a href="https://man.openbsd.org/hostname.if">hostname.if(5)</a> 文件的底部。然后应删除 `/etc/mygate` 文件。

```shell
$ tail -1 /etc/hostname.fxp1
!route add -mpath default 10.130.128.1
$ tail -1 /etc/hostname.fxp2
!route add -mpath default 10.132.0.1
```

最后，不要忘记通过启用适当的 <a href="https://man.openbsd.org/sysctl.8">sysctl(8)</a> 变量来激活多路径路由的使用。

```shell
# sysctl net.inet.ip.multipath=1
# sysctl net.inet6.ip6.multipath=1
```

务必编辑 <a href="https://man.openbsd.org/sysctl.conf">sysctl.conf(5)</a> 以使更改永久生效。

现在尝试 traceroute 到不同的目的地。内核将在每个多路径路由上对流量进行负载均衡。

```shell
# traceroute -n 154.11.0.4
traceroute to 154.11.0.4 (154.11.0.4), 64 hops max, 60 byte packets
 1  10.130.128.1  19.337 ms  18.194 ms  18.849 ms
 2  154.11.95.170  17.642 ms  18.176 ms  17.731 ms
 3  154.11.5.33  110.486 ms  19.478 ms  100.949 ms
 4  154.11.0.4  32.772 ms  33.534 ms  32.835 ms

# traceroute -n 154.11.0.5
traceroute to 154.11.0.5 (154.11.0.5), 64 hops max, 60 byte packets
 1  10.132.0.1  14.175 ms  14.503 ms  14.58 ms
 2  154.11.95.38  13.664 ms  13.962 ms  13.445 ms
 3  208.38.16.151  13.964 ms  13.347 ms  13.788 ms
 4  154.11.0.5  30.177 ms  30.95 ms  30.593 ms
```

有关如何选择路由的更多信息，请参阅 <a href="https://www.ietf.org/rfc/rfc2992.txt">RFC2992</a>，“等价多路径算法分析”。

值得注意的是，如果多路径路由使用的接口关闭（即失去载波），内核仍将尝试使用指向该接口的路由转发数据包。这种流量当然会被丢弃，最终无处可去。强烈建议使用 <a href="https://man.openbsd.org/ifstated">ifstated(8)</a> 来检查不可用的接口并相应地调整路由表。

<h2 id="NFS">使用 NFS</h2>

网络文件系统 (NFS) 用于通过网络共享文件系统。

本节将介绍简单的 NFS 设置步骤。该示例详细说明了局域网上的服务器，客户端访问局域网上的 NFS。它不涉及保护 NFS 的安全。

<h3>设置 NFS 服务器</h3>

首先，在服务器上启用 <a href="https://man.openbsd.org/portmap">portmap(8)</a>、<a href="https://man.openbsd.org/mountd">mountd(8)</a> 和 <a href="https://man.openbsd.org/nfsd">nfsd(8)</a> 服务：

```shell
# rcctl enable portmap mountd nfsd
```

然后配置将提供的文件系统列表。

在此示例中，我们有一台 IP 地址为 `10.0.0.1` 的服务器。此服务器将仅向其自己子网内的客户端提供 NFS 服务。这在以下 <a href="https://man.openbsd.org/exports">exports(5)</a> 文件中配置：

```shell
$ cat /etc/exports
/docs -alldirs -ro -network=10.0.0 -mask=255.255.255.0
```

本地文件系统 `/docs` 将通过 NFS 提供。`-alldirs` 选项指定客户端将能够挂载 `/docs` 下的任何点以及 `/docs` 本身。`-ro` 选项指定客户端将仅被授予只读访问权限。最后两个参数指定只有使用子网掩码 `255.255.255.0` 的 `10.0.0.0` 网络内的客户端才被授权挂载此文件系统。

现在启动服务器相关服务：

```shell
# rcctl start portmap mountd nfsd
```

如果在 NFS 已经运行时对 `/etc/exports` 进行了更改，必须让 mountd 知道这一点。

```shell
# rcctl reload mountd
```

<h3>挂载 NFS 文件系统</h3>

NFS 文件系统应通过 <a href="https://man.openbsd.org/mount_nfs">mount(8)</a> 挂载，更具体地说是 <a href="https://man.openbsd.org/mount_nfs">mount_nfs(8)</a>。

要将主机 `10.0.0.1` 上的 `/docs` 文件系统挂载到本地文件系统 `/mnt`，请运行：

```shell
# mount -t nfs 10.0.0.1:/docs /mnt
```

要在启动时挂载该文件系统，请向 <a href="https://man.openbsd.org/fstab">fstab(5)</a> 追加一行，如下所示：

```shell
# echo '10.0.0.1:/docs /mnt nfs ro,nodev,nosuid 0 0' >> /etc/fstab
```

在此行末尾使用 `0 0` 很重要，这样计算机就不会在启动时尝试对 NFS 文件系统进行 <a href="https://man.openbsd.org/fsck">fsck(8)</a>。

当以 root 用户身份访问 NFS 挂载时，服务器会自动将 root 的访问权限映射为用户名 `nobody` 和组 `nobody`。在考虑文件权限时，了解这一点很重要。例如，对于具有这些权限的文件：

```text
-rw-------    1 root     wheel           0 Dec 31 03:00 _daily.B20143
```

如果此文件位于 NFS 共享上，并且 root 用户尝试从 NFS 客户端访问此文件，访问将被拒绝。

root 映射到的用户和组可以通过 NFS 服务器上的 <a href="https://man.openbsd.org/exports">exports(5)</a> 文件进行配置。

<h3>检查 NFS 统计信息</h3>

确保 NFS 正常运行的一件事是检查所有守护进程是否已在 RPC 中正确注册。为此，请使用 <a href="https://man.openbsd.org/rpcinfo">rpcinfo(8)</a>。

```shell
$ rpcinfo -p 10.0.0.1
   program vers proto   port
    100000    2   tcp    111  portmapper
    100000    2   udp    111  portmapper
    100005    1   udp    633  mountd
    100005    3   udp    633  mountd
    100005    1   tcp    916  mountd
    100005    3   tcp    916  mountd
    100003    2   udp   2049  nfs
    100003    3   udp   2049  nfs
    100003    2   tcp   2049  nfs
    100003    3   tcp   2049  nfs
```

有一些实用程序允许管理员查看 NFS 正在发生的情况，例如 <a href="https://man.openbsd.org/showmount">showmount(8)</a> 和 <a href="https://man.openbsd.org/nfsstat">nfsstat(1)</a>。

