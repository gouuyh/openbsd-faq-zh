# [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">FAQ - 从源代码构建系统</span>

---

- <a href="#Flavors">OpenBSD 的版本分支 (Flavors)</a>
- <a href="#Snapshots">开发快照 (Snapshots)</a>
- <a href="#Bld">从源代码构建 OpenBSD</a>
- <a href="#Release">制作发布版 (Release)</a>
- <a href="#Xbld">构建 X</a>
- <a href="#buildprobs">编译时的常见问题</a>
- <a href="#Miscellanea">杂项问题和提示</a>
- <a href="#Custom">自定义内核</a>
- <a href="#Diff">准备 Diff 文件</a>

---

<h2 id="Flavors">OpenBSD 的版本分支 (Flavors)</h2>

OpenBSD 有三个版本分支：

- **releases:** 每六个月发布一次的 OpenBSD 版本。
- **-current:** 开发分支。每六个月，-current 分支会被标记并成为下一个 release 版本。
- **-stable:** 一个 release 版本加上在<a href="https://www.openbsd.org/errata.html">勘误表页面</a>上找到的补丁。当 -current 中有非常重要的修复时，它们会被向后移植到受支持的 -stable 分支。

只有最近的两个 OpenBSD release 版本会收到基础系统的<a href="./03_system.md#Patches">安全和可靠性修复</a>。

新用户应该运行 -stable 或 release 版本。话虽如此，许多人确实在生产系统上运行 -current，以帮助捕捉 Bug 并测试新功能。

<h2 id="Snapshots">开发快照 (Snapshots)</h2>

在 OpenBSD 的正式发布之间，会提供 -current 分支的快照。这些是当时源代码树中任何代码的构建版本。

最近的快照通常是你运行 -current 所需的全部内容。如果你希望从源代码构建它，则必须从最新的快照开始。请查看<a href="../200_current.md">跟随 -current 并使用快照</a>页面，了解从源代码构建所需的任何配置更改或额外步骤。

你可能会在快照中发现 Bug。这是构建和分发它们的原因之一。如果你发现了 Bug，请确保<a href="https://www.openbsd.org/report.html">报告</a>它。

<h2 id="Bld">从源代码构建 OpenBSD</h2>

从源代码构建 OpenBSD 涉及多个步骤：

- 升级到最接近的可用二进制文件
- 获取源代码
- 构建新的内核和用户空间，如 <a href="https://man.openbsd.org/release">release(8)</a> 的步骤 2 和 3 所述
- 可选地，<a href="#Release">制作发布版</a>和<a href="#Xbld">构建 X</a>

本 FAQ 章节旨在帮助你做好必要的准备。主要参考资料是 <a href="https://man.openbsd.org/release">release(8)</a>。

<h3 id="BldBinary">升级到最接近的可用二进制文件</h3>

不要试图通过从源代码编译来从一个 release 版本升级到另一个。

确保你安装了最接近的可用二进制文件。如果你想构建 OpenBSD x.y-stable，这就是 OpenBSD x.y；如果你想构建 -current，这就是最新的快照。

<h3 id="BldGetSrc">获取源代码</h3>

OpenBSD 使用 <a href="https://savannah.nongnu.org/projects/cvs">CVS</a> 版本控制系统来管理其源代码。<a href="https://man.openbsd.org/cvs">cvs(1)</a> 程序用于将所需源代码的副本拉取到你的本地机器进行编译。<a href="https://www.openbsd.org/anoncvs.html">匿名 CVS</a> 页面上有 <a href="https://man.openbsd.org/cvs">cvs(1)</a> 的介绍和获取源代码树的详细说明。首先，你必须 `cvs checkout` 源代码树。之后，你通过运行 `cvs update` 将更新的文件拉取到你的本地树来维护它。你也可以使用 `reposync` 程序维护本地 CVS 仓库，该程序可作为 <a href="./04_package.md">软件包</a> 获取。<a href="https://www.openbsd.org/anoncvs.html#rsync">匿名 CVS</a> 页面上也解释了如何设置仓库镜像。

<h4 id="wsrc">避免使用 Root 权限</h4>

避免以 root 身份运行 <a href="https://man.openbsd.org/cvs">cvs(1)</a>。`/usr/src` 目录（你的源代码通常会放在这里）默认对 `wsrc` 组可写，因此将需要使用 <a href="https://man.openbsd.org/cvs">cvs(1)</a> 的用户添加到该组。

