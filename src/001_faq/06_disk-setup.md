# [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">FAQ - 磁盘配置</span>

---

- <a href="#intro">磁盘和分区</a>
- <a href="#fdisk">使用 fdisk</a>
- <a href="#disklabel">磁盘标签 (Disk Labels)</a>
- <a href="#BootAmd64">amd64 引导过程</a>
- <a href="#altroot">Root 分区备份 (/altroot)</a>
- <a href="#DupFS">复制文件系统</a>
- <a href="#Quotas">磁盘配额</a>
- <a href="#foreignfs">访问其他文件系统</a>
- <a href="#MountImage">挂载磁盘镜像</a>
- <a href="#GrowPartition">扩容磁盘分区</a>
- <a href="#softraid">RAID 和磁盘加密</a>
    - <a href="#softraidDI">安装到镜像</a>
    - <a href="#softraidFDE">全盘加密</a>
    - <a href="#softraidCrypto">加密外部磁盘</a>

---

<h2 id="intro">磁盘和分区</h2>

在 OpenBSD 中设置磁盘的细节因<a href="https://www.openbsd.org/plat.html">平台</a>而异，因此你应该阅读你平台的 `INSTALL.<arch>` 文件中的说明。

<h3>驱动器标识</h3>

在大多数平台上，OpenBSD 使用两个驱动程序处理大容量存储：

- <a href="https://man.openbsd.org/wd">wd(4)</a>: 类 IDE 磁盘：IDE、SATA、MFM 或 ESDI 磁盘，或连接到 <a href="https://man.openbsd.org/wdc">wdc(4)</a> 或 <a href="https://man.openbsd.org/pciide">pciide(4)</a> 接口的闪存设备。
- <a href="https://man.openbsd.org/sd">sd(4)</a>: 类 SCSI 磁盘：使用 SCSI 命令的设备、USB 磁盘、连接到 <a href="https://man.openbsd.org/ahci">ahci(4)</a> 接口的 SATA 磁盘，以及连接到 RAID 控制器的磁盘阵列。

设备按引导时检测到的顺序编号，从零开始。因此，第一个类 IDE 磁盘将是 `wd0`，第三个类 SCSI 磁盘将是 `sd2`。请注意，OpenBSD 不一定按与引导 ROM 相同的顺序对驱动器进行编号。

<h3>分区和文件系统</h3>

术语“分区”在 OpenBSD 中可能意味着两件不同的事情：

- <a href="https://man.openbsd.org/disklabel">disklabel(8)</a> 分区，也称为文件系统分区。
- <a href="https://man.openbsd.org/fdisk">fdisk(8)</a> 分区，有时称为主引导记录 (MBR) 分区。

所有 OpenBSD 平台都使用 disklabel 程序作为管理文件系统分区的主要方式。在使用 fdisk 的平台上，一个 MBR 分区用于容纳所有 OpenBSD 文件系统。此分区可以切分为 16 个 disklabel 分区，标记为 `a` 到 `p`。一些标签是特殊的：

- `a`: 引导磁盘的 `a` 分区是你的根分区。
- `b`: 引导磁盘的 `b` 分区通常是交换分区。
- `c`: `c` 分区始终是整个磁盘。

要在 disklabel 分区上创建新文件系统，请使用 <a href="https://man.openbsd.org/newfs">newfs(8)</a> 命令：

```shell
# newfs sd2a
```

因此，设备名称加上 disklabel 标识一个 OpenBSD 文件系统。例如，标识符 `sd2a` 指的是第三个 `sd` 设备的 `a` 分区上的文件系统。其设备文件将是用于块设备的 `/dev/sd2a` 和用于原始（字符）设备的 `/dev/rsd2a`。记住一个很少使用的命令需要块设备还是字符设备是很困难的。因此，许多命令使用 <a href="https://man.openbsd.org/opendev">opendev(3)</a> 函数，该函数会自动将 `sd0` 扩展为 `/dev/rsd0c` 或 `/dev/sd0c`（视情况而定）。

<h3 id="DUID">磁盘标签唯一标识符 (DUID)</h3>

