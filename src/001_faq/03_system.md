# [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">FAQ - 系统管理</span>

---

- <a href="#Patches">安全更新</a>
- <a href="#rc">系统守护进程</a>
- <a href="#doas">以其他用户身份执行命令</a>
- <a href="#vipw">编辑密码文件</a>
- <a href="#LostPW">重置 Root 密码</a>
- <a href="#OpenNTPD">时钟同步</a>
- <a href="#TimeZone">时区</a>
- <a href="#locales">字符集和本地化</a>
- <a href="#SMT">对称多线程，或者“为什么只使用了一半的 CPU？”</a>
- <a href="#SKey">使用 S/Key</a>
- <a href="#Dir">目录服务</a>
    - <a href="#YP_secure">YP 安全注意事项</a>
    - <a href="#YP_server">设置 YP 服务器</a>
    - <a href="#YP_client">设置 YP 客户端</a>

---

<h2 id="Patches">安全更新</h2>

当发现严重 Bug 时，修复程序将尽快提交到 -current 树（并在<a href="./05_building-from-source.md#Flavors">快照构建</a>中提供）。然后，它将以<a href="https://www.openbsd.org/errata.html">勘误表 (errata)</a> 的形式向后移植到最近的两个 OpenBSD 版本，详细信息将发送到 <a href="https://lists.openbsd.org/cgi-bin/mj_wwwusr?func=lists-long-full&amp;extra=announce">announce</a> 邮件列表。

可以通过几种不同的方式获取这些修复：

- **应用二进制补丁**（适用于 amd64, arm64, i386）
  <br>
  <a href="https://man.openbsd.org/syspatch">syspatch(8)</a> 实用程序可用于在受支持的 OpenBSD 版本上升级任何需要安全或可靠性修复的文件。这是使基础系统保持最新状态的最快、最简单的方法。

- **将系统更新到 -stable**
  <br>
  使用 <a href="https://www.openbsd.org/anoncvs.html">CVS</a> 获取（或更新）你的源代码树，然后<a href="https://www.openbsd.org/stable.html">重新编译</a>内核和用户空间。

- **单独修补受影响的文件**
  <br>
  虽然应用勘误表页面中的修复程序通常比 CVS 检出/更新和重建所需的时间更少，但没有通用的说明集可供遵循。有时你必须修补并重新编译一个应用程序，有时则更多。

- **将系统升级到 -current**
  <br>
  由于所有修复都应用于 -current 代码库，将系统更新到最新的<a href="./05_building-from-source.md#Snapshots">快照</a>是一次性获取所有修复的好方法。但是，运行 -current 并不适合所有人。

通过<a href="./04_package.md">软件包 (packages)</a> 安装的第三方软件的安全修复通常只会向后移植到最新的发行版。要获取它们，请执行以下操作之一：

- **使用二进制软件包**（适用于 amd64, arm64, i386, sparc64）
  <br>
  为发行版和 -stable 系统提供更新的二进制软件包，以解决安全问题或进行其他主要修复。只需使用 `-u` 标志调用 <a href="https://man.openbsd.org/pkg_add">pkg_add(1)</a> 即可获取新文件。

- **使用 -stable ports 树**
  <br>
  获取（或更新）你的 <a href="../003_ports/00_index.md">ports 树</a>，运行 `/usr/ports/infrastructure/bin/pkg_outdated` 脚本列出任何需要重建的软件包，并在受影响的 port 目录中执行 `make update`。在某些情况下，现有的 port 需要在重建之前卸载。

- **将系统升级到 -current**
  <br>
  -current 的二进制软件包会定期重建，这些新软件包将包含任何安全修复。

要接收 port 更新的警报，请考虑关注 <a href="https://lists.openbsd.org/cgi-bin/mj_wwwusr?func=lists-long-full&amp;extra=ports-changes">ports-changes</a> 邮件列表。

<h2 id="rc">系统守护进程</h2>

系统守护进程（或“服务”）由 <a href="https://man.openbsd.org/rc">rc(8)</a> 脚本通过 <a href="https://man.openbsd.org/rc.d">rc.d(8)</a> 启动、停止和控制。