```shell
# user mod -G wsrc exampleuser
```

此更改将在 exampleuser 下次登录时生效。

如果你想以此用户身份获取 xenocara 或 ports，必须手动创建目录并设置其权限。

```shell
# cd /usr
# mkdir -p   xenocara ports
# chgrp wsrc xenocara ports
# chmod 775  xenocara ports
```

<h4>获取 -stable</h4>

要获取 -stable `src` 树，请使用 `-r` 标志指定你想要的分支：

```shell
$ cd /usr
$ cvs -qd anoncvs@anoncvs.example.org:/cvs checkout -rOPENBSD_7_8 -P src
```

一旦你检出了代码树，以后可以使用以下命令更新它：

```shell
$ cd /usr/src
$ cvs -q up -Pd -rOPENBSD_7_8
```

根据需要将 `src` 替换为 `xenocara` 或 `ports`。由于 OpenBSD 的所有部分必须保持同步，你使用的所有树都应该同时检出和更新。

<h4>获取 -current</h4>

要获取 -current src 树，你可以使用以下命令：

```shell
$ cd /usr
$ cvs -qd anoncvs@anoncvs.example.org:/cvs checkout -P src
```

使用以下命令更新树：

```shell
$ cd /usr/src
$ cvs -q up -Pd -A
```

将 `src` 替换为你想要的模块，如 `xenocara` 或 `ports`。

<h3>构建 OpenBSD</h3>

此时你已准备好从源代码构建 OpenBSD。

如果你正在构建 -current，请查看<a href="../200_current.md">此页面</a>上列出的更改和特殊构建说明。

按照 <a href="https://man.openbsd.org/release">release(8)</a> 步骤 2 和 3 中的详细说明进行操作。

<h3>关于构建过程的延伸阅读</h3>

- <a href="https://man.openbsd.org/mk.conf">mk.conf(5)</a>
- <a href="https://cvsweb.openbsd.org/src/Makefile">`src/Makefile`</a>
- <a href="https://cvsweb.openbsd.org/src/share/mk/bsd.README">`/usr/share/mk/bsd.README`</a>
- <a href="https://man.openbsd.org/config">config(8)</a>
- <a href="https://man.openbsd.org/options">options(4)</a>

<h2 id="Release">制作发布版 (Release)</h2>

发布版是可用于在另一台系统上安装或升级 OpenBSD 的完整文件集。一个使用示例是在一台快速机器上构建 -stable，然后制作一个发布版安装到你所有的其他机器上。如果你只有一台运行 OpenBSD 的计算机，你真的没有任何理由制作发布版，因为上述构建过程将完成你需要的一切。

制作发布版的说明在 <a href="https://man.openbsd.org/release">release(8)</a> 中。发布过程使用上述构建过程中在 `/usr/obj` 目录中创建的二进制文件。

注意：如果你希望通过 HTTP(s) 分发生成的文件集以供升级或安装脚本使用，你需要添加一个 `index.txt` 文件，其中包含你新创建的发布版中所有文件的列表。

```shell
# ls -nT > index.txt
```

如果你想对你创建的文件集进行加密签名，<a href="https://man.openbsd.org/signify">signify(1)</a> 手册页有关于如何操作的详细信息。

<h3>设置你的系统</h3>

制作发布版需要一个 `noperm` 分区。这允许构建基础设施在大部分过程中使用非特权 `build` 用户。

在 `/dest` 上创建一个设置了 `noperm` <a href="https://man.openbsd.org/mount">mount(8)</a> 选项的文件系统。相应的 <a href="https://man.openbsd.org/fstab">fstab(5)</a> 行可能如下所示：

```text
c73d2198f83ef845.m /dest ffs rw,nosuid,noperm 1 2
```

此文件系统的根目录必须由 `build` 拥有，权限为 `700`：

```shell
# chown build /dest
# chmod 700   /dest
```

为 base 和 xenocara 创建 `DESTDIR` 目录：

```shell
# mkdir /dest/{,x}base
```

你的 `RELEASEDIR` 不需要位于 `noperm` 文件系统上。确保它由 `build` 拥有，并且至少具有 `u=rwx` 权限。

<h3>使用 mfs noperm 分区</h3>