默认情况下，磁盘在 <a href="https://man.openbsd.org/fstab">fstab(5)</a> 文件中通过磁盘标签唯一标识符 (DUID) 标识。DUID 是首次创建 disklabel 时生成的 16 位十六进制随机数。它们由 <a href="https://man.openbsd.org/diskmap">diskmap(4)</a> 设备管理。要显示所有磁盘的 DUID，请执行：

```shell
$ sysctl hw.disknames
hw.disknames=wd0:bfb4775bb8397569,cd0:,wd1:56845c8da732ee7b,wd2:f18e359c8fa2522b
```

你可以通过附加句点和分区字母来指定磁盘上的分区。例如，`f18e359c8fa2522b.d` 是磁盘 `f18e359c8fa2522b` 的 `d` 分区，并且无论连接到系统的设备顺序如何，或者连接到哪种接口，它始终指向同一块存储。如果你将数据放在 `wd2d` 上，然后稍后从系统中移除 `wd1` 并重启，你的数据现在位于 `wd1d` 上，因为旧的 `wd2` 现在是 `wd1`。但是，驱动器的 DUID 在引导后不会改变。

<h2 id="fdisk">使用 fdisk</h2>

<a href="https://man.openbsd.org/fdisk">fdisk(8)</a> 实用程序在某些平台（i386, amd64 和 macppc）上用于创建系统引导 ROM 识别的分区。通常，磁盘上只会放置一个 OpenBSD fdisk 分区，然后该分区将被细分为 disklabel 分区。

使用以下命令查看分区表：

```shell
# fdisk sd0
Disk: sd0       geometry: 553/255/63 [8883945 Sectors]
Offset: 0       Signature: 0xAA55
         Starting       Ending       LBA Info:
 #: id    C   H  S -    C   H  S [       start:      size   ]
------------------------------------------------------------------------
 0: 12    0   1  1 -    2 254 63 [          63:       48132 ] Compaq Diag.
 1: 00    0   0  0 -    0   0  0 [           0:           0 ] unused
 2: 00    0   0  0 -    0   0  0 [           0:           0 ] unused
*3: A6    3   0  1 -  552 254 63 [       48195:     8835750 ] OpenBSD
```

在这里，OpenBSD 分区（id `A6`）标有 `*`，表示它是可引导分区。

完全空白的磁盘需要在引导之前将主引导记录的引导代码写入磁盘。通常，你只需要做：

```shell
# fdisk -iy sd0
```

或者，在交互模式下使用 `reinit` 或 `update` 命令。

`-e` 标志启动交互式编辑模式：

```shell
# fdisk -e sd0
Enter 'help' for information
fdisk: 1>
```

请注意，`quit` 保存更改并退出程序，而 `exit` 不保存并退出。这与许多人现在习惯的其他环境相反。还要注意，fdisk 在保存更改之前不会发出警告。

如果你的系统有维护或诊断分区，建议你将其保留在原位或在安装 OpenBSD **之前** 安装它。

<h2 id="disklabel">磁盘标签 (Disk Labels)</h2>

磁盘标签用于管理 OpenBSD 文件系统分区。它们包含有关磁盘的某些详细信息，例如驱动器几何结构和文件系统信息，如 <a href="https://man.openbsd.org/disklabel.5">disklabel(5)</a> 手册页中深入描述的那样。使用 <a href="https://man.openbsd.org/disklabel">disklabel(8)</a> 命令编辑标签。

这有助于克服某些架构的磁盘分区限制。例如，在 i386 上，只有四个主 MBR 分区可用。使用磁盘标签，这些主分区之一包含所有 OpenBSD 分区，而其他三个仍可用于其他操作系统。

在使用 fdisk 的平台上，你应该在 disklabel 和 fdisk 中都保留第一个逻辑磁道未使用。由于这个原因，默认设置是从块 64 开始第一个分区。

不要在 sparc64 上将交换分区放在磁盘的最开头。虽然 Solaris 经常这样做，但 OpenBSD 要求引导分区位于磁盘的开头。

<h3>删除磁盘标签后恢复分区</h3>

如果分区表损坏，你可以尝试做各种事情来恢复它。

作为日常系统维护的一部分，每个磁盘的 disklabel 副本保存在 `/var/backups` 中。假设你仍然拥有 `/var` 分区，你可以简单地读取输出，并使用 `-R` 标志将其放回 disklabel。

