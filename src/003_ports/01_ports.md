## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">Ports - 使用 Ports</span>

---

- <a href="#PortsIntro">简介</a>
- <a href="#PortsFetch">获取 Ports 树</a>
- <a href="#PortsConfig">Ports 系统的配置</a>
- <a href="#PortsSearch">搜索 Ports 树</a>
- <a href="#PortsInstall">直接安装：一个简单的例子</a>
- <a href="#PortsClean">构建后的清理</a>
- <a href="#PortsDelete">卸载 Port 的软件包</a>
- <a href="#PortsFlavors">使用 Flavors 和子包</a>
- <a href="#dpb">使用 dpb 构建多个 Ports</a>
- <a href="#PortsSecurity">安全更新 (-stable)</a>
- <a href="#PkgSig">软件包签名</a>
- <a href="#Problems">报告问题</a>
- <a href="#Backtrace">调试包、调试器和回溯</a>
- <a href="#Helping">帮助我们</a>

---

<h2 id="PortsIntro">简介</h2>

Ports 树是为高级用户准备的。**鼓励大家使用预编译的二进制软件包。**
如果你对 ports 树有疑问，我们假设你已经阅读了手册页和本 FAQ，并且你有能力使用它。

Ports 树是一组 Makefiles，每个第三方应用程序一个，它控制：

- 在哪里以及如何获取软件的源代码
- 它依赖于哪些其他软件
- 如何修改源代码（如果需要）
- 如何配置和构建它
- 如何测试它（可选）
- 如何安装它

除了 Makefile 之外，每个 port 还至少包含以下内容：

- `PLIST` - 应用程序构建完成后用于创建软件包的指令
- `DESCR` - 应用程序的描述
- `distinfo` - 分发文件的校验和及大小

所有这些信息都保存在 `/usr/ports` 下的目录层次结构中。
此层次结构包含三个特殊的子目录：

- `distfiles/` - 下载后存储软件分发集的位置
- `infrastructure/` - ports 基础设施所需的所有脚本和 makefiles
- `packages/` - ports 系统构建的二进制软件包

其他子目录构成了不同的应用程序类别，其中包含实际 ports 的子目录。
复杂的 ports 可能会组织到更深的层次，例如，如果它们有一个核心部分和一组扩展，或者应用程序的稳定版和快照版。
每个 port 目录必须包含一个 `pkg/` 子目录，其中包含装箱单 (packing list) 和描述文件。
可能还有 `patches/` 和 `files/` 子目录，分别用于源代码补丁和其他文件。

当用户在特定 port 的子目录中发出 <a href="https://man.openbsd.org/make">make(1)</a> 命令时，系统将递归遍历其依赖树，检查是否安装了所需的依赖项，构建并安装任何缺失的依赖项，然后继续构建所需的 port。
所有的构建都发生在 port 创建的 *工作目录* 内。
通常这在 `${WRKOBJDIR}` 下，默认为 `/usr/ports/pobj`，但你可以覆盖它（参见 <a href="#PortsConfig">ports 系统的配置</a>）。

Ports 树与 OpenBSD 的<a href="../001_faq/05_building-from-source.md#Flavors">版本分支 (flavors)</a> 绑定。不要检出 -current ports 树并期望它在 release 或 -stable 系统上工作。**如果你跟随 -current，你需要同时拥有 -current 基础系统和 -current ports 树**。因为 -stable 中没有进行侵入性更改，所以可以在 OpenBSD release 上使用 -stable ports 树，反之亦然。

另一个常见的失败原因是缺少 X11 安装。即使你尝试构建的 port 没有直接依赖 X11，它的子包或其依赖项也可能需要 X11 头文件和库。**不支持在没有 X11 的系统上构建 ports**。

有关 ports 系统的更多信息可以在这些手册页中找到：

- <a href="https://man.openbsd.org/ports">ports(7)</a> - 描述 port 安装的不同阶段（make 目标）、flavors 和子包的使用以及其他一些选项。
- <a href="https://man.openbsd.org/bsd.port.mk">bsd.port.mk(5)</a> - 关于所有 make 目标、变量、伪（安装目录）框架等的深入信息。