OpenBSD 附带的大多数守护进程和服务在引导时由 <a href="https://cvsweb.openbsd.org/src/etc/rc.conf?rev=HEAD">/etc/rc.conf</a> 中定义的变量控制。你会看到类似这样的行：

```text
httpd_flags=NO
```

这表明 <a href="https://man.openbsd.org/httpd">httpd(8)</a> 不会在引导时由 <a href="https://man.openbsd.org/rc">rc(8)</a> 启动。每一行都有注释，显示该守护进程或服务的常用标志。

不要直接修改 <a href="https://man.openbsd.org/rc.conf">rc.conf(8)</a>。相反，使用 <a href="https://man.openbsd.org/rcctl">rcctl(8)</a> 实用程序来维护 `/etc/rc.conf.local` 文件。这使得未来的升级更容易，因为所有更改都在一个升级期间不会被触及的文件中。

例如，要启动用于 CPU 缩放的 <a href="https://man.openbsd.org/apmd">apmd(8)</a> 守护进程，可以执行：

```shell
# rcctl enable apmd
# rcctl set apmd flags -A
# rcctl start apmd
```

<h2 id="doas">以其他用户身份执行命令</h2>

<a href="https://man.openbsd.org/doas">doas(1)</a> 工具允许系统管理员许可某些用户以其他用户身份运行特定命令。普通用户可以运行管理命令，只需验证自己的身份，而无需 root 密码。

例如，如果配置得当，以下命令将显示 root 的 <a href="https://man.openbsd.org/crontab.5">crontab(5)</a> 文件：

```shell
$ doas -u root crontab -l
```

默认情况下，由 <a href="https://man.openbsd.org/doas">doas(1)</a> 调用的命令会记录到 `/var/log/secure`。有关配置示例，请查看 <a href="https://man.openbsd.org/doas.conf">doas.conf(5)</a> 手册。

<h2 id="vipw">编辑密码文件</h2>

OpenBSD 的主密码文件位于 `/etc/master.passwd`，只有 root 可读。<a href="https://man.openbsd.org/pwd_mkdb">pwd_mkdb(8)</a> 工具从主文件生成全局可读的 `/etc/passwd` 和密码数据库（`/etc/pwd.db` 和 `/etc/spwd.db`）。格式在 <a href="https://man.openbsd.org/passwd.5">passwd(5)</a> 中描述。

始终使用 <a href="https://man.openbsd.org/vipw">vipw(8)</a> 来编辑密码文件。编辑完成后，它首先会对更改进行健全性检查，然后重新创建 `/etc/passwd` 和密码数据库，最后将副本安装到位以替换原始的 `/etc/master.passwd` 文件。

<h2 id="LostPW">重置 Root 密码</h2>

如果忘记了 root 密码，重新获得访问权限的基本过程是启动到单用户模式，挂载 `/` 和 `/usr` 分区，并运行 <a href="https://man.openbsd.org/passwd">passwd(1)</a> 来更改 root 密码。

- **启动到单用户模式。**
  这部分过程因使用的 <a href="https://www.openbsd.org/plat.html">平台</a> 而异。对于 amd64 和 i386，<a href="./06_disk-setup.md#BootAmd64">第二阶段引导加载程序</a>会暂停几秒钟，让你有机会向内核提供参数。在这里，可以将 `-s` 标志传递给 <a href="https://man.openbsd.org/OpenBSD-current/i386/boot">boot(8)</a>：

```text
probing: pc0 com0 com1 mem[638K 1918M a20=on]
disk: hd0+ hd1+
>> OpenBSD/amd64 BOOT 3.33
boot> boot -s
```

- **挂载分区。**
  `/` 和 `/usr` 都需要以读写方式挂载。

```shell
# fsck -p / && mount -uw /
# fsck -p /usr && mount /usr
```

- **更改 root 密码。**
  由于处于单用户模式，你已经拥有 root 权限，因此不会提示你提供当前密码。

```shell
# passwd
```

- **启动到多用户模式。**
  这可以通过按 CTRL-D 恢复正常引导过程或输入 `reboot` 来完成。

<h2 id="OpenNTPD">时钟同步</h2>