如果你再也看不到该分区，有两种选择：修复足够的磁盘以便你可以看到它，或者修复足够的磁盘以便你可以取出数据。<a href="https://man.openbsd.org/scan_ffs">scan_ffs(8)</a> 实用程序将浏览磁盘以查找分区。你可以使用它找到的信息来重新创建 disklabel。如果你只想找回 `/var`，你可以重新创建 `/var` 的分区，然后恢复备份的标签并从中添加其余部分。<a href="https://man.openbsd.org/disklabel">disklabel(8)</a> 实用程序将更新内核对 disklabel 的理解，并尝试将标签写入磁盘。因此，即使包含 disklabel 的磁盘区域不可读，你也能够在下次重启之前挂载。

<h2 id="BootAmd64">amd64 引导过程</h2>

<a href="https://man.openbsd.org/boot_amd64">boot_amd64(8)</a> 手册页中给出了 amd64 引导过程的详细信息。引导过程如下：

1.  主引导记录 (MBR) 和 GUID 分区表 (GPT)。<a href="https://man.openbsd.org/fdisk">fdisk(8)</a> 手册页包含详细信息。
2.  分区引导记录 (PBR)。引导磁盘 OpenBSD 分区的前 512 字节包含第一阶段引导加载程序 <a href="https://man.openbsd.org/biosboot">biosboot(8)</a>。它由 <a href="https://man.openbsd.org/installboot">installboot(8)</a> 实用程序安装。
3.  第二阶段引导加载程序 `/boot`。PBR 加载 <a href="https://man.openbsd.org/boot.8">boot(8)</a> 程序，该程序的任务是定位并加载内核。

因此，引导过程的最开始可能如下所示：

```text
Using drive 0, partition 3.                      <- MBR
Loading......                                    <- PBR
probing: pc0 com0 com1 mem[638K 1918M a20=on]    <- /boot
disk: hd0+ hd1+
>> OpenBSD/amd64 BOOT 3.33
boot>
booting hd0a:/bsd 4464500+838332 [58+204240+181750]=0x56cfd0
entry point at 0x100120

[ using 386464 bytes of bsd ELF symbol table ]
Copyright (c) 1982, 1986, 1989, 1991, 1993       <- Kernel
        The Regents of the University of California.  All rights reserved.
```

<h2 id="altroot">Root 分区备份 (/altroot)</h2>

OpenBSD 在 <a href="https://man.openbsd.org/daily">daily(8)</a> 脚本中提供了 `/altroot` 功能。如果在 `/etc/daily.local` 或 root 的 <a href="https://man.openbsd.org/crontab.5">crontab(5)</a> 中设置了环境变量 `ROOTBACKUP=1`，并且在 <a href="https://man.openbsd.org/fstab">fstab(5)</a> 中指定了一个分区挂载到 `/altroot` 且挂载选项为 `xx`，那么每天晚上根分区的全部内容将被复制到 `/altroot` 分区。

假设你想将根分区备份到由 <a href="#DUID">DUID</a> `bfb4775bb8397569.a` 指定的分区，请将以下内容添加到 `/etc/fstab`：

```text
bfb4775bb8397569.a /altroot ffs xx 0 0
```

并在 `/etc/daily.local` 中设置适当的环境变量：

```shell
# echo ROOTBACKUP=1 >>/etc/daily.local
```

由于 `/altroot` 过程将捕获你的 `/etc` 目录，这将确保那里的任何配置更改每天都会更新。这是使用 <a href="https://man.openbsd.org/dd">dd(1)</a> 完成的“磁盘镜像”副本，而不是逐文件复制，因此你的 `/altroot` 分区应该至少与你的根分区一样大。通常，你会希望你的 `/altroot` 分区位于不同的磁盘上，该磁盘已配置为在主磁盘发生故障时完全可引导。

<h2 id="DupFS">复制文件系统</h2>

要使用 <a href="https://man.openbsd.org/dump">dump(8)</a> 和 <a href="https://man.openbsd.org/restore">restore(8)</a> 将目录 `/SRC` 下的所有内容复制到目录 `/DST`，请执行：

```shell
# cd /SRC && dump 0f - . | (cd /DST && restore -rf - )
```

或使用 <a href="https://man.openbsd.org/tar">tar(1)</a>：