<h2 id="PortsFetch">获取 Ports 树</h2>

一旦你决定了想要哪个版本的 ports 树，你可以从不同的来源获取它。下表概述了在哪里可以找到不同的版本，以及以何种形式。'&#x2713;' 标记可用，'&#x2717;' 表示无法通过该特定来源获得。

| 来源 | 形式 | release | -stable | snapshots | -current |
| :--- | :--- | :---: | :---: | :---: | :---: |
| <a href="https://www.openbsd.org/ftp.html">镜像站</a> | .tar.gz | &#x2713; | &#x2717; | &#x2713; | &#x2717; |
| <a href="https://www.openbsd.org/anoncvs.html">AnonCVS</a> | cvs checkout | &#x2713; | &#x2713; | &#x2717; | &#x2713; |

在镜像站上查找名为 `ports.tar.gz` 的文件。

```shell
$ cd /tmp
$ ftp https://cdn.openbsd.org/pub/OpenBSD/$(uname -r)/{ports.tar.gz,SHA256.sig}
$ signify -Cp /etc/signify/openbsd-$(uname -r | cut -c 1,3)-base.pub -x SHA256.sig ports.tar.gz
```

你想在 `/usr` 目录中解压此文件，这将创建 `/usr/ports` 及其下的所有目录。

```shell
# cd /usr
# tar xzf /tmp/ports.tar.gz
```

镜像站上提供的快照 (snapshots) 是每天从 -current ports 树生成的。你可以在 `/pub/OpenBSD/snapshots/` 目录中找到快照。如果你使用的是 ports 树的快照，你应该安装了匹配的 OpenBSD 快照。确保你的 ports 树和你的 OpenBSD 系统保持同步！

有关通过 CVS 获取 ports 树的更多信息，请阅读 <a href="https://www.openbsd.org/anoncvs.html">AnonCVS 页面</a>，其中包含可用服务器列表和许多示例。

<h2 id="PortsConfig">Ports 系统的配置</h2>

本节介绍构建 ports 的一些额外全局设置。你可以跳过它，但那样你就需要在后面的示例中以 root 身份执行许多 <a href="https://man.openbsd.org/make">make(1)</a> 语句。

因为 OpenBSD 项目没有资源来全面审查 ports 树中所有软件的源代码，你可以配置 ports 系统采取一些安全预防措施。Ports 基础设施能够以普通用户身份执行所有构建，并仅以 root 身份执行那些需要超级用户权限的步骤（例如 `install` make 目标）。有关如何配置权限分离的详细信息，请参阅 <a href="https://man.openbsd.org/bsd.port.mk#PORTS_PRIVSEP">bsd.port.mk(5)</a> 手册页。

通过分离在 port 构建期间写入的目录，可以使用只读 ports 树：

- ports 的工作目录。这由 `WRKOBJDIR` 变量控制，该变量指定将包含工作目录的目录。
- 包含分发文件的目录。这由 `DISTDIR` 变量控制。
- 包含新建二进制软件包的目录。这由 `PACKAGE_REPOSITORY` 变量控制。

例如，你可以将以下行添加到 `/etc/mk.conf`：

```text
WRKOBJDIR=/usr/obj/ports
DISTDIR=/usr/distfiles
PACKAGE_REPOSITORY=/usr/packages
```

如果需要，你还可以将这些目录的所有权更改为你的本地用户名和组，以便 ports 系统可以作为普通用户创建底层工作目录。同样，ports 可以作为用户*构建*，但必须由 root 或使用 <a href="https://man.openbsd.org/doas">doas(1)</a> *安装*。

<h2 id="PortsSearch">搜索 Ports 树</h2>

一旦你的系统上有了 ports 树和 portslist 软件包，搜索软件就变得非常容易。只需使用 `make search key="searchkey"`，如下例所示：

