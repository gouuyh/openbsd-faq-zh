## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">FAQ - 虚拟专用网络 (VPN)</span>

---

- <a href="#introduction">简介</a>
- <a href="#authentication">认证</a>
- <a href="#server">配置 IKEv2 服务器</a>
    - <a href="#site2site">构建站点到站点 VPN</a>
- <a href="#clientikev2">连接到 IKEv2 OpenBSD VPN</a>
    - <a href="#clientopenbsd">使用 OpenBSD 客户端</a>
    - <a href="#clientandroid">使用 Android 客户端</a>
        - <a href="#autheap">使用 MSCHAP-V2 进行 EAP 认证</a>
        - <a href="#authx509">使用 X.509 证书认证</a>
    - <a href="#clientwindows">使用 Windows 客户端</a>
- <a href="#clientikev1">连接到 IKEv1/L2TP OpenBSD VPN</a>

---

<h2 id="introduction">简介</h2>

OpenBSD 附带了 <a href="https://man.openbsd.org/iked">iked(8)</a>，这是一个现代的、权限分离的 IKEv2 服务器。它既可以充当响应端 (responder)，即接收连接请求的服务器，也可以充当发起端 (initiator)，即向响应端发起连接的客户端。<a href="https://man.openbsd.org/ikectl">ikectl(8)</a> 实用程序用于控制服务器，服务器从 <a href="https://man.openbsd.org/iked.conf">iked.conf(5)</a> 文件获取配置。

<a href="https://man.openbsd.org/ikectl">ikectl(8)</a> 实用程序还允许你为 IKEv2 对等方维护一个简单的 X.509 证书颁发机构 (CA)。

IKEv1 服务器 (<a href="https://man.openbsd.org/isakmpd">isakmpd(8)</a>) 也是可用的，结合 <a href="https://man.openbsd.org/npppd">npppd(8)</a>，它允许你在无法部署 IKEv2 的地方构建 IKEv1/L2TP VPN。

原生 WireGuard 支持也可通过 <a href="https://man.openbsd.org/wg">wg(4)</a> 设备获得。正如手册所解释的，它的配置方式与 OpenBSD 中的所有其他<a href="./07_networking.md">网络接口</a>相同。

<h2 id="authentication">认证</h2>

<a href="https://man.openbsd.org/iked">iked(8)</a> 支持以下认证方法：

- 预共享密钥 (PSK)（不推荐）
- RSA 和 ECDSA 公钥：连接到 iked、RouterOS 和其他一些实现时设置简单
- EAP MSCHAPv2（服务器端使用 X.509 证书）：iked 仅在“响应端”（服务器）侧支持此功能
- X.509 证书：Windows、Android 和 Apple 客户端通常需要

默认情况下，引导时会在 `/etc/iked/local.pub` 中生成 RSA 公钥，私钥存储在 `/etc/iked/private/local.key` 中。

<h2 id="server">配置 IKEv2 服务器</h2>

<h3 id="site2site">构建站点到站点 VPN</h3>

这可以通过交换默认提供的 RSA 公钥来实现：第一个系统（"server1"）上的 `/etc/iked/local.pub` 应复制到第二个系统（"server2"）上的 `/etc/iked/pubkeys/fqdn/server1.domain`。然后，第二个系统上的 `/etc/iked/local.pub` 应复制到第一个系统上的 `/etc/iked/pubkeys/fqdn/server2.domain`。请将 "serverX.domain" 替换为你自己的 FQDN。

从这点出发，假设 server1 的公网 IP 为 `192.0.2.1`，内部网络为 `10.0.1.0/24`；server2 的公网 IP 为 `198.51.100.1`，内部网络为 `10.0.2.0/24`。

为了使发起端能够到达响应端，响应端上应打开 `isakmp` UDP 端口。如果其中一个对等方在 NAT 后面，响应端上还应打开 `ipsec-nat-t` UDP 端口。如果两个对等方都有公网 IP，则应允许 ESP 协议。

```text
pass in log on $ext_if proto udp from 198.51.100.1 to 192.0.2.1 port {isakmp, ipsec-nat-t} tag IKED
pass in log on $ext_if proto esp from 198.51.100.1 to 192.0.2.1 tag IKED
```

server1（充当响应端）的 `/etc/iked.conf` 配置示例可能如下所示：

```text
ikev2 'server1_rsa' passive esp \
        from 10.0.1.0/24 to 10.0.2.0/24 \
        local 192.0.2.1 peer 198.51.100.1 \
        srcid server1.domain
```

server2（充当发起端）的简单配置：