```shell
# cd /SRC && tar cf - . | (cd /DST && tar xpf - )
```

<h2 id="Quotas">磁盘配额</h2>

配额用于限制某些用户和组可用的磁盘空间量。

使用关键字 `userquota` 和 `groupquota` 在 <a href="https://man.openbsd.org/fstab">fstab(5)</a> 中标记你要对其强制执行配额的每个文件系统。默认情况下，文件 `quota.user` 和 `quota.group` 将在这些文件系统的根目录下创建。这是一个 `/etc/fstab` 行示例：

```text
0123456789abcdef.k /home ffs rw,nodev,nosuid,userquota 1 2
```

要设置用户的配额，请使用 <a href="https://man.openbsd.org/edquota">edquota(8)</a>。例如，发出

```shell
# edquota ericj
```

并编辑软限制和硬限制：

```text
Quotas for user ericj:
/home: KBytes in use: 62, limits (soft = 1000000, hard = 1500000)
        inodes in use: 25, limits (soft = 0, hard = 0)
```

在此示例中，软限制设置为 1000000k，硬限制设置为 1500000k。由于相应的软限制和硬限制设置为 0，因此不会强制执行 inode 数量限制。超过软限制的用户将收到警告，并获得一个宽限期，以便将其磁盘使用量降至限制以下。可以使用 <a href="https://man.openbsd.org/edquota">edquota(8)</a> 上的 `-t` 选项设置宽限期。宽限期结束后，软限制将作为硬限制处理。这通常会导致分配失败。

使用 <a href="https://man.openbsd.org/quotaon">quotaon(8)</a> 启用配额：

```shell
# quotaon -a
```

这将扫描 <a href="https://man.openbsd.org/fstab">fstab(5)</a> 并在具有配额选项的文件系统上启用配额。使用 <a href="https://man.openbsd.org/quota">quota(1)</a> 查看配额统计信息。

<h2 id="foreignfs">访问其他文件系统</h2>

从 <a href="https://man.openbsd.org/mount">mount(8)</a> 手册开始，其中包含解释如何挂载一些最常用文件系统的示例。<a href="https://man.openbsd.org/?query=mount_&amp;sec=8&amp;apropos=1">支持的文件系统</a>和相关命令的部分列表可以通过以下命令获得：

```shell
$ man -k -s 8 mount
```

请注意，支持可能仅限于只读操作。

<h2 id="MountImage">挂载磁盘镜像</h2>

要在 OpenBSD 中挂载磁盘镜像，必须配置 <a href="https://man.openbsd.org/vnd">vnd(4)</a> 设备。例如，如果你有一个位于 `/tmp/ISO.image` 的 ISO 镜像，你将采取以下步骤来挂载该镜像。

```shell
# vnconfig vnd0 /tmp/ISO.image
# mount -t cd9660 /dev/vnd0c /mnt
```

由于这是 CD 和 DVD 使用的 ISO 9660 镜像，因此挂载时必须指定类型 `cd9660`。

要卸载镜像并取消配置 <a href="https://man.openbsd.org/vnd">vnd(4)</a> 设备，请执行：

```shell
# umount /mnt
# vnconfig -u vnd0
```

有关更多信息，请参阅 <a href="https://man.openbsd.org/vnconfig">vnconfig(8)</a> 和 <a href="https://man.openbsd.org/mount">mount(8)</a>。

<h2 id="GrowPartition">扩容磁盘分区</h2>

如果现有分区后面有未分配的可用空间，你可以使用 <a href="https://man.openbsd.org/growfs">growfs(8)</a> 实用程序增加其大小。确保分区当前未挂载。使用 `disklabel -E sd0` 交互式编辑分区表，并使用 `m` 命令修改分区大小。使用 <a href="https://man.openbsd.org/growfs">growfs(8)</a> 调整文件系统以使用整个分区：

```shell
# growfs sd0h
```

在重新挂载分区之前，必须使用 <a href="https://man.openbsd.org/fsck">fsck(8)</a> 检查其完整性：

```shell
# fsck /dev/sd0h
```

<h2 id="softraid">RAID 和磁盘加密</h2>