```shell
$ cd /usr/ports
$ doas pkg_add portslist
$ make search key=rsnapshot
Port:   rsnapshot-1.4.2
Path:   net/rsnapshot
Info:   remote filesystem snapshot utility
Maint:  Antoine Jacoutot <ajacoutot@openbsd.org>
Index:  net sysutils
L-deps:
B-deps: devel/autoconf/2.69 devel/automake/1.15 devel/metaauto net/rsync
R-deps: devel/p5-Lchown net/rsync
Archs:  any
```

搜索结果给出了找到的每个应用程序的概览：port 名称、port 路径、单行描述、port 维护者、与 port 相关的关键字、库/构建/运行时依赖项以及已知该 port 可工作的架构。

然而，这只是一个非常基本的机制，它只是在 ports 索引文件上运行 awk(1)。一个名为 "sqlports" 的 port 允许使用 SQL 进行非常细粒度的搜索。它是一个 SQLite 数据库，但几乎任何数据库格式都可以使用 ports 基础设施创建。`sqlports` port 包含了用于生成数据库的脚本，可以用作生成不同格式数据库的基础。

只需安装 `sqlports` 软件包即可开始使用。一个示例会话可能如下所示：

```shell
$ sqlite3 /usr/local/share/sqlports
sqlite> SELECT FULLPKGNAME,COMMENT FROM Ports WHERE COMMENT LIKE '%statistics%';
pg_statsinfo-3.3.0p0|monitor PostgreSQL activity & statistics
kactivities-stats-5.51.0|statistics for the KDE Activity concept
p5-Devel-Cover-Report-Clover-0.35|backend for Clover reporting of coverage statistics
mailgraph-1.14p1|RRDtool frontend for Postfix statistics
R-3.5.1p1|powerful math/statistics/graphics language
p5-Math-VecStat-0.08p0|provides basic statistics on numerical vectors
py-probstat-0.912p8|probability and statistics utilities for Python
py-statistics-1.0.3.5|port of Python 3.4 statistics module to Python 2
darkstat-3.0.719p1|network statistics gatherer with graphs
pfstat-2.5p2|packet filter statistics visualization
rtg-0.7.4p12|SNMP statistics monitoring system
slurm-0.4.3|network traffic monitor and statistics
tcpstat-1.5p0|report network interface statistics
libstatgrab-0.91p3|system statistics gathering library
p5-Unix-Statgrab-0.109p0|interface to libstatgrab system statistics library
diffstat-1.62|accumulates and displays statistics from a diff file
fragistics-1.7.0p2|Quake 3 statistics program
drupal7-google_analytics-1.2p2|add the Google Analytics web statistics to your site
dstat-0.5p0|dwm status bar statistics
kdf-4.14.3p4|KDE storage device statistics
libsysstat-0.4.1|library used to query system info and statistics
qdirstat-1.4p1|Qt-based directory statistics
sqlite>
```

以上仍然是非常基本的搜索。使用 SQL，几乎可以搜索任何内容，包括依赖项、配置标志、共享库等。

<h2 id="PortsInstall">直接安装：一个简单的例子</h2>

为了清楚起见，让我们考虑一个简单的例子：rsnapshot。此应用程序有几个构建依赖项：devel/autoconf/2.69 devel/automake/1.15 devel/metaauto net/rsync 以下命令假设你已按照 <a href="https://man.openbsd.org/bsd.port.mk#PORTS_PRIVSEP">bsd.port.mk</a> 手册中的详细说明配置了 PORTS_PRIVSEP。