```text
ikev2 'server2_rsa' active esp \
        from 10.0.2.0/24 to 10.0.1.0/24 \
        peer 192.0.2.1 \
        srcid server2.domain
```

使用 `iked -dv` 可以帮助你理解交换过程。在这个例子中，响应端位于 NAT 后面：

```text
server1# iked -dv
...
ikev2_recv: IKE_SA_INIT request from initiator 198.51.100.1:500 to 192.0.2.1:500 policy 'server1_rsa' id 0, 510 bytes
ikev2_msg_send: IKE_SA_INIT response from 192.0.2.1:500 to 198.51.100.1:500 msgid 0, 451 bytes
ikev2_recv: IKE_AUTH request from initiator 198.51.100.1:4500 to 192.0.2.1:4500 policy 'server1_rsa' id 1, 800 bytes
ikev2_msg_send: IKE_AUTH response from 192.0.2.1:4500 to 198.51.100.1:4500 msgid 1, 720 bytes, NAT-T
sa_state: VALID -> ESTABLISHED from 198.51.100.1:4500 to 192.0.2.1:4500 policy 'server1_rsa'
```

在发起端：

```text
server2# iked -dv
...
ikev2_msg_send: IKE_SA_INIT request from 0.0.0.0:500 to 192.0.2.1:500 msgid 0, 510 bytes
ikev2_recv: IKE_SA_INIT response from responder 192.0.2.1:500 to 198.51.100.1:500 policy 'server2_rsa' id 0, 451 bytes
ikev2_msg_send: IKE_AUTH request from 198.51.100.1:4500 to 192.0.2.1:4500 msgid 1, 800 bytes, NAT-T
ikev2_recv: IKE_AUTH response from responder 192.0.2.1:4500 to 198.51.100.1:4500 policy 'server2_rsa' id 1, 720 bytes
sa_state: VALID -> ESTABLISHED from 192.0.2.1:4500 to 198.51.100.1:4500 policy 'server2_rsa'
```

可以使用 <a href="https://man.openbsd.org/ipsecctl">ipsecctl(8)</a> 查看 IPsec 流：

```text
server1# ipsecctl -sa
FLOWS:
flow esp in from 10.0.2.0/24 to 10.0.1.0/24 peer 198.51.100.1 srcid FQDN/server1.domain dstid FQDN/server2.domain type use
flow esp out from 10.0.1.0/24 to 10.0.2.0/24 peer 198.51.100.1 srcid FQDN/server1.domain dstid FQDN/server2.domain type require
flow esp out from ::/0 to ::/0 type deny

SAD:
esp tunnel from 192.0.2.1 to 198.51.100.1 spi 0xabb5968a auth hmac-sha2-256 enc aes-256
esp tunnel from 198.51.100.1 to 192.0.2.1 spi 0xb1fc90b8 auth hmac-sha2-256 enc aes-256

server2# ipsecctl -sa
FLOWS:
flow esp in from 10.0.1.0/24 to 10.0.2.0/24 peer 192.0.2.1 srcid FQDN/server2.domain dstid FQDN/server1.domain type use
flow esp out from 10.0.2.0/24 to 10.0.1.0/24 peer 192.0.2.1 srcid FQDN/server2.domain dstid FQDN/server1.domain type require
flow esp out from ::/0 to ::/0 type deny

SAD:
esp tunnel from 192.0.2.1 to 198.51.100.1 spi 0xabb5968a auth hmac-sha2-256 enc aes-256
esp tunnel from 198.51.100.1 to 192.0.2.1 spi 0xb1fc90b8 auth hmac-sha2-256 enc aes-256
```

这样，两个内部网络应该能够相互访问。它们之间的流量在解封装后应出现在 `enc0` 接口上，并可以据此进行过滤。在该示例中，`tag VPN` 已添加到策略中：

```text
# pfctl -vvsr|grep VPN
@16 pass log on enc0 tagged VPN
# tcpdump -nei pflog0 rnr 16
00:03:26.793522 rule 16/(match) pass in on enc0: 10.0.2.24 > 10.0.1.13: icmp: echo request
```

一些警告：

- 如果响应端未设置 `srcid`，则 iked 默认会尝试使用与其 FQDN 匹配的密钥。
- 响应端**不需要**设置 `local`，它只是确保 `iked` 在正确的接口上监听。
- 响应端**不需要**设置 `peer`，它只是确保连接来自受信任的 IP。

如果 VPN 端点需要访问远程内部网络，或者内部网络需要访问远程 VPN 端点，则必须在双方设置额外的流：

- 从本地公网 IP 到远程网络
- 从内部网络到远程公网 IP

