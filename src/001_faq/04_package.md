# [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">FAQ - 包管理</span>

---

- <a href="#Intro">简介</a>
- <a href="#Mirror">选择镜像站</a>
- <a href="#PkgFind">查找软件包</a>
- <a href="#PkgInstall">安装软件包</a>
- <a href="#PkgUpdate">更新软件包</a>
- <a href="#PkgRemove">移除软件包</a>
- <a href="#PkgDup">在另一台机器上复制已安装的软件包</a>
- <a href="#PkgPartial">不完整的软件包安装或移除</a>

---

<h2 id="Intro">简介</h2>

在 OpenBSD 系统上，人们可能想要使用许多应用程序。为了使这些软件更易于安装和管理，它们被 *移植 (ported)* 到 OpenBSD 并打包。包管理系统的目的是跟踪哪些软件被安装，以便它们可以轻松地更新或移除。在几分钟内，就可以获取并安装大量软件包，并且所有内容都放在正确的位置。

Ports 集合**没有**经过与 OpenBSD 基础系统相同的彻底安全审计。虽然我们努力保持软件包的高质量，但我们只是没有足够的资源来确保同等水平的健壮性和安全性。

OpenBSD ports 团队认为软件包是他们移植工作的目标，而不是 ports 本身。通常，建议你使用软件包，而不是从 ports 构建应用程序。

可以借助几个实用程序轻松管理软件包：

- <a href="https://man.openbsd.org/pkg_add">pkg_add(1)</a> - 用于安装和更新软件包
- <a href="https://man.openbsd.org/pkg_check">pkg_check(8)</a> - 用于检查已安装软件包的一致性
- <a href="https://man.openbsd.org/pkg_delete">pkg_delete(1)</a> - 用于移除已安装的软件包
- <a href="https://man.openbsd.org/pkg_info">pkg_info(1)</a> - 用于显示有关软件包的信息

为了正常运行，应用程序 X 可能需要安装其他应用程序 Y 和 Z。应用程序 X 被称为依赖于这些其他应用程序，这就是为什么 Y 和 Z 被称为 X 的 *依赖项 (dependencies)*。反过来，Y 可能需要其他应用程序 P 和 Q，而 Z 可能需要应用程序 R 才能正常运行。这样，就形成了一个完整的依赖树。

软件包看起来像简单的 `.tgz` 压缩包。基本上它们就是这样，但有一个关键的区别：它们包含一些额外的 *打包信息*。此信息被 <a href="https://man.openbsd.org/pkg_add">pkg_add(1)</a> 用于多种目的：

- 各种检查：软件包是否已经安装，或者它是否与其他已安装的软件包或文件名冲突？
- 系统上尚未存在的依赖项会在继续安装软件包之前自动获取并安装。
- 有关软件包的信息记录在中央存储库中，默认位于 `/var/db/pkg`。这除其他外，将防止在删除软件包本身之前删除该软件包的依赖项。这有助于确保应用程序不会被粗心的用户意外破坏。

<h2 id="Mirror">选择镜像站</h2>

<a href="https://man.openbsd.org/pkg_add">pkg_add(1)</a> 会在两个地方查找软件包：<a href="https://man.openbsd.org/installurl">installurl(5)</a> 文件 (`/etc/installurl`) 或 `PKG_PATH` 环境变量。前者是首选方法，并且在新安装中默认配置。

如果需要使用多个镜像，`PKG_PATH` 允许通过冒号分隔的列表来实现：

```shell
# export PKG_PATH=scp://user@company-build-server/usr/ports/packages/%a/all:https://trusted-public-server/%m:installpath
```

虽然默认设置对大多数人来说应该工作良好，但在<a href="https://www.openbsd.org/ftp.html">镜像页面</a>上可以找到备用位置列表。

<h2 id="PkgFind">查找软件包</h2>

对于最常见的架构，有大量的预编译软件包集合可用。

要搜索任何给定的软件包名称，请使用 <a href="https://man.openbsd.org/pkg_info">pkg_info(1)</a> 的 `-aQ` 标志。

```shell
$ pkg_info -aQ unzip
lunzip-1.14p0
unzip-6.0p17
unzip-6.0p17-iconv
```

另一种查找所需内容的方法是使用 `pkg_locate` 命令，该命令可从 `pkglocatedb` 软件包中获得。

```shell
$ pkg_locate mutool
mupdf-1.24.9-js:textproc/mupdf,js:/usr/local/bin/mutool
mupdf-1.24.9-js:textproc/mupdf,js:/usr/local/man/man1/mutool.1
mupdf-1.24.9:textproc/mupdf:/usr/local/bin/mutool
mupdf-1.24.9:textproc/mupdf:/usr/local/man/man1/mutool.1
```

如果你正在寻找特定的文件名，它可以用来查找哪些软件包包含该文件。

你会注意到某些软件包有几种不同的变体。这些被称为 **flavors**。<a href="../003_ports/02_guide.md#PortsFlavors">ports FAQ</a> 详细解释了 flavors，但基本上意味着它们配置了不同的选项集。例如，一个软件包可能有可选的数据库支持，支持没有 X11 的系统等。有些软件包还分为 **subpackages**（子包），可以单独安装。

并非所有可能的软件包都一定在镜像服务器上可用。某些应用程序根本无法在所有架构上运行。由于许可原因，某些应用程序无法通过镜像分发。

