# [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">FAQ - 安装指南</span>

---

- <a href="#bsd.rd"      >安装过程概览</a>
- <a href="#Checklist"   >安装前检查清单</a>
- <a href="#Download"    >下载 OpenBSD</a>
- <a href="#MkInsMedia"  >制作安装介质</a>
- <a href="#Install"     >执行简单安装</a>
- <a href="#FilesNeeded" >文件集</a>
- <a href="#Partitioning">磁盘分区</a>
- <a href="#WifiOnly"    >引导无线固件</a>
- <a href="#SendDmesg"   >安装后发送 dmesg</a>
- <a href="#site"        >自定义安装过程</a>
- <a href="#Multibooting">多重引导</a>

---

<h2 id="bsd.rd">安装过程概览</h2>

OpenBSD 安装程序使用一个特殊的内存盘内核（`bsd.rd`），它会生成一个完全在内存中运行的实时环境。它包含安装脚本和执行完整安装所需的少量实用程序。这些实用程序也可用于灾难恢复。

内存盘内核可以从多种不同的来源引导：

- CD/DVD
- USB 驱动器
- 现有的分区
- 通过网络（<a href="./07_networking.md#PXE">PXE</a> 或其他<a href="https://man.openbsd.org/diskless">网络引导选项</a>）
- 软盘

并非每个<a href="https://www.openbsd.org/plat.html">平台</a>都支持所有这些选项。

如果你已经有一个运行中的 OpenBSD 系统，`bsd.rd` 就是你重新安装或升级到新版本所需的全部内容。要做到这一点，<a href="#Download">下载并验证</a>新的 `bsd.rd`，将其放在现有的文件系统上，并从中引导。引导 `bsd.rd` 的通用方法是通过你所在平台使用的任何手段，将引导内核从 `/bsd` 更改为 `/bsd.rd`。

在 amd64 系统上引导 `bsd.rd` 可以像这样操作：

```text
Using drive 0, partition 3.
Loading......
probing: pc0 com0 com1 mem[638K 1918M a20=on]
disk: hd0+ hd1+
>> OpenBSD/amd64 BOOT 3.33
boot> bsd.rd
```

这将从第一个识别到的硬盘的第一个分区引导名为 `bsd.rd` 的内核。

如果你需要指定不同的驱动器或分区，只需在内核名称前加上其位置前缀。下面的示例将从第二个硬盘的第四个分区引导：

```text
Using drive 0, partition 3.
Loading......
probing: pc0 com0 com1 mem[638K 1918M a20=on]
disk: hd0+ hd1+
>> OpenBSD/amd64 BOOT 3.33
boot> boot hd1d:/bsd.rd
```

OpenBSD 引导加载程序记录在特定架构的 <a href="https://man.openbsd.org/boot.8">boot(8)</a> 手册页中。

<h2 id="Checklist">安装前检查清单</h2>

在开始之前，你应该对最终想要的结果有个概念。以下是一些值得预先考虑的事情：