```shell
$ cd /usr/ports/net/rsnapshot
$ make install
===>  Checking files for rsnapshot-1.4.2
>> Fetch https://github.com/rsnapshot/rsnapshot/archive/1.4.2/rsnapshot-1.4.2.tar.gz
100% |**************************************************|   365 KB    00:02
>> (SHA256) rsnapshot-1.4.2.tar.gz: OK
===> rsnapshot-1.4.2 depends on: metaauto-* - not found
===>  Verifying install for metaauto-* in devel/metaauto
===>  Checking files for metaauto-1.0p1
[...]
===>  Installing metaauto-1.0p1 from /usr/ports/packages/amd64/all/
metaauto-1.0p1: ok
===> Returning to build of rsnapshot-1.4.2
===> rsnapshot-1.4.2 depends on: metaauto-* -> metaauto-1.0p1
===> rsnapshot-1.4.2 depends on: autoconf-2.69 - not found
[...]
===> Returning to build of rsnapshot-1.4.2
===>  Extracting for rsnapshot-1.4.2
===>  Patching for rsnapshot-1.4.2
[...]
===>  Compiler link: clang -> /usr/bin/clang
===>  Compiler link: clang++ -> /usr/bin/clang++
===>  Compiler link: cc -> /usr/bin/cc
===>  Compiler link: c++ -> /usr/bin/c++
===>  Generating configure for rsnapshot-1.4.2
===>  Configuring for rsnapshot-1.4.2
[...]
===>  Building for rsnapshot-1.4.2
[...]
===>  Faking installation for rsnapshot-1.4.2
[...]
===>  Building package for rsnapshot-1.4.2
[...]
Link to /usr/ports/packages/amd64/all/rsnapshot-1.4.2.tgz
Link to /usr/ports/packages/amd64/ftp/rsnapshot-1.4.2.tgz
Link to /usr/ports/packages/amd64/cdrom/rsnapshot-1.4.2.tgz
===> rsnapshot-1.4.2 depends on: p5-Lchown-* - not found
[...]
===>  Installing rsnapshot-1.4.2 from /usr/ports/packages/amd64/all/
rsnapshot-1.4.2: ok
```

如你所见，ports 系统自动执行许多操作。它将获取、解压和修补源代码，配置和构建（编译）源代码，将文件安装到伪目录中，创建软件包（对应于装箱单）并将此软件包安装到你的系统上（通常在 `/usr/local/` 下）。并且它会递归地**为 port 的所有依赖项**执行此操作。注意上面的输出中的 "`===> Verifying install for ...`" 和 "`===> Returning to build of ...`" 行，表明了对依赖树的遍历。

如果你要安装的应用程序的旧版本已安装在你的系统上，你可以使用 `make update` 代替 `make install`。
这将调用带有 `-r` 标志的 <a href="https://man.openbsd.org/pkg_add">pkg_add(1)</a>。

大型应用程序将需要大量系统资源来构建。如果在构建此类 port 时遇到“内存不足”类型的错误，这通常有两个原因之一：

- 你的资源限制太严格。使用 ksh 的 `ulimit` 或 csh 的 `limit` 命令调整它们。
- 你根本没有足够的 RAM。

<h2 id="PortsClean">构建后的清理</h2>

你可能希望在构建软件包并安装后清理 port 的默认工作目录。

```shell
$ make clean
===>  Cleaning for rsnapshot-1.4.2
```

默认情况下，workdir 依赖项会自动清理。你可以使用以下命令移除仅用于构建的已安装软件包：

```shell
# pkg_delete -a
p5-Module-Build-0.4224: ok
autoconf-2.69p2:automake-1.15.1: ok
autoconf-2.69p2: ok
metaauto-1.0p1: ok
Read shared items: ok
```

如果你希望移除 port 的源分发集，你可以使用：

```shell
$ make clean=dist
===>  Cleaning for rsnapshot-1.4.2
===>  Dist cleaning for rsnapshot-1.4.2
```

如果你一直在编译同一 port 的多个 flavor，你可以使用以下命令一次性清除所有这些 flavor 的工作目录：

```shell
$ make clean=flavors
```

你还可以通过将 `BULK` 从 `auto` 更改为 `Yes` 来在每次构建后清理所有内容。

```shell
$ make package BULK=Yes
```

<h2 id="PortsDelete">卸载 Port 的软件包</h2>

卸载 port 非常容易：

```shell
$ make uninstall
===> Deinstalling for rsnapshot-1.4.2
rsnapshot-1.4.2: ok
Read shared items: ok
```

这将调用 <a href="https://man.openbsd.org/pkg_delete">pkg_delete(1)</a> 从你的系统中移除相应的软件包。

如果你想摆脱刚刚构建的软件包，你可以这样做：

```shell
$ make clean=packages
===>  Cleaning for rsnapshot-1.4.2
rm -f /usr/ports/packages/amd64/all/rsnapshot-1.4.2.tgz
```