<a href="https://man.openbsd.org/bioctl">bioctl(8)</a> 命令通过 <a href="https://man.openbsd.org/bio">bio(4)</a> 层管理硬件和软件 RAID 设备。<a href="https://man.openbsd.org/softraid">softraid(4)</a> 子系统允许将多个 OpenBSD <a href="https://man.openbsd.org/disklabel">disklabel(8)</a> 分区组合成一个虚拟 <a href="https://man.openbsd.org/sd">sd(4)</a> 磁盘。此虚拟磁盘被视为任何其他磁盘。

支持的 softraid 规则包括以下内容：

| 级别 | 描述 | 可引导平台 |
| :--- | :--- | :--- |
| RAID0 | 条带化 | --- |
| RAID1 | 镜像 | amd64, arm64, i386, riscv64, sparc64 |
| RAID1C | 带磁盘加密的镜像 | amd64, arm64, riscv64, sparc64 |
| RAID5 | 带浮动奇偶校验的条带化 | --- |
| CONCAT | 无冗余串联 | --- |
| CRYPTO | 磁盘加密 | amd64, arm64, i386, riscv64, sparc64 |

磁盘设置可能因平台而异。请注意，目前 **不支持** “堆叠” softraid 模式。

<h3 id="softraidDI">安装到镜像</h3>

本节介绍将 OpenBSD 安装到镜像硬盘对，并假设你熟悉<a href="./02_installation.md">安装过程</a>。

在使用安装脚本之前，你将进入 shell 并设置 <a href="https://man.openbsd.org/softraid">softraid(4)</a> 设备。

安装内核在引导时只有有限数量的 `/dev` 条目，因此你需要手动为你的 softraid 设置创建所需的磁盘设备。例如，如果你需要支持三个 <a href="https://man.openbsd.org/sd">sd(4)</a> 设备，可以从 shell 提示符执行以下操作：

```text
Welcome to the OpenBSD/amd64 7.8 installation program.
(I)nstall, (U)pgrade, (A)utoinstall or (S)hell? s
# cd /dev
# sh MAKEDEV sd0 sd1 sd2
```

安装程序现在将完全支持 `sd0`、`sd1` 和 `sd2` 设备。如果要从 USB 驱动器安装 <a href="./02_installation.md#FilesNeeded">sets</a>，别忘了也要考虑该设备。

接下来，使用 <a href="https://man.openbsd.org/fdisk">fdisk(8)</a> 初始化磁盘，并使用 <a href="https://man.openbsd.org/disklabel">disklabel(8)</a> 创建 RAID 分区。

如果你从 MBR 引导，请执行：

```shell
# fdisk -iy sd0
# fdisk -iy sd1
```

如果你使用 GPT 进行 UEFI 引导，请执行：

```shell
# fdisk -gy -b 532480 sd0
# fdisk -gy -b 532480 sd1
```

在第一个设备上创建分区布局：

```shell
# disklabel -E sd0
Label editor (enter '?' for help at any prompt)
sd0> a a
offset: [64]
size: [39825135] *
FS type: [4.2BSD] RAID
sd0*> w
sd0> q
No label changes.
```

将分区布局复制到第二个设备：

```shell
# disklabel sd0 > layout
# disklabel -R sd1 layout
# rm layout
```

使用 <a href="https://man.openbsd.org/bioctl">bioctl(8)</a> 命令组装镜像：

```shell
# bioctl -c 1 -l sd0a,sd1a softraid0
scsibus1 at softraid0: 1 targets
sd2 at scsibus2 targ 0 lun 0: <OPENBSD, SR RAID 1, 005> SCSI2 0/direct fixed
sd2: 10244MB, 512 bytes/sec, 20980362 sec total
```

这表明我们现在有一个新的 SCSI 总线和一个新磁盘 `sd2`。此卷将在系统引导时自动检测并组装。

即使你创建多个 RAID 阵列，设备名称也将始终是 `softraid0`。

因为新设备在你期望主引导记录和 disklabel 的地方可能有大量垃圾，强烈建议将其第一个块清零。使用此命令时要**非常小心**；在错误的设备上发出它可能会导致非常糟糕的一天。这假设新的 softraid 设备已创建为 `sd2`。

```shell
# dd if=/dev/zero of=/dev/rsd2c bs=1m count=1
```

你现在准备好在系统上安装 OpenBSD 了。在新的 softraid 磁盘（此示例中的 `sd2`）上创建所有应该存在的分区，而不是在 `sd0` 或 `sd1`（非 RAID 磁盘）上。