响应端配置将如下所示：

```text
ikev2 'server1_rsa' passive esp \
        from 10.0.1.0/24 to 10.0.2.0/24 \
        from 10.0.1.0/24 to 198.51.100.1 \
        from 192.0.2.1 to 10.0.2.0/24 \
        local 192.0.2.1 peer 198.51.100.1 \
        srcid server1.domain
```

发起端配置将是：

```text
ikev2 'server2_rsa' active esp \
        from 10.0.2.0/24 to 10.0.1.0/24 \
        from 10.0.2.0/24 to 192.0.2.1 \
        from 198.51.100.1 to 10.0.1.0/24 \
        peer 192.0.2.1 \
        srcid server2.domain
```

<h2 id="clientikev2">连接到 IKEv2 OpenBSD VPN</h2>

作为 *移动用户 (road warrior)* 连接到 IKEv2 VPN 与前面的情况类似，不同之处在于发起端通常计划通过响应端路由其互联网流量，响应端将对其应用 NAT，以便发起端的流量看起来像是来自响应端的公网 IP。

根据用例，由于所有流量都将通过响应端，因此必须确保发起端配置为使用它可以访问的 DNS 服务器（可能是响应端上的 DNS 服务器）。

<h3 id="clientopenbsd">使用 OpenBSD 客户端</h3>

在我们的示例中，`10.0.5.0/24` 网络用于支持 VPN。实际的内部 IP 地址将由 iked 自动安装在 lo1 接口上。我们假设客户端的公网 IP 为 `203.0.113.2`。

与前面的示例一样，交换默认提供的 RSA 公钥足以在响应端和发起端之间建立简单的认证：第一个系统（"server1"）上的 `/etc/iked/local.pub` 应复制到第二个系统（"roadwarrior"）上的 `/etc/iked/pubkeys/fqdn/server1.domain`。然后，roadwarrior 系统上的 `/etc/iked/local.pub` 应复制到第一个系统上的 `/etc/iked/pubkeys/fqdn/roadwarrior`。请将 "serverX.domain" 替换为你自己的 FQDN。

响应端 <a href="https://man.openbsd.org/iked.conf">iked.conf(5)</a> 创建从任何目的地到地址池动态 IP 租约的流（将在运行时决定），并用 ROADW 标记数据包：

```text
ikev2 'responder_rsa' passive esp \
        from any to dynamic \
        local 192.0.2.1 peer any \
        srcid server1.domain \
        config address 10.0.5.0/24 \
        tag "ROADW"
```

响应端需要向发起端提供 IP 地址。这是通过 `config` 指令实现的。当使用 `config address` 选项时，`to dynamic` 将被替换为分配的动态 IP 地址。

它还需要允许来自任何主机的 IPsec（因为客户端可能从任何地方连接），允许 `enc0` 上标记为 ROADW 的流量并对其应用 NAT：

```text
pass in log on $ext_if proto udp from any to 192.0.2.1 port {isakmp, ipsec-nat-t} tag IKED
pass in log on $ext_if proto esp from any to 192.0.2.1 tag IKED
pass log on enc0 tagged ROADW
match out log on $ext_if inet tagged ROADW nat-to $ext_if
```

发起端配置全局流以将其所有流量发送到响应端，并告诉它使用名为 "roadwarrior" 的密钥标识自己：

```text
ikev2 'roadwarrior' active esp \
        from dynamic to any \
        peer 192.0.2.1 \
        srcid roadwarrior \
        dstid server1.domain \
        request address any \
        iface lo1
```

发起端使用 `request address any` 选项向响应端请求动态 IP 地址。`iface lo1` 选项指定将在其上安装接收到的地址和相应路由的接口。

响应端应该为移动用户客户端配置适当的 NAT。

可以使用 `ikectl decouple` 优雅地停止发起端上的 VPN（`iked` 仍在运行，等待 `ikectl couple` 以便重新连接到响应端），或者使用 `ikectl reset sa && rcctl stop iked` 永久停止 `iked` 并确保没有遗留流。

<h3 id="clientandroid">使用 Android 客户端</h3>

默认的 Android VPN 客户端仅支持 IKEv1。要使用 IKEv2，<a href="https://www.strongswan.org/">strongSwan</a> 是一个选择。

还需要设置 PKI 和 X.509 证书，以便发起端可以验证响应端通告的证书：