<h2 id="PortsFlavors">使用 Flavors 和子包</h2>

请阅读 <a href="https://man.openbsd.org/ports">ports(7)</a> 手册页，它很好地概述了此主题。有两种机制可以根据不同需求控制软件的打包。

第一种机制称为 **flavors**（风味/变体）。Flavor 通常表示一组特定的编译选项。例如，某些应用程序具有 "no_x11" flavor，可用于没有 X 的系统。某些 shell 具有 "static" flavor，将构建静态链接版本。有些 ports 具有不同的 flavor，用于使用不同的图形工具包构建它们。其他示例包括支持不同的数据库格式、不同的网络选项（SSL、IPv6...）、不同的纸张尺寸等。

**总结**：当没有为你正在寻找的 flavor 提供软件包时，你最有可能使用 flavors。在这种情况下，你可以指定所需的 flavor 并自己构建 port。

大多数 port flavors 在编译期间都有自己的工作目录，并且每个 flavor 都将被打包成相应命名的软件包以避免混淆。要查看特定 port 的不同 flavors，你可以切换到其子目录并发出：

```shell
$ make show=FLAVORS
```

你还可以查看 port 的 `DESCR` 文件，其中更详细地解释了可用的 flavors。

第二种机制称为 **subpackages**（子包）。如果同一应用程序的不同部分可以在逻辑上分开，porter 可能会决定为其创建子包。你经常会看到程序的客户端部分和服务器部分的子包。有时，大量文档被捆绑在一个单独的子包中，因为它占用了大量磁盘空间。引入大量依赖项的额外功能通常会单独打包。Porter 还将决定哪个子包是主子包，默认安装。其他示例包括随软件附带的大量测试套件、支持不同事物的单独模块等。

**总结**：有些 ports 分为几个软件包。`make install` 只会安装主子包。

要列出 port 构建的不同软件包，请使用：

```shell
$ make show=PKGNAMES
```

`make install` 只会安装主子包。要全部安装，请使用：

```shell
$ make install-all
```

要列出 port 可用的不同子包，请使用：

```shell
$ make show=MULTI_PACKAGES
```

可以从 ports 树中选择要安装的子包。经过一些测试后，此过程将只调用 <a href="https://man.openbsd.org/pkg_add">pkg_add(1)</a> 来安装所需的子包。

```shell
# env SUBPACKAGE="-server" make install
```

子包机制仅处理软件包。它不会在构建 port 之前修改任何配置选项。你必须为此使用 flavors。

<h2 id="dpb">使用 dpb 构建多个 Ports</h2>

当你需要一次构建多个 port 时，可以使用 `/usr/ports/infrastructure/bin/dpb` 工具。

<a href="https://man.openbsd.org/dpb">dpb(1)</a> 接受要构建的 ports 列表，并以最佳顺序自动构建它们，尽可能利用并行性。它还可以使用多台机器进行构建，并生成详细的构建过程日志以进行故障排除，默认放置在 `/usr/ports/logs` 中。

```shell
# /usr/ports/infrastructure/bin/dpb -P ~/localports
```

此命令将读取 `~/localports` 中的 <a href="https://man.openbsd.org/pkgpath">pkgpaths</a> 列表并构建所有软件包。它还可以在构建后安装软件包。`~/localports` 文件可能如下所示：

```text
net/rsync
www/mozilla-firefox
editors/vim
```

如果你没有在命令行上或通过 `-P` 或 `-I` 提供要构建的 ports 列表，<a href="https://man.openbsd.org/dpb">dpb(1)</a> 将构建 ports 树中的所有 ports。如果以 root 身份运行，dpb 将自动降低权限为专用用户，以获取 distfiles 和构建 ports。这是推荐的使用方式，但它也可以作为普通用户运行。此外，可以使用 <a href="https://man.openbsd.org/proot">proot(1)</a> 实用程序进一步隔离构建操作。

<h2 id="PortsSecurity">安全更新 (-stable)</h2>