<h2 id="PkgInstall">安装软件包</h2>

<a href="https://man.openbsd.org/pkg_add">pkg_add(1)</a> 实用程序用于安装软件包。如果一个软件包存在多个 flavor，系统会提示你选择要安装哪一个。

```shell
# pkg_add rsync
Ambiguous: choose package for rsync
a       0: <None>
        1: rsync-3.3.0p2
        2: rsync-3.3.0p2-minimal
Your choice:
```

在这里，如果你想要标准软件包，你会选择 **1**，如果你需要 iconv 支持，你会选择 **2**。你也可以直接在命令行上选择 flavor，使用 `pkg_add rsync--`（默认）或 `pkg_add rsync--iconv`（iconv flavor）。

可以在一行中指定多个软件包名称，然后它们都会与其依赖项一起一次性安装。你也可以指定软件包的绝对位置，无论是本地文件还是远程 URL。支持的 URL 前缀有 http、https、ftp 和 scp。

对于某些软件包，将给出有关应用程序配置或使用的重要附加信息。

```shell
# pkg_add jove
jove-4.16.0.73p1: ok
--- +jove-4.16.0.73p1 -------------------
See /usr/local/share/jove/README about changes to /etc/rc or
/etc/rc.local so that the system recovers jove files
on reboot after a system crash
```

此外，某些软件包在位于 `/usr/local/share/doc/pkg-readmes` 的文件中提供配置和其他信息。

为了你的安全，如果你安装了一个之前安装并移除过的软件包，已修改的配置文件将不会被覆盖。当你更新软件包时也是如此。

有时你可能会遇到类似以下示例的错误：

```shell
# pkg_add xv
xv-3.10ap4:jpeg-6bp3: ok
xv-3.10ap4:png-1.2.14p0: ok
xv-3.10ap4:tiff-3.8.2p0: ok
Can't install xv-3.10ap15 because of libraries
|library X11.16.1 not found
| not found anywhere
Direct dependencies for xv-3.10ap15 resolve to png-1.6.31 jasper-1.900.1p5 tiff-4.0.8p1 jpeg-1.5.1p0v0
Full dependency tree is png-1.6.31 tiff-4.0.8p1 jasper-1.900.1p5 jpeg-1.5.1p0v0
```

软件包中捆绑的打包信息包含有关该软件包期望安装的共享库的信息。如果找不到所需的库之一，则不会安装该软件包，因为它无论如何都无法运行。

有几件事需要检查：

- 你的系统可能不完整：你没有安装包含所需库的<a href="./02_installation.md#FilesNeeded">文件集</a>之一。
- 你的系统（或软件包）可能已过时：你拥有所需库的旧版本。确保基础系统和任何已安装的软件包都是最新的。
- 如果你运行的是 -current，基础系统和软件包快照可能稍微不同步。等待镜像赶上并重试。

<h2 id="PkgUpdate">更新软件包</h2>

已安装的软件包可以使用 <a href="https://man.openbsd.org/pkg_add">pkg_add(1)</a> 更新，如下所示：

```shell
# pkg_add -u
```

这将尝试更新所有已安装的软件包，包括它们的依赖项。

<h2 id="PkgRemove">移除软件包</h2>

要移除软件包，只需使用 <a href="https://man.openbsd.org/pkg_delete">pkg_delete(1)</a>：

```shell
# pkg_delete screen
```

同样，修改后的配置文件将不会被移除。

之后可以使用 `-a` 标志移除不再需要的依赖项：

```shell
# pkg_delete -a
```

<h2 id="PkgDup">在另一台机器上复制已安装的软件包</h2>

安装一个新的 OpenBSD 系统，使其拥有与旧机器相同的软件包集是一个相当常见的使用案例。<a href="https://man.openbsd.org/pkg_info">pkg_info(1)</a> 的 `-mz` 标志将产生适当的结果，使此任务更容易。

- `-m` 标志仅选择手动安装的软件包。依赖项不会被记录，因为它们会自动被拉入。
- `-z` 标志从输出中排除版本信息。仅显示 flavor 和分支，确保未来的软件包安装将选择适当的版本。

例如：

```shell
$ pkg_info -mz | tee list
abcde--
mpv--
python--%3.6
vim--no_x11
```

将 "list" 文件复制到另一台机器并运行：

```shell
# pkg_add -l list
```

每个软件包规范都有一个附加到其名称后的 flavor（`--` 是默认值），并且共存多个版本的软件包也有分支信息。在这种情况下，后续的 <a href="https://man.openbsd.org/pkg_add">pkg_add(1)</a> 命令将选择 `3.6` 版本分支的当前 python 软件包。

<h2 id="PkgPartial">不完整的软件包安装或移除</h2>

在某些奇怪的情况下，你可能会发现由于与其他文件冲突，软件包未完全添加。不完整的安装通常在软件包名称前加上 "partial-" 标记。例如，当你恰好在安装过程中按 CTRL+C 时，就会发生这种情况。安装可以稍后完成，"partial-*" 软件包将消失，或者可以使用 <a href="https://man.openbsd.org/pkg_delete">pkg_delete(8)</a> 将其移除。

更严重的系统故障，如文件系统问题，可能导致 `/var/db/pkg` 损坏或不一致。

<a href="https://man.openbsd.org/pkg_check">pkg_check(8)</a> 实用程序可以帮助清理这些问题。