```shell
server1# ikectl ca vpn create
server1# ikectl ca vpn install
certificate for CA 'vpn' installed into /etc/iked/ca/ca.crt
CRL for CA 'vpn' installed to /etc/iked/crls/ca.crl
server1# ikectl ca vpn certificate server1.domain create
server1# ikectl ca vpn certificate server1.domain install
writing RSA key
server1# cp /etc/iked/ca/ca.crt /var/www/htdocs/
```

在 Android 设备上，浏览 `http://192.0.2.1/ca.crt` 并在 strongSwan 客户端中导入 CA 证书。

从这点出发，有几种选择来向响应端认证发起端：

- 带有用户名/密码的 EAP
- X.509 证书

<h4 id="autheap">使用 MSCHAP-V2 进行 EAP 认证</h4>

响应端配置需要指定用户名/密码列表，并指定它将使用 `eap "mschap-v2"`（这是目前支持的唯一 EAP 方法），如下所示：

```text
user 'android' 'password'
ikev2 'responder_eap' passive esp \
        from any to dynamic \
        local 192.0.2.1 peer any \
        srcid server1.domain \
        eap "mschap-v2" \
        config address 10.0.5.0/24 \
        config name-server 192.0.2.1 \
        tag "ROADW"
```

在 strongSwan 客户端中，配置一个新的配置文件：

- *VPN 类型* 选择 *IKEv2 EAP*
- *服务器* 字段填写 `192.0.2.1`
- 在响应端配置中设置的登录名/密码值
- *CA 证书* 字段选择新导入的 `CN=VPN CA` 证书
- *用户身份* 字段填写 `client1.domain`
- *服务器身份* 字段填写 `server1.domain`（在“高级设置”下）

这样，Android 设备就可以连接到响应端，使用 CA 证书验证响应端证书，使用 EAP 登录名/密码向响应端验证自己，在 `10.0.5.0/24` 网络中获取地址，并且其所有流量都通过 VPN，使用 `192.0.2.1` 作为其 DNS 服务器。

<h4 id="authx509">使用 X.509 证书认证</h4>

对于此方法，为客户端生成证书，安装在 `iked ca` 中，导出为归档文件，并且 .pfx 文件应在线提供以便客户端可以安装它。.pfx 文件捆绑了：

- X.509 证书
- 使用 RSA 加密的 X.509 私钥
- 用于加密 X.509 私钥的私有 RSA 密钥
- 公共 RSA 密钥

```shell
server1# ikectl ca vpn certificate client1.domain create
server1# cp /etc/ssl/vpn/client1.domain.crt /etc/iked/certs/
server1# ikectl ca vpn certificate client1.domain export
server1# tar -C /tmp -xzf client1.domain.tgz *pfx
server1# cp /tmp/export/client1.domain.pfx /var/www/htdocs/client1.domain.pfx
```

配置新配置文件时，必须在 strongSwan 客户端中导入 CA 公共证书和客户端证书包。

响应端配置稍微简单一些，因为不需要指定 `eap` 也不需要设置用户名/密码：

```text
ikev2 'responder_x509' passive esp \
        from any to dynamic \
        local 192.0.2.1 peer any \
        srcid server1.domain \
        config address 10.0.5.0/24 \
        config name-server 192.0.2.1 \
        tag "ROADW"
```

在 strongSwan 客户端中，配置一个新的配置文件，使用：

- *VPN 类型* 选择 *IKEv2 certificate*
- *服务器* 字段填写 `192.0.2.1`
- *用户证书* 字段选择新导入的 `CN=client1.domain` 证书
- *用户身份* 字段填写 `client1.domain`
- *服务器身份* 字段填写 `server1.domain`（在“高级设置”下）

就像在 EAP 案例中一样，Android 设备现在可以连接到响应端并使用 VPN。

<h3 id="clientwindows">使用 Windows 客户端</h3>

Windows 7 及更高版本提供了一个 IKEv2 发起端，它也需要使用 X.509 证书，这些证书需要导出为 .pfx/.p12 包并导入到本地计算机（不是用户帐户）证书存储中，包括 CA 和客户端证书。可以使用图形化的 Microsoft 管理控制台（在命令行中输入 mmc 并添加证书管理单元作为计算机帐户）或在 Windows 10 中使用 certutil 命令。将 `ca.crt` 导入 *受信任的根证书颁发机构* 存储，将 `ClientIP.p12` 导入 *个人* 存储。

<a href="https://wiki.strongswan.org/projects/strongswan/wiki/Win7Certs">StrongSwan</a> 项目对此主题有很好的文档，并附有截图。

