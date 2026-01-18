## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 用于验证网关的用户 Shell (authpf)</span>

---

- <a href="#intro">简介</a>
- <a href="#config">配置</a>
    - <a href="#enable">启用 authpf</a>
    - <a href="#linkmain">将 authpf 链接到主规则集</a>
    - <a href="#loadrules">配置加载的规则</a>
    - <a href="#acl">访问控制列表</a>
    - <a href="#message">显示登录消息</a>
    - <a href="#shell">将 authpf 指定为用户的 Shell</a>
- <a href="#loginclass">创建 authpf 登录类别</a>
- <a href="#wholog">查看谁已登录</a>
- <a href="#example">示例</a>

---

<h2 id="intro">简介</h2>

<a href="https://man.openbsd.org/authpf">authpf(8)</a> 实用程序是一个用于认证网关的用户 Shell。认证网关就像普通的网络网关（也称为路由器）一样，只是用户必须先对其进行身份验证，然后才允许其流量通过。当用户的 Shell 设置为 `/usr/sbin/authpf` 并且他或她使用 SSH 登录时，authpf 将对活动的 <a href="https://man.openbsd.org/pf">pf(4)</a> 规则集进行必要的更改，以便用户的流量可以通过过滤器和/或使用 NAT/重定向进行转换。一旦用户注销或会话断开，authpf 将删除为该用户加载的所有规则，并终止该用户打开的所有状态连接。因此，用户通过网关传递流量的能力仅在用户保持 SSH 会话打开时才存在。

authpf 将用户的规则加载到唯一的<a href="./09_anchors.md">锚 (anchor)</a> 点中。锚的命名方式是将用户名和 authpf 进程 ID 组合成 `username(PID)` 格式。每个用户的锚都存储在 `authpf` 锚中，而 `authpf` 锚本身又锚定到主规则集。完整的锚路径变为：

```text
main_ruleset/authpf/username(PID)
```

authpf 加载的规则可以在每个用户的基础上或全局基础上进行配置。

authpf 的使用示例包括：

- 要求用户在允许互联网访问之前进行身份验证。
- 授予某些用户（如管理员）访问网络受限部分的权限。
- 仅允许已知用户从无线网段访问网络的其余部分或互联网。
- 允许在家或在路上的员工访问公司网络上的资源。办公室外的用户不仅可以打开对公司网络的访问权限，还可以根据他们用于身份验证的用户名重定向到特定资源（例如他们自己的桌面）。
- 在图书馆或其他拥有公共互联网终端的场所，可以配置 PF 以允许访客用户进行有限的互联网访问。然后可以为注册用户提供完全访问权限。

authpf 工具通过 <a href="https://man.openbsd.org/syslogd">syslogd(8)</a> 记录每个成功认证的用户的用户名和 IP，以及登录会话的开始和结束时间。通过使用这些信息，管理员可以确定谁在何时登录，并让用户对其网络流量负责。

<h2 id="config">配置</h2>

这里概述了配置 authpf 所需的基本步骤。有关 authpf 配置的完整描述，请参阅<a href="https://man.openbsd.org/authpf">手册页</a>。

<h3 id="enable">启用 authpf</h3>

如果 `/etc/authpf/authpf.conf` 配置文件不存在，authpf 实用程序将不会运行。该文件可以为空，但除非它存在，否则 authpf 将在用户成功认证后立即退出。

以下配置指令可以放置在 `authpf.conf` 中：

- `anchor=name` - 使用指定的<a href="./09_anchors.md">锚</a> `name` 而不是 `authpf`
- `table=name` - 使用指定的<a href="./03_tables.md">表</a> `name` 而不是 `authpf_users`

<h3 id="linkmain">将 authpf 链接到主规则集</h3>

要将 authpf 链接到主规则集，请使用 `anchor` 规则：

```text
anchor "authpf/*"
```

`anchor` 规则放置在规则集中的位置就是 PF 将从主规则集分支出来评估 authpf 规则的位置。

<h3 id="loadrules">配置加载的规则</h3>