当在第三方软件中发现严重错误或安全漏洞时，它们会在 ports 树的 -stable 分支中修复。与基础系统相比，-stable ports 树仅获取最新版本的安全向后移植。

这意味着你只需要确保检出 ports 树的正确分支，并从中构建所需的软件。你可以使用 <a href="https://www.openbsd.org/anoncvs.html">CVS</a> 保持你的树最新，并订阅 ports-changes <a href="https://www.openbsd.org/mail.html">邮件列表</a>以接收与 ports 树中软件相关的安全公告。

<h2 id="PkgSig">软件包签名</h2>

签名是确保软件包合法且未损坏的好方法。OpenBSD 使用 <a href="https://man.openbsd.org/signify">signify(1)</a> 提供官方签名软件包。用户无需付出额外努力即可确保软件包未被篡改。

如果你想构建*你自己的*签名软件包，你首先需要创建用于签名的密钥。

```shell
# signify -Gns /etc/signify/my-pkg.sec -p /etc/signify/my-pkg.pub
```

注意名称：用于签署软件包的密钥应以 `pkg` 结尾。

构建后，可以使用 <a href="https://man.openbsd.org/pkg_sign">pkg_sign(1)</a> 对现有软件包进行签名。

```shell
# cd /usr/ports/packages/$(uname -p)
# pkg_sign -s signify2 -s /etc/signify/my-pkg.sec -o signed -S all
```

为了在另一台机器上安装该软件包，应将公钥 `my-pkg.pub` 放入该机器上的 `/etc/signify` 目录中。

<h2 id="Problems">报告问题</h2>

如果你在使用现有 port 时遇到问题，请向 port 维护者发送电子邮件。要查看 port 的维护者，请输入，例如：

```shell
$ cd /usr/ports/archivers/unzip
$ make show=MAINTAINER
```

或者，如果没有维护者，或者你无法联系到他们，请发送电子邮件至 <a href="mailto:ports@openbsd.org">ports@openbsd.org</a> 邮件列表。

在任何情况下，请提供：

- 你的 OpenBSD 版本，包括你可能已应用的任何补丁。内核版本由 `sysctl -n kern.version` 给出。
- 你的 ports 树版本：如果文件 `/usr/ports/CVS/Tag` 存在，请提供其内容。如果此文件不存在，则你使用的是 -current ports 树。
- 问题的完整描述。不要害怕提供细节。提及你在问题发生之前遵循的所有步骤。问题是否可重现？你提供的信息越多，获得帮助的可能性就越大。
- 如果应用程序崩溃，请尝试提供<a href="#Backtrace">回溯 (backtrace)</a>。

对于无法正确构建的 ports，几乎总是需要完整的构建记录。你可以使用位于 `/usr/ports/infrastructure/bin` 中的 `portslogger` 脚本来完成此操作。示例运行可能如下所示：

```shell
$ mkdir ~/portlog
$ cd /usr/ports/archivers/unzip
$ make 2>&1 | /usr/ports/infrastructure/bin/portslogger ~/portlog
```

之后，你应该在 `~/portlog` 目录中有一个构建日志文件，你可以将其发送给 port 维护者。另外，确保你在构建中没有使用任何特殊选项，例如在 `/etc/mk.conf` 中。

或者，你可以：

- 使用 <a href="https://man.openbsd.org/script">script(1)</a> 创建完整的构建记录。
- 如果看起来哪怕只有一点点相关，也请附上 <a href="https://man.openbsd.org/pkg_info">pkg_info(1)</a> 的输出。
- <a href="https://man.openbsd.org/gcc">gcc(1)</a> 内部编译器错误要求你向 gcc 邮件列表报告错误。如果你遵循他们的指示，并至少提供 `gcc -save-temps` 生成的各种文件，那将节省时间。

<h2 id="Backtrace">调试包、调试器和回溯</h2>

如果程序崩溃，获取回溯通常有助于指出问题所在。通常这是通过 GDB 完成的；基础操作系统中包含一个旧版本，但它通常不能很好地与当前的编译器配合使用。对于软件包调试，请从 ports 安装较新的版本，它提供 `egdb` 二进制文件：