你可能想使用 <a href="https://man.openbsd.org/mount_mfs">mfs</a> 分区而不是物理磁盘。在你的 `/etc/fstab` 中添加类似这样的一行：

```text
swap /dest mfs rw,nosuid,noperm,-P/var/dest,-s1.5G,noauto 0 0
```

创建原型 `DESTDIR` 目录：

```shell
# mkdir -p /var/dest/{,x}base
# chown -R build /var/dest
# chmod -R 700   /var/dest
```

现在你可以在制作发布版之前挂载 `/dest`：

```shell
# mount /dest
```

<h2 id="Xbld">构建 X</h2>

从 <a href="https://www.x.org/wiki/">X.Org</a> v7 开始，X 切换到了模块化构建系统，将 X.Org 源代码树拆分为三百多个或多或少独立的软件包。

为了简化 OpenBSD 用户的生活，开发了一个名为 <a href="https://xenocara.org">Xenocara</a> 的元构建 (meta-build)。该系统将 X 转换回一个大树，以便在一个过程中构建。作为额外的奖励，这个构建过程比以前的版本更类似于 OpenBSD 其余部分使用的构建过程。

构建 X 的官方说明存在于 <a href="https://cvsweb.openbsd.org/xenocara/README">`xenocara/README`</a> 文件和 <a href="https://man.openbsd.org/release">release(8)</a> 的步骤 5 中。

<h2 id="buildprobs">编译时的常见问题</h2>

大多数时候，构建过程中的问题是由于没有仔细遵循说明造成的。从最新的快照构建 -current 偶尔会有真正的问题，但在构建 release 或 -stable 时的失败几乎总是用户错误。

大多数问题通常是以下之一：

- 未能从<a href="#BldBinary">适当的二进制文件</a>开始。
- <a href="#BldGetSrc">检出</a>了错误的代码树分支。
- 在构建 -current 之前忘记阅读<a href="../200_current.md">跟随 -current</a>。
- 在构建和安装新内核后跳过了重启步骤。

<h3 id="ProbObj">我在 <code>make build</code> 之前忘记了 <code>make obj</code></h3>

通过在执行 `make obj` 之前执行 `make build`，你的对象文件将散落在 `/usr/src` 目录中。这是一件坏事。如果你想尝试避免重新获取整个 src 树，可以尝试以下方法来清理 obj 文件：

```shell
$ cd /usr/src
$ find . -type l -name obj -delete
$ make cleandir
$ rm -rf /usr/obj/*
$ make obj
```

<h3 id="sig11">构建停止并出现 "Signal 11" 错误</h3>

从源代码构建 OpenBSD 和其他程序是一项比大多数其他任务更重地推硬件的任务，大量使用 CPU、磁盘和内存。Signal 11 故障 *通常* 是由硬件问题引起的。

你可能会发现最好修理或更换引起麻烦的组件，因为问题将来可能会以其他方式表现出来。

有关更多信息，请参阅 <a href="https://www.bitwizard.nl/sig11/">Sig11 FAQ</a>。

<h2 id="Miscellanea">杂项问题和提示</h2>

<h3>将你的用户添加到 <code>wobj</code> 组</h3>

如果你打算在源代码树中编译单个程序——例如，进行开发——你会想将你的用户添加到 `wobj` 组。这将允许你写入 `/usr/obj`。

<h3>Tags 文件</h3>

作为开发者的编辑器，<a href="https://man.openbsd.org/mg">mg(1)</a> 和 <a href="https://man.openbsd.org/vi">vi(1)</a> 内置了对 <a href="https://man.openbsd.org/ctags">ctags(1)</a> 文件的支持，这允许你快速导航源代码树。

在大多数程序或库的源代码目录中，你可以通过运行以下命令创建 `./tags` 文件：

```shell
$ make tags
```

在构建和安装 `libc` 时，也会创建一个 `/var/db/libc.tags` 文件。

默认情况下，每个架构的内核 tags 位于 `/sys/arch/$(machine)/`。这些文件可以从 `/sys/kern` 运行 `make tags` 创建。你可能想以 root 身份运行 `make links`，以便在每个目录和 `/var/db/` 中放置一个指向你架构的内核 `tags` 文件的符号链接。

<h3>我如何跳过构建树的部分内容？</h3>

使用 <a href="https://man.openbsd.org/mk.conf">mk.conf(5)</a> 的 `SKIPDIR` 选项。