Windows 不容易允许为客户端设置 `srcid` 参数，因此客户端证书的 CN 字段必须与发送给响应端的客户端 FQDN 匹配，或者默认情况下与其 IP 匹配。还要求响应端上的 `srcid` 与响应端 FQDN（或其 IP，如果不使用 FQDN）匹配 - 否则可能会遇到可怕的 `error 3801`。Libreswan 项目有关于这些<a href="https://libreswan.org/wiki/Interoperability#Windows_Certificate_requirements">要求</a>的<a href="https://libreswan.org/wiki/VPN_server_for_remote_clients_using_IKEv2#Common_Windows_7_client_errors">宝贵细节</a>。

导入证书后，配置新的 VPN 连接：

- 在 *常规* 选项卡中，目标主机名填写响应端 FQDN
- 在 *安全* 选项卡中，类型选择 *IKEv2*
- 认证选择“计算机证书”或“EAP 认证”
- 如果使用 EAP，连接时将提示输入登录名/密码

响应端配置文件将类似于 Android 案例。

```text
user 'windows' 'password'
ikev2 'responder_eap' passive esp \
        from any to dynamic \
        local 192.0.2.1 peer any \
        srcid server1.domain.fqdn \
        eap "mschap-v2" \
        config address 10.0.5.0/24 \
        config name-server 192.0.2.1 \
        tag "ROADW"
```

默认情况下，所有 Windows 流量现在都将通过 IKEv2 VPN。

在撰写本文时，当前版本的 Windows 默认使用弱加密 (3DES/SHA1)。这可以通过 PowerShell 命令 `Set-VpnConnectionIPsecConfiguration` 进行更正。

<h2 id="clientikev1">连接到 IKEv1/L2TP VPN</h2>

有时，你无法控制 VPN 服务器，只能选择连接到 IKEv1 服务器。在这种情况下，需要 `xl2tpd` 第三方软件包来充当 L2TP 客户端。

首先需要启用 <a href="https://man.openbsd.org/isakmpd">isakmpd(8)</a> 和 `ipsec` 服务，以便启动守护进程并在引导时加载 <a href="https://man.openbsd.org/ipsec.conf">ipsec.conf(5)</a> 配置文件：

```shell
# rcctl enable ipsec
# rcctl enable isakmpd
# rcctl set isakmpd flags -K
```

以下 <a href="https://man.openbsd.org/ipsec.conf">ipsec.conf(5)</a> 配置应允许连接到 `A.B.C.D` 处的 IKEv1 服务器，并提供 PSK，仅允许 L2TP 的 UDP 端口 1701：

```text
ike dynamic esp transport proto udp from egress to A.B.C.D port l2tp \
        psk mekmitasdigoat
```

启动 <a href="https://man.openbsd.org/isakmpd">isakmpd(8)</a> 并使用 <a href="https://man.openbsd.org/ipsecctl">ipsecctl(8)</a> 加载 <a href="https://man.openbsd.org/ipsec.conf">ipsec.conf(5)</a> 应该允许你可视化配置的安全关联 (SA) 和流：

```shell
# rcctl start isakmpd
# ipsecctl -f /etc/ipsec.conf
# ipsecctl -sa
FLOWS:
flow esp in proto udp from A.B.C.D port l2tp to W.X.Y.Z peer A.B.C.D srcid my.client.fqdn dstid A.B.C.D/32 type use
flow esp out proto udp from W.X.Y.Z to A.B.C.D port l2tp peer A.B.C.D srcid my.client.fqdn dstid A.B.C.D/32 type require

SAD:
esp transport from A.B.C.D to W.X.Y.Z spi 0x0d16ad1c auth hmac-sha1 enc aes
esp transport from W.X.Y.Z to A.B.C.D spi 0xcd0549ba auth hmac-sha1 enc aes
```

如果情况并非如此，可能需要调整阶段 1 (Main) 和阶段 2 (Quick) 参数，此时双方交换加密参数以商定最佳可用组合。理想情况下，这些参数应由远程服务器管理员提供，并应在 <a href="https://man.openbsd.org/ipsec.conf">ipsec.conf(5)</a> 中使用：

```text
ike dynamic esp transport proto udp from egress to A.B.C.D port l2tp \
        main auth "hmac-sha1" enc "3des" group modp1024 \
        quick auth "hmac-sha1" enc "aes" \
        psk mekmitasdigoat
```

一旦 IKEv1 隧道启动并运行，就需要配置 L2TP 隧道。OpenBSD 默认不提供 L2TP 客户端，因此需要安装 `xl2tpd`。

```shell
# pkg_add xl2tpd
```

请参阅 `/usr/local/share/doc/pkg-readmes/xl2tpd` 以获取有关如何正确设置 L2TP 客户端的说明。
