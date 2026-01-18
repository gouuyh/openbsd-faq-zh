## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - FTP 相关问题</span>

---

- <a href="#modes">FTP 模式</a>
- <a href="#client">防火墙后的 FTP 客户端</a>
- <a href="#server">PF "自保护" FTP 服务器</a>
- <a href="#natserver">受运行 NAT 的外部 PF 防火墙保护的 FTP 服务器</a>
- <a href="#info">关于 FTP 的更多信息</a>
- <a href="#tftp-proxy">代理 TFTP</a>

---

<h2 id="modes">FTP 模式</h2>

FTP 是一种协议，其历史可以追溯到互联网还是一个小型、友好的计算机集合且每个人都互相认识的时候。那时，不需要过滤或严格的安全性。FTP 不是为过滤、通过防火墙或与 NAT 一起工作而设计的。

FTP 可以以两种方式之一使用：被动 (passive) 或主动 (active)。通常，选择主动还是被动是为了确定谁有防火墙问题。

在主动 FTP 中，当用户连接到远程 FTP 服务器并请求信息或文件时，FTP 服务器会建立一个新的连接回到客户端以传输请求的数据。这被称为 *数据连接*。首先，FTP 客户端选择一个随机端口来接收数据连接。客户端将它选择的端口号发送给 FTP 服务器，并在该端口上监听传入连接。然后，FTP 服务器发起连接到客户端地址的选定端口并传输数据。这对于试图从 NAT 网关后面访问 FTP 服务器的用户来说是一个问题。由于 NAT 的工作方式，FTP 服务器通过连接到 NAT 网关外部地址的选定端口来发起数据连接。NAT 机器将收到此连接，但由于其状态表中没有该数据包的映射，它将丢弃该数据包并且不会将其传递给客户端。

在被动模式 FTP（OpenBSD <a href="https://man.openbsd.org/ftp">ftp(1)</a> 客户端的默认模式）中，客户端请求服务器选择一个随机端口来监听数据连接。服务器将其选择的端口通知客户端，客户端连接到此端口以传输数据。不幸的是，这并不总是可行或可取的，因为 FTP 服务器前面的防火墙可能会阻止传入的数据连接。要强制使用主动模式 FTP，请使用 `ftp` 的 `-A` 标志，或通过在 "`ftp>`" 提示符下发出命令 "`passive off`" 将被动模式设置为 "off"。

<h2 id="client">防火墙后的 FTP 客户端</h2>

如前所述，FTP 不能很好地通过 NAT 和防火墙。

PF 通过将 FTP 流量转移到 FTP 代理服务器来解决这种情况。此过程通过主动添加所需的规则到 PF 并在完成时通过<a href="./09_anchors.md">锚 (anchor)</a> 系统将其移除，从而起到“引导”FTP 流量通过 NAT 网关/防火墙的作用。PF 使用的 FTP 代理是 <a href="https://man.openbsd.org/ftp-proxy">ftp-proxy(8)</a>。

要激活它，请在 `pf.conf` 的规则部分早期放置类似这样的内容：

```text
pass in quick on $int_if inet proto tcp to port 21 divert-to 127.0.0.1 port 8021
```

这会将 FTP 从客户端转移到 ftp-proxy 程序，该程序在服务器的 8021 端口上监听。

规则部分还需要一个锚：

```text
anchor "ftp-proxy/*"
```

代理服务器必须在 OpenBSD 机器上启用并运行。

```shell
# rcctl enable ftpproxy
# rcctl start  ftpproxy
```

为了支持来自某些（挑剔的）客户端的主动模式连接，可能需要 `-r` 标志。

<h2 id="server">PF "自保护" FTP 服务器</h2>

在这种情况下，PF 运行在 FTP 服务器本身而不是专用的防火墙计算机上。当服务被动连接时，FTP 将使用随机选择的高位 TCP 端口来接收传入数据。默认情况下，OpenBSD 原生的 <a href="https://man.openbsd.org/ftpd">ftpd(8)</a> 使用 49152 到 65535 的范围。显然，这些必须与端口 21（FTP 控制端口）一起通过过滤规则：

```text
pass in on egress proto tcp to port 21
pass in on egress proto tcp to port > 49151
```

如果需要，可以收紧该端口范围。在 <a href="https://man.openbsd.org/ftpd">ftpd(8)</a> 的情况下，这是使用 <a href="https://man.openbsd.org/sysctl">sysctl(8)</a> 变量 `net.inet.ip.porthifirst` 和 `net.inet.ip.porthilast` 完成的。

<h2 id="natserver">受运行 NAT 的外部 PF 防火墙保护的 FTP 服务器</h2>

在这种情况下，防火墙除了不阻止所需的端口外，还必须将流量重定向到 FTP 服务器。

ftp-proxy 可以运行在一种模式下，使其将所有 FTP 连接转发到特定的 FTP 服务器。代理将设置为在防火墙的端口 21 上监听，并将所有连接转发到后端服务器。

```shell
# rcctl set ftpproxy flags -R 10.10.10.1 -p 21 -b 192.168.0.1
```

这里 10.10.10.1 是实际 FTP 服务器的 IP 地址，21 是 ftp-proxy 将监听的端口，192.168.0.1 是防火墙上代理将绑定的地址。

现在是 `pf.conf` 规则：

```text
ext_ip = "192.168.0.1"
ftp_ip = "10.10.10.1"
match out on egress inet from $int_if nat-to (egress)
anchor "ftp-proxy/*"
pass in  on  egress inet proto tcp to $ext_ip port 21
pass out on $int_if inet proto tcp to $ftp_ip port 21 user _ftp_proxy
```

这里允许外部接口上的入站端口 21 连接，以及到 FTP 服务器的相应出站连接。出站规则中的 `user _ftp_proxy` 添加确保仅允许由 ftp-proxy 发起的连接。

<h2 id="info">关于 FTP 的更多信息</h2>

关于过滤 FTP 以及 FTP 一般如何工作的更多信息可以在<a href="https://web.archive.org/web/20170812032619/http://www.pintday.org/whitepapers/ftp-review.shtml">这份白皮书</a>中找到。

<h2 id="tftp-proxy">代理 TFTP</h2>

简单文件传输协议 (TFTP) 在通过防火墙时遇到了一些与 FTP 相同的限制。幸运的是，PF 有一个名为 <a href="https://man.openbsd.org/tftp-proxy">tftp-proxy(8)</a> 的 TFTP 辅助代理。

tftp-proxy 的设置方式与上面的<a href="#client">防火墙后的 FTP 客户端</a>部分中的 ftp-proxy 非常相似。

```text
match out on egress inet from $int_if nat-to (egress)
anchor "tftp-proxy/*"
pass in  quick on $int_if inet proto udp from $lan to port tftp \
    divert-to 127.0.0.1 port 6969
pass out quick on $ext_if inet proto udp from $lan to port tftp \
    group _tftp_proxy divert-reply
```

上述规则允许从内部网络到外部网络上的 TFTP 服务器的出站 TFTP。

最后一步是启用并启动 tftp-proxy。

```shell
# rcctl enable tftpproxy
# rcctl start  tftpproxy
```