<a href="https://www.openntpd.org">OpenNTPD</a> 是一种安全且简单的 NTP 兼容方式，可让你的计算机拥有准确的时间。<a href="https://man.openbsd.org/ntpd">ntpd(8)</a> 守护进程默认启用，并将根据从 NTP 对等方接收的数据设置时钟。一旦时钟被准确设置，它将使用 <a href="https://man.openbsd.org/ntpd.conf">ntpd.conf(5)</a> 中指定的配置时间服务器保持高精度。在引导时，`ntpd` 只会将时钟向前跳变。如果你的时钟必须向后调整，请使用 <a href="https://man.openbsd.org/date">date(1)</a> 手动设置时钟。

要将 OpenNTPD 用作服务器，请在 <a href="https://man.openbsd.org/ntpd.conf">ntpd.conf(5)</a> 文件中添加 `listen on *` 行并重启守护进程。你也可以指示它仅在特定地址或接口上监听。

当你有 <a href="https://man.openbsd.org/ntpd">ntpd(8)</a> 在监听时，其他机器可能无法立即同步它们的时钟。这是因为在本地时钟同步到合理的稳定水平之前，不会提供时间信息。一旦达到此水平，`/var/log/daemon` 中将出现 "clock is now synced"（时钟现已同步）消息。

<h2 id="TimeZone">时区</h2>

默认情况下，OpenBSD 假定你的硬件时钟设置为协调世界时 (UTC) 而不是本地时间。这在<a href="./02_installation.md#Multibooting">多重引导</a>时可能会导致问题。许多其他操作系统可以配置为执行相同的操作，从而完全避免此问题。

如果将硬件时钟设置为 UTC 是个问题，你可以通过 <a href="https://man.openbsd.org/sysctl.conf">sysctl.conf(5)</a> 更改 OpenBSD 的默认行为。例如，将以下内容放入 `/etc/sysctl.conf` 以配置 OpenBSD 使用设置为美国东部标准时间（比 UTC 晚 5 小时，即减去 300 分钟）的硬件时钟：

```text
kern.utc_offset=-300
```

有关更多信息，请参阅 <a href="https://man.openbsd.org/sysctl.2#KERN_UTC_OFFSET~2">sysctl(2)</a>。

请注意，在使用上述配置引导 OpenBSD 之前，硬件时钟必须已经以所需的偏移量运行，否则系统时间将在引导时被错误调整。

通常，时区是在安装期间设置的。如果需要更改时区，可以在 `/usr/share/zoneinfo` 中创建一个指向相应时区文件的新符号链接。例如，将机器设置为使用 EST5EDT 作为新的本地时区：

```shell
# ln -fs /usr/share/zoneinfo/EST5EDT /etc/localtime
```

另请参阅 <a href="https://man.openbsd.org/date">date(1)</a> 手册。

<h2 id="locales">字符集和本地化</h2>

OpenBSD 基础系统完全支持 ASCII 字符集和编码，并部分支持 Unicode 字符集的 UTF-8 编码。基础系统不支持其他编码或字符集，但可以使用 ports 来处理它们。UTF-8 支持级别和默认编码配置因程序或库而异。

要在支持的地方使用 UTF-8 编码的 Unicode 字符集，请将 `LC_CTYPE` 环境变量设置为值 `en_US.UTF-8`：

- 如果通过 <a href="https://man.openbsd.org/xenodm">xenodm(1)</a> 登录，请在启动窗口管理器之前将 `export LC_CTYPE="en_US.UTF-8"` 添加到你的 `~/.xsession`。有关更多详细信息，请参阅<a href="./09_X.md#CustomizingX">自定义 X</a>。
- 如果通过文本控制台登录，请将 `export LC_CTYPE="en_US.UTF-8"` 添加到你的 `~/.profile`。文本控制台的 UTF-8 支持正在进行中，某些非 ASCII 字符可能无法正确显示。

使用 <a href="https://man.openbsd.org/ssh">ssh(1)</a> 登录远程系统时，`LC_CTYPE` 环境变量不会传播，在连接之前，你必须确保本地终端设置为远程服务器使用的字符编码。如果该编码未知或 OpenBSD 不支持，请确保使用默认的 <a href="https://man.openbsd.org/xterm">xterm(1)</a> 配置，并在连接后在远程 shell 中设置 `LC_CTYPE=en_US.UTF-8`。