authpf 规则可以从以下两个文件之一加载：

- `/etc/authpf/users/$USER/authpf.rules`
- `/etc/authpf/authpf.rules`

第一个文件包含仅在用户 `$USER`（替换为用户的用户名）登录时加载的规则。当特定用户（如管理员）需要一组与默认集不同的规则时，使用每用户规则配置。第二个文件包含默认规则，这些规则为没有任何自己的 `authpf.rules` 文件的用户加载。如果存在特定于用户的文件，它将覆盖默认文件。必须至少存在一个文件，否则 authpf 将不会运行。

规则具有与任何其他 PF 规则集相同的语法，唯一的例外是 authpf 允许使用两个预定义的宏：

- `$user_ip` - 登录用户的 IP 地址
- `$user_id` - 登录用户的用户名

建议的做法是使用 `$user_ip` 宏仅允许来自经过身份验证的用户计算机的流量通过网关。

除了 `$user_ip` 宏之外，authpf 还将利用 `authpf_users` 表（如果存在）来存储所有经过身份验证的用户的 IP 地址。请务必在使用前定义该表：

```text
table <authpf_users> persist
pass in on egress proto tcp from <authpf_users> to port smtp
```

此表应仅用于旨在应用于所有经过身份验证的用户的规则中。

<h3 id="acl">访问控制列表</h3>

可以通过在 `/etc/authpf/banned` 目录中创建一个与用户名匹配的文件来阻止特定用户访问 authpf。该文件的内容将在 authpf 断开连接之前显示给用户。这提供了一种方便的方式来通知用户为何被禁止访问以及联系谁来恢复访问。

相反，也可以通过将用户名放在 `/etc/authpf/authpf.allow` 文件中来仅授予特定用户访问权限。如果文件不存在，或者如果在其中输入了 "`*`"，则 authpf 将允许任何通过 SSH 成功登录的用户访问，只要他们没有被明确禁止。

如果 authpf 无法确定用户名是被允许还是被拒绝，它将打印一条简短的消息，然后断开用户的连接。`/etc/authpf/banned` 中的文件始终覆盖 `/etc/authpf/authpf.allow` 中的条目。

<h3 id="message">显示登录消息</h3>

每当用户成功向 authpf 认证时，都会打印一条问候语，表明用户已通过认证。

```text
Hello charlie. You are authenticated from host "198.51.100.10"
```

可以通过在 `/etc/authpf/authpf.message` 中放置自定义消息来补充此消息。该文件的内容将显示在默认欢迎消息之后。

<h3 id="shell">将 authpf 指定为用户的 Shell</h3>

为了使 authpf 工作，必须将其指定为用户的登录 Shell。当用户成功向 <a href="https://man.openbsd.org/sshd">sshd(8)</a> 认证时，authpf 将作为用户的 Shell 执行。然后它将检查用户是否被允许使用 authpf，从适当的文件加载规则等。

有几种方法可以将 authpf 指定为用户的 Shell：

1.  使用 <a href="https://man.openbsd.org/chsh">chsh(1)</a>、<a href="https://man.openbsd.org/vipw">vipw(8)</a>、<a href="https://man.openbsd.org/useradd">useradd(8)</a>、<a href="https://man.openbsd.org/usermod">usermod(8)</a> 等为每个用户手动指定。
2.  通过将用户分配到登录类别并在 <a href="https://man.openbsd.org/login.conf">login.conf(5)</a> 中更改该类别的 `shell` 选项。

<h2 id="loginclass">创建 authpf 登录类别</h2>

当在拥有普通用户帐户和 authpf 用户帐户的系统上使用 authpf 时，为 authpf 用户使用单独的登录类别可能是有益的。这允许在全局基础上对这些帐户进行某些更改，并且还允许对普通帐户和 authpf 帐户实施不同的策略。可以设置的一些策略示例：