```shell
# pkg_add gdb
```

对于某些机器架构上的某些 ports，可以使用调试包（例如，如果主 port 是 `neomutt-20201127`，则调试包将是 `debug-neomutt-20201127`）。这些包包含已分离到不同文件中的调试符号；GDB 知道如何自动加载它。调试包必须与主包匹配。如果你使用的是快照，你可能需要重新安装以确保它们来自同一构建。像往常一样使用 pkg_add 安装调试包：

```shell
# pkg_add debug-neomutt
```

如果没有可用的调试包，你可能能够从核心转储 (coredump) 中看到函数名称（但不是确切的行）；有时这足以指出问题。

可以通过使用 `make DEBUG=-g` 构建 port 来临时构建带有调试符号的 port（这在通常不构建调试包的架构上很有用）。你需要先清理现有的构建和软件包（有时除非先卸载，否则可能会遇到构建问题）。如果感兴趣的软件包是一个库或被其他 ports 依赖，你可能需要卸载整个链才能执行此操作。禁用优化（将 `-O0` 添加到 DEBUG）也可能很有用。

```shell
$ cd /usr/ports/math/moo
$ make clean; make DEBUG="-g" repackage; make reinstall
```

如果你有核心转储文件，请将 GDB 指向它并获取回溯：

```shell
$ egdb neomutt neomutt.core
GNU gdb (GDB) 7.12.1
Copyright (C) 2017 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-unknown-openbsd6.8".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from /usr/local/bin/neomutt...Reading symbols from /usr/local/bin/.debug/neomutt.dbg...done.
done.
[New process 577437]
Core was generated by `neomutt'.
Program terminated with signal SIGABRT, Aborted.
#0  _thread_sys_poll () at /tmp/-:3
3       /tmp/-: No such file or directory.
(gdb) bt
#0  _thread_sys_poll () at /tmp/-:3
<...>
#9  0x00000b3f1d5ec847 in main (argc=<optimized out>, argv=0x7f7ffffc8a08, envp=<optimized out>) at ../neomutt-20201127/main.c:1236
(gdb) q
```

对于 setuid/setgid 程序，默认情况下禁用核心转储，请参阅 <a href="https://man.openbsd.org/sysctl.8#EXAMPLES">sysctl(8)</a> 了解允许它们的方法。或者，你可以使用调试器加载程序，根据需要设置参数，然后 `run`：

```shell
$ egdb neomutt
<...>
(gdb) set args -d 2
(gdb) run
<...>
```

当收到某些信号时（在大多数情况下，在它崩溃的地方），你将返回到调试器。

要在典型 port 的正常批量构建中启用调试包，请将 "`DEBUG_PACKAGES=${BUILD_PACKAGES}`" 添加到 Makefile。这将在构建期间设置 DEBUG，并在打包阶段自动将调试符号与主二进制文件分离。它还将尝试使用 dwz 压缩符号；此步骤有时会失败；如果是这样，可以用 "`DWZ=:`" 覆盖它。如果你经常进行 ports 开发工作，在 /etc/mk.conf 中设置 "`INSTALL_DEBUG_PACKAGES=Yes`" 将自动安装新构建的调试包。

<h2 id="Helping">帮助我们</h2>

有很多方法可以提供帮助。下面按难度递增顺序列出。

- <a href="#Problems">报告问题</a> 当你遇到它们时。
- 你可以系统地测试 ports 并报告损坏情况，或建议改进。只需查看 <a href="./04_testing.md">Port 测试指南</a>。你**必须运行 -current 系统**来创建和测试 ports。
- 测试发布到 ports 邮件列表的 ports 更新。
- 向 port 维护者发送更新或补丁，如果 port 没有维护者，则发送到 ports 邮件列表。通常这非常受欢迎，除非你的补丁会导致开发人员浪费时间而不是节省时间。
- <a href="./02_guide.md">创建新 ports</a>。

<a href="https://www.openbsd.org/want.html">硬件</a>捐赠可以协助在 OpenBSD 运行的各种<a href="https://www.openbsd.org/plat.html">平台</a>上测试 ports。