OpenBSD 基础系统完全忽略除 `LC_CTYPE` 之外的所有与区域设置相关的环境变量；即使是 `LC_ALL` 和 `LANG` 也只影响字符编码。某些 ports 可能会遵循其他 `LC_*` 变量，但不建议使用它们或将 `LC_CTYPE` 设置为除 `C`、`POSIX` 或 `en_US.UTF-8` 之外的任何值。

<h2 id="SMT">对称多线程，或者“为什么只使用了一半的 CPU？”</h2>

某些 CPU 使用对称多线程 (SMT; Intel 的实现称为“超线程”)。在这种情况下，一个物理处理器向操作系统呈现一个额外的逻辑处理器——在 <a href="https://man.openbsd.org/dmesg">dmesg(8)</a> 和 <a href="https://man.openbsd.org/top">top(1)</a> 等工具中显示为单独的 CPU。这些逻辑处理器不具备完整的 CPU 资源，而是为了允许与多个并发进程共享单个核心的部分资源。

此功能可以提高某些工作负载的性能，但会降低其他工作负载的性能。

SMT 涉及各种 CPU 漏洞，特别是与推测执行有关的漏洞。这可能导致进程获取它们不应访问的其他进程的信息。为了缓解这种情况，OpenBSD 默认禁止在检测到的 SMT“虚拟”核心上运行代码。

可以通过将 sysctl `hw.smt` 设置为 `1` 来重新启用它们，但通常不建议这样做。

<h2 id="SKey">使用 S/Key</h2>

S/Key 是一个“一次性密码”认证系统。它通过哈希函数（<a href="https://man.openbsd.org/md5">md5</a>、<a href="https://man.openbsd.org/sha1">sha1</a> 或 <a href="https://man.openbsd.org/rmd160">rmd160</a>）从用户的秘密短语以及从服务器接收到的挑战生成一系列一次性（单次使用）密码。

> **警告**：一次性密码系统仅保护认证信息。它们不能防止网络窃听者获取私人信息。此外，如果你正在访问安全系统 A，建议你从另一个受信任的系统 B 进行操作，以确保没有人通过记录你的击键或捕获和/或伪造你终端设备上的输入和输出来获得对系统 A 的访问权限。

<h3>设置 S/Key</h3>

首先，目录 `/etc/skey` 必须存在。如果此目录不存在，请让超级用户通过执行以下操作创建它：

```shell
# skeyinit -E
```

然后使用 <a href="https://man.openbsd.org/skeyinit">skeyinit(1)</a> 初始化你的 S/Key。首先会提示你输入登录密码，然后会要求你输入 S/Key 秘密短语，该短语必须至少有 10 个字符长：

```shell
$ skeyinit
Password:
[Adding ericj with md5]
Enter new secret passphrase:
Again secret passphrase:

ID ericj skey is otp-md5 100 oshi45820
Next login password: HAUL BUS JAKE DING HOT HOG
```

注意最后两行中的信息。用于创建你的 S/Key 密码的程序是 <a href="https://man.openbsd.org/otp-md5">otp-md5(1)</a>，序列号是 `100`，密钥是 `oshi45820`。六个小单词 `HAUL BUS JAKE DING HOT HOG` 构成了序列号为 `100` 的 S/Key 密码。

<h3>生成 S/Key 密码</h3>

要生成下次登录的 S/Key 密码，请使用 <a href="https://man.openbsd.org/skeyinfo">skeyinfo(1)</a> 找出要运行的命令：

```shell
$ skeyinfo -v
otp-md5 95 oshi45820
$ otp-md5 95 oshi45820
Enter secret passphrase:
NOOK CHUB HOYT SAC DOLE FUME
```

为了生成 S/Key 密码列表，请执行：

```shell
$ otp-md5 -n 5 95 oshi45820
Enter secret passphrase:
91: SHIM SET LEST HANS SMUG BOOT
92: SUE ARTY YAW SEED KURD BAND
93: JOEY SOOT PHI KYLE CURT REEK
94: WIRE BOGY MESS JUDE RUNT ADD
95: NOOK CHUB HOYT SAC DOLE FUME
```