- **shell** - 指定用户的登录 Shell。这可用于强制用户的 Shell 为 `authpf`，而不管 <a href="https://man.openbsd.org/passwd">passwd(5)</a> 数据库中的条目如何。
- **welcome** - 指定用户登录时显示的 <a href="https://man.openbsd.org/motd">motd(5)</a> 文件。这对于显示仅与 authpf 用户相关的消息很有用。

登录类别在 <a href="https://man.openbsd.org/login.conf">login.conf(5)</a> 文件中创建。OpenBSD 附带了一个定义如下的 authpf 登录类别：

```text
authpf:\
    :welcome=/etc/motd.authpf:\
    :shell=/usr/sbin/authpf:\
    :tc=default:
```

通过编辑用户 passwd(5) 数据库条目的 `class` 字段将用户分配到登录类别。执行此操作的一种方法是使用 <a href="https://man.openbsd.org/chsh">chsh(1)</a> 命令。

<h2 id="wholog">查看谁已登录</h2>

一旦用户成功登录并且 authpf 调整了 PF 规则，authpf 就会更改其进程标题以指示登录用户的用户名和 IP 地址：

```shell
# ps -ax | grep authpf
23664 p0  Is+     0:00.11 -authpf: charlie@192.168.1.3 (authpf)
```

这里用户 `charlie` 从机器 192.168.1.3 登录。通过向 authpf 进程发送 SIGTERM 信号，可以强制用户注销。为该用户加载的任何规则都将被删除，并且该用户打开的任何状态连接都将被终止。

```shell
# kill -TERM 23664
```

<h2 id="example">示例</h2>

authpf 工具正被用于 OpenBSD 网关上，以对无线网络（大型校园网络的一部分）的用户进行身份验证。经过身份验证的用户将被允许 SSH 输出、浏览网页和访问校园 DNS 服务器，假设他们不在禁止名单上。

`/etc/authpf/authpf.rules` 文件包含以下规则：

```text
wifi_if = "wi0"

pass in quick on $wifi_if \
   proto tcp from $user_ip to any port { ssh, http, https }
```

管理用户 `charlie` 除了浏览网页和使用 SSH 外，还需要能够访问校园 SMTP 和 POP3 服务器。在 `/etc/authpf/users/charlie/authpf.rules` 中设置了以下规则：

```text
wifi_if = "wi0"
smtp_server = "10.0.1.50"
pop3_server = "10.0.1.51"

pass in quick on $wifi_if \
   proto tcp from $user_ip to $smtp_server port smtp
pass in quick on $wifi_if \
   proto tcp from $user_ip to $pop3_server port pop3
pass in quick on $wifi_if \
   proto tcp from $user_ip to port { ssh, http, https }
```

主 `/etc/pf.conf` 规则集设置如下：

```text
wifi_if = "wi0"
ext_if  = "fxp0"
dns_servers = "{ 10.0.1.56, 10.0.2.56 }"

table <authpf_users> persist

block drop all

pass out quick on $ext_if \
   inet proto { tcp, udp, icmp } from { $wifi_if:network, $ext_if }

pass in quick on $wifi_if \
   inet proto tcp from $wifi_if:network to $wifi_if port ssh

pass in quick on $wifi_if \
   inet proto { tcp, udp } from <authpf_users> to $dns_servers port domain

anchor "authpf/*" in on $wifi_if
```

该规则集非常简单，执行以下操作：

- 阻止一切（默认拒绝）
- 允许外部接口上来自无线网络和网关本身的传出 TCP、UDP 和 ICMP 流量
- 允许来自无线网络发往网关本身的传入 SSH 流量（此规则是允许用户登录所必需的。）
- 允许来自所有经过身份验证的 authpf 用户的传入 DNS 请求访问校园 DNS 服务器
- 为无线接口上的传入流量创建锚点 "authpf"

主规则集背后的想法是阻止一切，并允许尽可能少的流量通过。流量可以在外部接口上自由流出，但默认拒绝策略会阻止其进入无线接口。经过身份验证的用户的流量被允许在无线接口上进入，然后通过网关流入网络的其余部分。通篇使用 `quick` 关键字，以便当新连接通过网关时，PF 不必评估每个命名规则集。