要检查镜像的状态，请发出以下命令：

```shell
# bioctl sd2
```

每晚运行 cron 作业来检查状态可能是一个好主意。

<h4>重建镜像</h4>

当驱动器发生故障时，你将更换故障驱动器，创建 RAID 和其他 disklabel 分区，然后重建镜像。假设你的 RAID 卷是 `sd2`，并且你正在用 `sd1m` 替换故障设备，以下命令应该有效：

```shell
# bioctl -R /dev/sd1m sd2
```

这也可以在 <a href="./03_system.md#LostPW">单用户模式</a> 下或从 <a href="./02_installation.md#bsd.rd">内存盘内核</a> 中执行。

<h3 id="softraidFDE">全盘加密</h3>

与 RAID 非常相似，OpenBSD 中的全盘加密由 <a href="https://man.openbsd.org/softraid">softraid(4)</a> 子系统和 <a href="https://man.openbsd.org/bioctl">bioctl(8)</a> 命令处理。OpenBSD 安装程序提供了一种安装到完全加密磁盘的方法，使用密码短语来解锁它。

作为使用密码短语的替代方法，也可以使用存储在单独设备（例如 USB 记忆棒）上的密钥来加密磁盘。

在安装程序的 `(S)hell` 提示符下，使用 <a href="https://man.openbsd.org/fdisk">fdisk(8)</a> 初始化密钥盘，然后使用 <a href="https://man.openbsd.org/disklabel">disklabel(8)</a> 为密钥数据创建一个 1 MB 的 RAID 分区。如果你的密钥盘是 `sd1`，而你想加密的驱动器是 `sd0`，输出将如下所示：

```shell
# bioctl -c C -k sd1a -l sd0a softraid0
sd2 at scsibus3 targ 1 lun 0: <OPENBSD, SR CRYPTO, 005> SCSI2 0/direct fixed
sd2: 19445MB, 512 bytes/sector, 39824607 sectors
softraid0: CRYPTO volume attached as sd2
```

因为你使用了密钥盘，所以不会提示你输入密码短语。必须在启动时插入密钥盘。

你可以使用 <a href="https://man.openbsd.org/dd">dd(1)</a> 备份和恢复你的密钥盘：

```shell
# dd bs=8192 skip=1 if=/dev/rsd1a of=backup-keydisk.img
# dd bs=8192 seek=1 if=backup-keydisk.img of=/dev/rsd1a
```

<h3 id="softraidCrypto">加密外部磁盘</h3>

本节解释如何为外部 USB 驱动器设置加密 softraid 卷。步骤概述如下：

- 用随机数据覆盖驱动器内容
- 使用 <a href="https://man.openbsd.org/disklabel">disklabel(8)</a> 创建所需的 RAID 类型分区
- 使用 <a href="https://man.openbsd.org/bioctl">bioctl(8)</a> 加密驱动器
- 将新伪分区的第一个兆字节清零
- 使用 <a href="https://man.openbsd.org/newfs">newfs(8)</a> 在伪设备上创建文件系统
- 解锁并 <a href="https://man.openbsd.org/mount">mount(8)</a> 新伪设备
- 根据需要访问文件
- 卸载驱动器并分离加密容器

以下是步骤的快速示例演示，其中 `sd3` 是 USB 驱动器。

```shell
# dd if=/dev/urandom of=/dev/rsd3c bs=1m
# fdisk -iy sd3
# disklabel -E sd3 # 创建一个类型为 RAID 的 "a" 分区
# bioctl -c C -l sd3a softraid0
New passphrase:
Re-type passphrase:
softraid0: CRYPTO volume attached as sd4
# dd if=/dev/zero of=/dev/rsd4c bs=1m count=1
# fdisk -iy sd4
# disklabel -E sd4 # 创建一个 "i" 分区
# newfs sd4i
# mkdir -p /mnt/secretstuff
# mount /dev/sd4i /mnt/secretstuff
# mv somefile /mnt/secretstuff/
# umount /mnt/secretstuff
# bioctl -d sd4
```

用于创建卷的相同 <a href="https://man.openbsd.org/bioctl">bioctl(8)</a> 命令可用于稍后附加它。