<h3>使用 S/Key 登录</h3>

这是一个使用 S/Key 登录到 `localhost` 上的 ftp 服务器的示例会话。要执行 S/Key 登录，请将 `:skey` 附加到你的登录名后。

```shell
$ ftp localhost
Connected to localhost.
220 oshibana.shin.ms FTP server (Version 6.5/OpenBSD) ready.
Name (localhost:ericj): ericj:skey
331- otp-md5 93 oshi45820
331 S/Key Password: JOEY SOOT PHI KYLE CURT REEK
[...]
230 User ericj logged in.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> quit
221 Goodbye.
```

同样，对于 <a href="https://man.openbsd.org/ssh">ssh(1)</a>：

```shell
$ ssh -l ericj:skey localhost
otp-md5 91 oshi45821
S/Key Password: SHIM SET LEST HANS SMUG BOOT
Last login: Thu Apr  7 12:21:48 on ttyp1 from 156.63.248.77
$
```

<h2 id="Dir">目录服务</h2>

OpenBSD 既可以用作包含用户凭据、组信息和其他网络相关数据的数据库的服务器，也可以用作客户端。

当然，你可以在 OpenBSD 上使用各种目录服务。但是 YP 是唯一可以直接使用标准 C 库函数（如 <a href="https://man.openbsd.org/getpwent">getpwent(3)</a>、<a href="https://man.openbsd.org/getgrent">getgrent(3)</a>、<a href="https://man.openbsd.org/gethostbyname">gethostbyname(3)</a> 等）访问的服务。因此，如果你将数据保存在 YP 数据库中，则无需将其复制到本地配置文件（如 <a href="https://man.openbsd.org/master.passwd">master.passwd(5)</a>）中即可使用它，例如用于验证系统用户。

YP 是一种与 Sun Microsystems NIS (网络信息系统) 兼容的目录服务。有关可用手册页的概述，请参阅 <a href="https://man.openbsd.org/yp">yp(8)</a>。请小心，某些操作系统包含名称相似但实际上不兼容的目录服务，例如 NIS+。

要使用 YP 以外的目录服务，你要么需要从目录填充本地配置文件，要么需要该目录的 YP 前端。例如，当你选择前者时，可以使用 `sysutils/login_ldap` port，而 <a href="https://man.openbsd.org/ypldap">ypldap(8)</a> 守护进程提供后者。

对于某些应用程序，只需使用 <a href="https://man.openbsd.org/rdist">rdist(1)</a>、<a href="https://man.openbsd.org/cron">cron(8)</a>、<a href="https://man.openbsd.org/scp">scp(1)</a> 或 `rsync`（可从 ports 获取）等工具在一组机器之间同步少量配置文件，就构成了全功能目录服务的简单且强大的替代方案。

<h3 id="YP_secure">YP 安全注意事项</h3>

出于兼容性原因，OpenBSD 实现的 YP 中内置的所有安全功能默认都是 *关闭* 的。即使它们全部打开，NIS 协议本质上仍然是不安全的，原因有二：所有数据（包括密码哈希等敏感数据）都在网络上未加密传输，并且客户端和服务器都无法可靠地验证对方的身份。

因此，在设置任何 YP 服务器之前，你应该考虑这些固有的安全缺陷在你的环境中是否可以接受。特别是，如果潜在攻击者可以物理访问你的网络，YP 是不充分的。任何获得对承载 YP 流量的网络段连接的任何计算机的 root 访问权限的人都可以绑定你的 YP 域并检索其数据。在某些情况下，通过 SSL 或 IPsec 隧道传递 YP 流量可能是一种选择。

<h3 id="YP_server">设置 YP 服务器</h3>

YP 服务器服务于一组称为“域”的客户端。你应该首先选择一个域名；它可以是任意字符串，不需要与 DNS 域名有任何关系。选择像 "Eepoo5vi" 这样的随机名称可以略微提高安全性，尽管这种效果主要是隐晦式安全。如果你需要维护几个不同的 YP 域，最好选择描述性的名称，如 "sales"、"marketing" 和 "research"，以防止因混淆而导致的系统管理错误。另外请注意，某些版本的 SunOS 要求使用主机的 DNS 域名，因此在包含此类主机的网络中，你的选择可能会受到限制。