<h3>我可以交叉编译吗？</h3>

系统中包含交叉编译工具，供开发者在启动新平台时使用。但是，它们不作为一般用途进行维护。

当开发者为新平台提供支持时，第一个大测试就是原生构建。从源代码构建系统会对操作系统和机器施加相当大的负载，并且能很好地测试系统的实际工作情况。出于这个原因，OpenBSD 在构建所使用的平台上进行所有构建过程。

<h2 id="Custom">自定义内核</h2>

有三种自定义内核的方法：

- 使用 <a href="https://man.openbsd.org/boot_config">boot_config(8)</a> 进行临时引导时配置
- 使用 <a href="https://man.openbsd.org/config">config(8)</a> 永久修改已编译的内核
- 编译自定义内核

<h3 id="BootConfig">引导时配置</h3>

OpenBSD 的引导时内核配置 <a href="https://man.openbsd.org/boot_config">boot_config(8)</a> 允许管理员修改某些内核设置，例如启用或禁用对各种设备的支持，而无需重新编译内核本身。

要引导进入用户内核配置 (User Kernel Config, UKC)，请在启动时使用 `-c` 选项：

```text
Using drive 0, partition 3.
Loading......
probing: pc0 com0 com1 mem[638K 1918M a20=on]
disk: hd0+ hd1+
>> OpenBSD/amd64 BOOT 3.33
boot> boot hd0a:/bsd -c
```

这样做将调出 UKC 提示符。输入 `help` 查看可用命令列表。

使用 <a href="https://man.openbsd.org/boot_config">boot_config(8)</a> 仅提供临时更改，这意味着必须在每次重启时重复该过程。下一节将解释如何使更改永久生效。

<h3 id="config">使用 config(8) 更改内核选项</h3>

使用 `-e` 标志调用 <a href="https://man.openbsd.org/config">config(8)</a> 允许你在运行的系统上进入 UKC。所做的任何更改将在下次重启时生效。`-u` 标志测试在引导期间是否对运行的内核进行了任何更改，这意味着你在引导系统时使用了 `boot -c` 进入 UKC。

为了避免用损坏的内核覆盖工作内核的风险，请考虑使用 `-o` 标志将更改写入单独的内核文件进行测试：

```shell
# config -e -o /bsd.new /bsd
```

这将把你的更改写入 `/bsd.new` 文件。一旦你从这个新内核引导并验证一切正常，可以通过将更改放入 <a href="https://man.openbsd.org/bsd.re-config">bsd.re-config(5)</a> 来使所需的更改永久生效。这样做消除了在启动时选择内核的需要，并确保休眠和内核重新链接保持工作。

<a href="https://man.openbsd.org/config">config(8)</a> 手册页中给出了内核修改示例。

<h3 id="Options">构建自定义内核</h3>

OpenBSD 团队仅支持 `GENERIC` 和 `GENERIC.MP` 内核。`GENERIC` 内核配置是 `/sys/arch/$(machine)/conf/GENERIC` 和 `/sys/conf/GENERIC` 中选项的组合。报告自定义内核上的问题几乎总是会导致你被告知尝试用 `GENERIC` 内核重现该问题。

首先阅读 <a href="https://man.openbsd.org/config">config(8)</a> 和 <a href="https://man.openbsd.org/options">options(4)</a> 手册页。以下步骤是编译自定义内核的一部分：

```shell
$ cd /sys/arch/$(machine)/conf
$ cp GENERIC CUSTOM
$ vi CUSTOM    # 进行你的更改
$ config CUSTOM
$ cd ../compile/CUSTOM
$ make
```

<h2 id="Diff">准备 Diff 文件</h2>

如果你对源代码进行了更改并希望与开发者分享，请遵循这些约定：

- 将你的 diff 基于 -current 检出版本。
- 使用 `cvs diff -uNp` 生成 diff。
- 将你的 diff 内联在电子邮件中发送到 **tech** <a href="https://www.openbsd.org/mail.html">邮件列表</a>。确保你的电子邮件客户端不会破坏空白字符。

如果你使用的是 OpenBSD 源代码树的 <a href="https://github.com/openbsd">git 镜像</a>，请设置

```shell
$ git config diff.noprefix true
```

在你的仓库中，并像这样生成你的 diff：

```shell
$ git diff --relative .
```