- 机器名称
- 已安装和可用的硬件：
    - 验证与你硬件的兼容性。你可能需要查阅特定平台的安装说明，特别是如果你使用的是非 x86 CPU 架构。它们包含详细的说明和任何可能的注意事项：
      <br>
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/alpha/INSTALL.alpha">alpha</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/amd64/INSTALL.amd64">amd64</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/arm64/INSTALL.arm64">arm64</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/armv7/INSTALL.armv7">armv7</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/hppa/INSTALL.hppa">hppa</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/i386/INSTALL.i386">i386</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/landisk/INSTALL.landisk">landisk</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/loongson/INSTALL.loongson">loongson</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/luna88k/INSTALL.luna88k">luna88k</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/macppc/INSTALL.macppc">macppc</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/octeon/INSTALL.octeon">octeon</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/powerpc64/INSTALL.powerpc64">powerpc64</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/riscv64/INSTALL.riscv64">riscv64</a>]
      [<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/sparc64/INSTALL.sparc64">sparc64</a>]
    - 如果无线网络是你唯一的网络选项，网卡是否需要额外的固件？如果是，请阅读关于<a href="#WifiOnly">引导无线固件</a>的章节。
- 要使用的安装方法
- 期望的磁盘布局：
    - 现有数据是否需要保存到别处？
    - OpenBSD 是否要与另一个操作系统共存于该系统上？如果是，如何引导每个系统？你需要安装引导管理器吗？
    - 整个磁盘都用于 OpenBSD，还是你想保留现有的分区/操作系统？（或者为未来的系统预留空间）
    - 你希望如何对磁盘的 OpenBSD 部分进行子分区？
    - 你需要磁盘加密吗？
- 网络设置（如果不使用 DHCP）：
    - 域名和 DNS 地址
    - 每个网卡的 IP 地址和子网掩码
    - 网关地址

<h2 id="Download">下载 OpenBSD</h2>

提供以下安装镜像：

| | |
| ----------- | ----------- |
| install78.img | 一个可以写入 USB 闪存盘或类似设备的磁盘镜像。包含[文件集（File Sets）](https://www.openbsd.org/faq/faq4.html#FilesNeeded)。<hr/>[amd64](https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.img) \| [arm64](https://cdn.openbsd.org/pub/OpenBSD/7.8/arm64/install78.img) \| [i386](https://cdn.openbsd.org/pub/OpenBSD/7.8/i386/install78.img) \| [octeon](https://cdn.openbsd.org/pub/OpenBSD/7.8/octeon/install78.img) \| [powerpc64](https://cdn.openbsd.org/pub/OpenBSD/7.8/powerpc64/install78.img) \| [riscv64](https://cdn.openbsd.org/pub/OpenBSD/7.8/riscv64/install78.img) \| [sparc64](https://cdn.openbsd.org/pub/OpenBSD/7.8/sparc64/install78.img)|
| miniroot78.img | 与上述相同，但不包含文件集。它们可以从互联网或本地磁盘获取。<hr/>[alpha](https://cdn.openbsd.org/pub/OpenBSD/7.8/alpha/miniroot78.img) \| [amd64](https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/miniroot78.img) \| [arm64](https://cdn.openbsd.org/pub/OpenBSD/7.8/arm64/miniroot78.img) \| [armv7](https://cdn.openbsd.org/pub/OpenBSD/7.8/armv7/) \| [i386](https://cdn.openbsd.org/pub/OpenBSD/7.8/i386/miniroot78.img) \| [landisk](https://cdn.openbsd.org/pub/OpenBSD/7.8/landisk/miniroot78.img) \| [luna88k](https://cdn.openbsd.org/pub/OpenBSD/7.8/luna88k/miniroot78.img) \| [octeon](https://cdn.openbsd.org/pub/OpenBSD/7.8/octeon/miniroot78.img) \| [powerpc64](https://cdn.openbsd.org/pub/OpenBSD/7.8/powerpc64/miniroot78.img) \| [riscv64](https://cdn.openbsd.org/pub/OpenBSD/7.8/riscv64/miniroot78.img) \| [sparc64](https://cdn.openbsd.org/pub/OpenBSD/7.8/sparc64/miniroot78.img)|
| install78.iso | 一个 ISO 9660 镜像，可用于制作安装 CD/DVD。包含文件集。<hr/>[alpha](https://cdn.openbsd.org/pub/OpenBSD/7.8/alpha/install78.iso) \| [amd64](https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.iso) \| [hppa](https://cdn.openbsd.org/pub/OpenBSD/7.8/hppa/install78.iso) \| [i386](https://cdn.openbsd.org/pub/OpenBSD/7.8/i386/install78.iso) \| [macppc](https://cdn.openbsd.org/pub/OpenBSD/7.8/macppc/install78.iso) \| [powerpc64](https://cdn.openbsd.org/pub/OpenBSD/7.8/powerpc64/install78.iso) \| [sparc64](https://cdn.openbsd.org/pub/OpenBSD/7.8/sparc64/install78.iso)|
| cd78.iso | 与上述相同，但不包含文件集。<hr/>[alpha](https://cdn.openbsd.org/pub/OpenBSD/7.8/alpha/cd78.iso) \| [amd64](https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/cd78.iso) \| [hppa](https://cdn.openbsd.org/pub/OpenBSD/7.8/hppa/cd78.iso) \| [i386](https://cdn.openbsd.org/pub/OpenBSD/7.8/i386/cd78.iso) \| [macppc](https://cdn.openbsd.org/pub/OpenBSD/7.8/macppc/cd78.iso) \| [sparc64](https://cdn.openbsd.org/pub/OpenBSD/7.8/sparc64/cd78.iso)|
| floppy78.img | 支持一些缺乏其他引导选项的旧机器。<hr/>[amd64](https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/floppy78.img) \| [i386](https://cdn.openbsd.org/pub/OpenBSD/7.8/i386/floppy78.img) \| [sparc64](https://cdn.openbsd.org/pub/OpenBSD/7.8/sparc64/floppy78.img)|

镜像也可以从许多备用的<a href="https://www.openbsd.org/ftp.html">镜像站点</a>下载。

在安装文件所在的目录中可以找到一个包含校验和的 `SHA256` 文件。你可以使用 <a href="https://man.openbsd.org/sha256">sha256(1)</a> 命令确认下载的文件在传输过程中没有损坏。

```shell
$ sha256 -C SHA256 miniroot*.img
(SHA256) miniroot78.img: OK
```

或者，如果你使用的是 GNU coreutils：

```shell
$ sha256sum -c --ignore-missing SHA256
miniroot78.img: OK
```

然而，这只检查了*意外*的损坏。你可以使用 <a href="https://man.openbsd.org/signify">signify(1)</a> 和 `SHA256.sig` 文件对下载的镜像进行加密验证。

```shell
$ signify -Cp /etc/signify/openbsd-78-base.pub -x SHA256.sig miniroot*.img
Signature Verified
miniroot78.img: OK
```

请注意，其他操作系统上的 signify 软件包可能不包含所需的<a href="https://ftp.openbsd.org/pub/OpenBSD/7.8/openbsd-78-base.pub">公钥</a>，或者它可能安装在其他位置。

`install78.iso` 和 `install78.img` 镜像不包含 `SHA256.sig` 文件，因此安装程序会抱怨它无法检查包含的文件集的签名：

```text
Directory does not contain SHA256.sig. Continue without verification? [no]
```

这是因为让安装程序验证它们是没有意义的。如果有人制作了一个恶意的安装镜像，他们肯定可以修改安装程序，让它说文件是合法的。如果镜像的签名事先已经验证过，那么在该提示下回答 "yes" 是安全的。

<h2 id="MkInsMedia">制作安装介质</h2>

<h3 id="MkFlash">闪存盘</h3>

通过连接目标设备并使用 <a href="https://man.openbsd.org/dd">dd(1)</a> 复制镜像，可以创建一个可引导的 USB 闪存盘。

使用 OpenBSD，假设设备被识别为 `sd6`：

```shell
# dd if=install*.img of=/dev/rsd6c bs=1M
```

注意使用的是 **raw I/O 设备**：是 `rsd6c` 而不是 `sd6c`。

其他平台上的细节会有所不同。如果你使用的是不同的操作系统，请务必选择适当的设备名称：例如 Linux 上的 `/dev/sdX` 或 macOS 上的 `/dev/rdiskX`。

<h3 id="MkCD-ROM">CD-ROM</h3>

你可以在 OpenBSD 上使用 <a href="https://man.openbsd.org/cdio">cdio(1)</a> 创建可引导的 CD-ROM。

```shell
# cdio tao cd*.iso
```

<h2 id="Install">执行简单安装</h2>

如果你需要关于从首选介质引导的说明，请查看你机器的相关<a href="https://www.openbsd.org/plat.html">平台页面</a>。

安装程序旨在以极少用户干预的情况下安装一个非常可用的默认配置。实际上，你通常只需按 `<Enter>` 键就能获得一个良好的 OpenBSD 安装，只有在输入 root 密码时才需要把手移到键盘的其他位置。

在显示 <a href="https://man.openbsd.org/dmesg">dmesg(8)</a> 后，你会看到第一个安装程序问题：

```text
...
root on rd0a swap on rd0b dump on rd0b
erase ^?, werase ^W, kill ^U, intr ^C, status ^T

Welcome to the OpenBSD/amd64 7.8 installation program.
(I)nstall, (U)pgrade, (A)utoinstall or (S)hell?
```

选择 `(I)nstall` 并按照说明操作。

<h2 id="FilesNeeded">文件集</h2>

完整的 OpenBSD 安装被分解为若干个文件集：

| 文件集 | 描述 |
| :--- | :--- |
| `bsd` | 内核 **(必需)** |
| `bsd.mp` | 多处理器内核（仅在某些平台上） |
| `bsd.rd` | <a href="#bsd.rd">内存盘内核</a> |
| `base78.tgz` | 基础系统 **(必需)** |
| `comp78.tgz` | 编译器集合、头文件和库 |
| `man78.tgz` | 手册页 |
| `game78.tgz` | 基于文本的游戏 |
| `xbase78.tgz` | X11 的基础库和实用程序（需要 `xshare78.tgz`） |
| `xfont78.tgz` | X11 使用的字体 |
| `xserv78.tgz` | X11 的 X 服务器 |
| `xshare78.tgz` | X11 的手册页、区域设置和包含文件 |

建议新用户安装所有文件集。

`xbase78.tgz` 中的一些库，如 freetype 或 fontconfig，可以被 X 之外的程序用于处理文本或图形。这些程序通常需要字体，要么来自 `xfont78.tgz`，要么来自字体软件包。为了简单起见，开发者决定不维护一个允许大多数非 X ports 运行的最小化 `xbase78.tgz` 集。

<h3>安装后添加文件集</h3>

如果你在安装时选择跳过某些文件集，后来可能会发现确实需要它们。只需从你的根文件系统引导 <a href="#bsd.rd">bsd.rd</a> 并选择 `(U)pgrade`。当你到达文件集列表时，选择你需要的那些。

<h2 id="Partitioning">磁盘分区</h2>

OpenBSD 可以安装在只有 512MB 的空间中，但使用这么小的设备是高级用户做的事。在你有一些经验之前，建议使用 8GB 或更多的磁盘空间。

与某些其他操作系统不同，OpenBSD 鼓励用户将磁盘分割成多个分区，而不是仅仅一个或两个大分区。这样做的一些原因是：

- 安全性：OpenBSD 的一些默认安全特性依赖于文件系统<a href="https://man.openbsd.org/mount#o">挂载选项</a>，如 `nosuid`、`nodev`、`noexec` 或 `wxallowed`。
- 稳定性：如果用户或行为不端的程序拥有写权限，可能会用垃圾数据填满文件系统。你的关键程序（希望它们运行在不同的文件系统上）不会因此中断。
- <a href="https://man.openbsd.org/fsck">fsck(8)</a>：你可以将从不或很少需要写入的分区大部分时间挂载为 `readonly`，这将消除在崩溃或断电后进行文件系统检查的需要。

安装程序将根据你的硬盘大小创建一个分区方案。虽然这对所有人来说并不都是完美的布局，但它提供了一个很好的起点，让你弄清楚自己需要什么。在决定自定义分区方案之前，请阅读关于 disklabel 默认值的<a href="https://man.openbsd.org/disklabel#AUTOMATIC_DISK_ALLOCATION">自动磁盘分配</a>和 <a href="https://man.openbsd.org/hier">hier(7)</a> 手册页。

- 由于一些<a href="./04_package.md">软件包</a>需要从 `wxallowed` 文件系统启动，建议拥有一个单独的 `/usr/local` 分区。
- 当你需要升级时，非常小的分区可能会变得麻烦。
- 一个 `/home` 分区会很不错。操作系统有新版本？保持你的 `/home` 分区不动，擦除并重新加载其他所有内容。
- 你可能还想创建一个 <a href="./06_disk-setup.md#altroot">altroot 分区</a>用于备份你的根文件系统。
- 暴露在互联网上的系统应该有一个单独的 `/var`，甚至可能有一个单独的 `/var/log`。
- 从源代码编译一些 <a href="../003_ports/00_index.md">ports</a> 可能会占用你的 `/usr` 和 `/tmp` 分区的大量空间。

<h2 id="WifiOnly">引导无线固件</h2>

由于许可原因，某些固件无法直接随 OpenBSD 分发。<a href="https://man.openbsd.org/fw_update">fw_update(8)</a> 工具会自动下载并安装任何缺失的固件，但这需要一个工作的互联网连接。

在某些硬件配置的情况下，例如没有以太网端口的笔记本电脑，用户必须手动下载并安装固件才能首次上网。这可以在安装前完成（通过将固件文件添加到安装介质），或者在从 CD 或磁盘安装操作系统后完成。

将固件文件添加到安装介质不会在安装过程中启用硬件。它们将被添加到目标磁盘，以便在首次引导进入安装好的系统后可以使用硬件。

首先使用 <a href="https://man.openbsd.org/dmesg">dmesg(8)</a> 找到无线适配器的<a href="https://man.openbsd.org/man/?query=wireless&apropos=1">接口名称</a>。

从现有的 OpenBSD 安装中，使用 <a href="https://man.openbsd.org/vnconfig">vnconfig(8)</a> 将安装镜像挂载为 vnode 磁盘，并使用 <a href="https://man.openbsd.org/fw_update">fw_update(8)</a> 将所需文件下载到其中。此示例使用 <a href="https://man.openbsd.org/iwm">iwm(4)</a> 卡的固件：

```shell
# vnconfig install78.img
vnd0
# mount /dev/vnd0a /mnt
# cd /mnt
# fw_update -Fv iwm
Get/Verify iwm-firmware-20240410.tgz ... done.
fw_update: download iwm
# cd /
# umount /mnt
# vnconfig -u vnd0
```

生成的文件随后可用于<a href="#MkInsMedia">创建</a>带有必要固件的可引导安装镜像。

如果你没有现有的可访问互联网的 OpenBSD 系统，请使用另一台计算机从 <a href="http://firmware.openbsd.org/firmware/">http://firmware.openbsd.org/firmware/</a> 下载适当的文件，并将其放在 OpenBSD 可读的 USB 驱动器上。然后，在 OpenBSD 机器上，<a href="https://man.openbsd.org/mount">挂载</a>该驱动器并使用 <a href="https://man.openbsd.org/fw_update">fw_update(8)</a> 从那里安装它。

<h2 id="SendDmesg">安装后发送 dmesg</h2>

安装成功后，查看 <a href="https://man.openbsd.org/dmesg">dmesg(8)</a> 命令的输出，看看是否有任何引人注目的内容。如果设备显示为 `not configured`，这意味着内核目前不支持它。通过发送 dmesg，将来可能会对此进行改进。

引用自 `/usr/src/etc/root/root.mail`：

```text
If you wish to ensure that OpenBSD runs better on your machines, please do us
a favor (after you have your mail system configured!) and type something like:

# (dmesg; sysctl hw.sensors) | \
   mail -s "Sony VAIO 505R laptop, apm works OK" dmesg@openbsd.org

so that we can see what kinds of configurations people are running.  As shown,
including a bit of information about your machine in the subject or the body
can help us even further.  We will use this information to improve device driver
support in future releases.  (Please do this using the supplied GENERIC kernel,
not for a custom compiled kernel, unless you're unable to boot the GENERIC
kernel.  If you have a multi-processor machine, dmesg results of both GENERIC.MP
and GENERIC kernels are appreciated.)  The device driver information we get from
this helps us fix existing drivers. Thank you!
```

或者，将你的 dmesg 输出保存到文本文件并发送其内容给我们：

```shell
$ (dmesg; sysctl hw.sensors) > ~/dmesg.txt
```

请将你的电子邮件客户端配置为使用纯文本。特别是，不要使用 HTML 格式或强制换行。将 dmesg 放入邮件正文中，而不是作为附件。

<h2 id="site">自定义安装过程</h2>

<h3><code>site78.tgz</code> 文件集</h3>

OpenBSD 安装和升级脚本允许选择一个用户创建的名为 `site78.tgz` 的集合。像官方<a href="#FilesNeeded">文件集</a>一样，这是一个以 `/` 为根的 <a href="https://man.openbsd.org/tar">tar(1)</a> 归档文件，并使用 `-xzphf` 选项解压。它最后安装，因此可用于补充和修改默认安装的文件。此外，可以使用名为 `site78-$(hostname -s).tgz` 的依赖于主机名的集合。

**注意：** 如果你打算通过 HTTP(s) 提供集合，请将 `site78.tgz` 放在你的源目录中，并将其包含在你的 `index.txt` 中。这样它将在安装时成为一个选项。

<h3><code>install.site</code> 和 <code>upgrade.site</code> 脚本</h3>

如果 `site78.tgz` 文件集包含可执行文件 `/install.site`，安装程序将基于新安装系统的根目录运行 <a href="https://man.openbsd.org/chroot">chroot(8)</a> 来执行它。同样，升级脚本运行 `/upgrade.site`。后者可以在重启进行升级之前放置在系统的根目录中。

示例用法：

- 设置系统时间。
- 在向外界暴露新系统之前，立即对其进行备份/归档。
- 在首次引导后运行一组任意命令。如果使用 install.site 将任何此类命令附加到 <a href="https://man.openbsd.org/rc.firsttime">rc.firsttime(8)</a> 文件，就会发生这种情况（附加到此文件是必要的，因为安装程序本身可能会写入此文件）。在引导时，`rc.firsttime` 执行一次然后被删除。

<h2 id="Multibooting">多重引导</h2>

多重引导是指在一台计算机上拥有多个操作系统，并有某种手段选择要引导哪个操作系统。在开始之前，你可能想熟悉一下 <a href="./06_disk-setup.md#BootAmd64">OpenBSD 引导过程</a>。关于 <a href="https://man.openbsd.org/fdisk">fdisk(8)</a> 的简要介绍在<a href="./06_disk-setup.md#fdisk">使用 OpenBSD 的 fdisk</a> 章节中。

如果你要将 OpenBSD 添加到现有系统，你可能需要在安装 OpenBSD 之前创建一些空闲空间。除了你现有系统的原生工具外，<a href="https://gparted.org/">gparted</a> 可能对删除或调整现有分区大小很有用。最好使用四个主 MBR 分区之一来引导 OpenBSD。扩展分区可能无法工作。

据报道 <a href="https://www.rodsbooks.com/refind/">rEFInd</a> 通常可以工作。<a href="https://www.gnu.org/software/grub/">GRUB</a> 据报道通常会失败。无论哪种情况，后果自负。

<h3>Windows</h3>

引导配置数据 (BCD) 存储允许通过 `bcdedit` 引导多个版本的 Windows。在<a href="https://technet.microsoft.com/en-us/library/cc721886%28WS.10%29.aspx">这篇文章</a>中可以找到很好的介绍。如果你想要 GUI 替代方案，你可以尝试 <a href="https://neosmart.net/EasyBCD/">EasyBCD</a>。

你需要一份 OpenBSD 安装的<a href="./06_disk-setup.md#BootAmd64">分区引导记录 (PBR)</a> 的副本。你可以使用类似以下的过程将其复制到文件：

```shell
# dd if=/dev/rsd0a of=openbsd.pbr bs=512 count=1
```

其中 `sd0a` 是你的引导设备，你需要将文件 `openbsd.pbr` 弄到你的 Windows 系统分区。

一旦 OpenBSD 的 PBR 被复制到 Windows 系统分区，你需要一个具有管理员权限的 Shell 来运行以下命令：

```cmd
C:\Windows\system32> bcdedit /create /d "OpenBSD/i386" /application bootsector
The entry {0154a872-3d41-11de-bd67-a7060316bbb1} was successfully created.
C:\Windows\system32> bcdedit /set {0154a872-3d41-11de-bd67-a7060316bbb1} device boot
The operation completed successfully.
C:\Windows\system32> bcdedit /set {0154a872-3d41-11de-bd67-a7060316bbb1} path \openbsd.pbr
The operation completed successfully.
C:\Windows\system32> bcdedit /set {0154a872-3d41-11de-bd67-a7060316bbb1} device partition=c:
The operation completed successfully.
C:\Windows\system32> bcdedit /displayorder {0154a872-3d41-11de-bd67-7060316bbb1} /addlast
The operation completed successfully.
```

请注意，OpenBSD 期望计算机的实时时钟设置为协调世界时 (UTC)。更多信息请参见<a href="./03_system.md#TimeZone">此章节</a>。