使用 <a href="https://man.openbsd.org/domainname">domainname(1)</a> 实用程序设置域名，并将其放入 <a href="https://man.openbsd.org/defaultdomain">defaultdomain(5)</a> 文件中，以便在系统启动时自动设置。

```shell
# echo "puffynet" > /etc/defaultdomain
# domainname `cat /etc/defaultdomain`
```

使用交互式命令初始化 YP 服务器：

```shell
# ypinit -m
```

此时，还不需要指定从服务器。要添加从服务器，你可以稍后使用 `-u` 选项重新运行 <a href="https://man.openbsd.org/ypinit">ypinit(8)</a>。

为每个域设置至少一个从服务器对于避免服务中断很有用。例如，如果主服务器宕机或失去网络连接，尝试访问 YP 映射的客户端进程将无限期阻塞，直到它们收到请求的信息。因此，YP 服务中断通常会导致客户端主机完全无法使用，直到 YP 恢复服务。

决定在哪里存储用于生成 YP 映射的源文件。将服务器配置与被服务的配置分开有助于控制哪些信息将被服务，哪些不被服务，因此默认的 `/etc` 通常不是最佳选择。

更改源目录带来的唯一不便是，你将无法使用 <a href="https://man.openbsd.org/user">user(8)</a> 和 <a href="https://man.openbsd.org/group">group(8)</a> 等实用程序在 YP 域中添加、删除和修改用户和组。相反，你必须使用文本编辑器编辑配置文件。

要定义源目录，请编辑文件 `/var/yp/`domainname`/Makefile` 并更改 `DIR` 变量，例如：

```text
DIR=/etc/yp/src/puffynet
```

考虑自定义 `/var/yp/`domainname`/Makefile` 中的其他变量。有关详细信息，请参阅 <a href="https://man.openbsd.org/Makefile.yp">Makefile.yp(8)</a>。

例如，即使你使用默认源目录 `/etc`，你通常也不需要在所有客户端主机上拥有服务器上存在的所有帐户和组。特别是，不提供 root 帐户从而保持 root 密码哈希的机密性通常有利于安全。检查 `MINUID`、`MAXUID`、`MINGID` 和 `MAXGID` 的值并根据你的需要进行调整。

如果你所有的 YP 客户端都运行 OpenBSD 或 FreeBSD，可以通过在 `/var/yp/`domainname`/Makefile` 中设置 `UNSECURE=""` 来从 `passwd` 映射中排除加密密码。

不再建议编辑模板文件 `/var/yp/Makefile.yp` 的旧做法。对该文件的更改会影响更改后初始化的所有域，但不影响更改前初始化的域，因此无论哪种方式都容易出错：你既冒着预期更改不生效的风险，也冒着忘记它们并让它们稍后影响从未打算受影响的其他域的风险。

创建源目录并用你需要的配置文件填充它。请参阅 <a href="https://man.openbsd.org/Makefile.yp">Makefile.yp(8)</a> 了解哪些 YP 映射需要哪些源文件。有关单个配置文件的格式，请参阅 <a href="https://man.openbsd.org/passwd.5">passwd(5)</a>、<a href="https://man.openbsd.org/group.5">group(5)</a>、<a href="https://man.openbsd.org/hosts">hosts(5)</a> 等，并查看 `/etc` 中的示例。

使用以下命令创建 YP 映射的初始版本：

```shell
# cd /var/yp
# make
```

现在不要担心 <a href="https://man.openbsd.org/yppush">yppush(8)</a> 的错误消息。YP 服务器尚未运行。

YP 使用 <a href="https://man.openbsd.org/rpc">rpc(3)</a>（远程过程调用）与客户端通信，因此必须启用 <a href="https://man.openbsd.org/portmap">portmap(8)</a>。为此，请使用 <a href="https://man.openbsd.org/rcctl">rcctl(8)</a>。

```shell
# rcctl enable portmap
# rcctl start portmap
```

考虑使用 YP 服务器守护进程的 <a href="https://man.openbsd.org/securenet">securenet(5)</a> 或 <a href="https://man.openbsd.org/ypserv.acl">ypserv.acl(5)</a> 安全功能。但请注意，这两者都仅提供基于 IP 的访问控制。因此，只有当潜在攻击者既没有对承载 YP 流量的网络段的硬件进行物理访问，也没有对连接到这些网络段的任何主机的 root 访问权限时，它们才有帮助。

最后，启动 YP 服务器守护进程：

```shell
# rcctl enable ypserv
# rcctl start ypserv
```

要测试新服务器，请考虑按照下一节第一部分的说明将其设为自己的客户端。如果你不希望服务器使用自己的映射，可以在测试后使用以下命令禁用客户端部分：

```shell
# rcctl stop ypbind
# rcctl disable ypbind
```

请记住，每次更改 YP 映射所源自的文件时，都必须重新生成 YP 映射。

```shell
# cd /var/yp
# make
```

这将更新 `/var/yp/`domainname`` 中的所有数据库文件，只有一个例外：文件 `ypservers.db`，列出了与该域关联的所有 YP 主服务器和从服务器，它直接由 `ypinit -m` 创建，并仅由 `ypinit -u` 修改。如果你不小心删除了它，请运行 `ypinit -u` 从头开始重新创建它。

<h3 id="YP_client">设置 YP 客户端</h3>

设置 YP 客户端涉及两个不同的部分。首先，你必须让 YP 客户端守护进程运行，将你的客户端主机绑定到 YP 服务器。完成以下步骤将允许你从 YP 服务器检索数据，但系统尚未使用该数据：

像在服务器上一样，你必须设置域名并启用 portmapper：

```shell
# echo "puffynet" > /etc/defaultdomain
# domainname `cat /etc/defaultdomain`
# rcctl enable portmap
# rcctl start portmap
```

建议在配置文件 `/etc/yp/`domainname`` 中提供 YP 服务器列表。否则，YP 客户端守护进程将使用网络广播来查找其域的 YP 服务器。显式指定服务器既更健壮，也稍微不那么容易受到攻击。如果你没有设置任何从服务器，只需将主服务器的主机名放入 `/etc/yp/`domainname`` 中。

启用并启动 YP 客户端守护进程 <a href="https://man.openbsd.org/ypbind">ypbind(8)</a>。

```shell
# rcctl enable ypbind
# rcctl start ypbind
```

如果一切顺利，你应该能够使用 <a href="https://man.openbsd.org/ypcat">ypcat(1)</a> 查询 YP 服务器并看到返回的 passwd 映射。

```shell
# ypcat passwd
bob:*:5001:5000:Bob Nuggets:/home/bob:/usr/local/bin/zsh
...
```

另一个用于调试 YP 设置的有用工具是 <a href="https://man.openbsd.org/ypmatch">ypmatch(1)</a>。

配置 YP 客户端的第二部分涉及编辑本地配置文件，以便各种系统设施使用某些 YP 映射。并非所有服务器都提供操作系统支持的所有标准映射，某些服务器提供额外的非标准映射，而且你绝不是被迫使用所有这些映射。应使用哪些可用映射，以及用于哪些目的，完全由客户端主机的系统管理员决定。

有关标准 YP 映射及其标准用法的列表，请参阅 <a href="https://man.openbsd.org/Makefile.yp">Makefile.yp(8)</a>。

如果你想包含 YP 域中的所有用户帐户，请将默认 YP 标记附加到主密码文件并重建密码数据库：

```shell
# echo '+:*::::::::' >> /etc/master.passwd
# pwd_mkdb -p /etc/master.passwd
```

有关选择性包含和排除用户帐户的详细信息，请参阅 <a href="https://man.openbsd.org/passwd.5">passwd(5)</a>。要测试包含是否实际有效，请使用 <a href="https://man.openbsd.org/id">id(1)</a> 实用程序。

如果你想包含 YP 域中的所有组，请将默认 YP 标记附加到组文件：

```shell
# echo '+:*::' >> /etc/group
```

有关选择性组包含的详细信息，请参阅 <a href="https://man.openbsd.org/group.5">group(5)</a>。