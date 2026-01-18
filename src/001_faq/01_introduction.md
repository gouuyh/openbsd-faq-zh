## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">FAQ - OpenBSD 简介</span>

---

- <a href="#WhatIs">关于 OpenBSD</a>
- <a href="#Platforms">硬件支持</a>
- <a href="#ManPages">手册页 (Manual Pages)</a>
- <a href="#MailLists">邮件列表</a>
- <a href="#OtherUnixes">迁移到 OpenBSD</a>
- <a href="#Bugs">报告 Bug</a>
- <a href="#Support">支持本项目</a>

---

<h2 id="WhatIs">关于 OpenBSD</h2>

<a href="https://www.openbsd.org/index.html">OpenBSD</a> 项目开发了一个免费、多平台、基于 4.4BSD 的类 UNIX 操作系统。我们的<a href="https://www.openbsd.org/goals.html">目标</a>强调正确性、安全性、标准化和可移植性。

<h3>我为什么要使用它？</h3>

我们认为 OpenBSD 是一个有用的操作系统的部分原因如下：

- OpenBSD 可以在许多不同的硬件平台上运行。
- 由于持续不断的全面源代码审计，OpenBSD 被许多安全专业人士认为是<a href="https://www.openbsd.org/security.html">最安全</a>的类 UNIX 操作系统。
- OpenBSD 是一个功能齐全的类 UNIX 操作系统，免费提供源代码和二进制形式。
- OpenBSD 集成了尖端的安全技术，适用于在分布式环境中<a href="../002_pf/17_example1.md">构建防火墙</a>和私有网络服务。
- OpenBSD 受益于许多领域强大的持续开发，提供了使用新兴技术的机会，并拥有一个由开发者和最终用户组成的国际社区。
- OpenBSD 试图最大限度地减少定制和调整的需要。对于绝大多数用户来说，OpenBSD 在他们的硬件上运行他们的应用程序时**开箱即用**。

<h3 id="ReallyFree">OpenBSD 真的免费吗？</h3>

OpenBSD 是完全免费的。二进制文件是免费的。源代码是免费的。OpenBSD 的所有部分都拥有合理的版权条款，允许自由再分发。关于 OpenBSD 版权政策的更多信息可以在<a href="https://www.openbsd.org/policy.html">这里</a>找到。

OpenBSD 的维护者主要自费支持该项目。这包括为项目编程花费的时间、用于支持众多架构移植的设备、用于向您分发 OpenBSD 的网络资源，以及回答问题和调查用户 Bug 报告所花费的时间。OpenBSD 开发者并非独立富豪，即使是时间、设备和资源的微小贡献也会产生巨大的影响。

<h3>基础系统中包含什么？</h3>

OpenBSD 分发时包含许多第三方软件产品，包括：

- <a href="https://www.x.org/wiki">X.org</a>
- <a href="https://clang.llvm.org">LLVM/Clang</a>
- <a href="https://gcc.gnu.org">GCC</a>
- <a href="https://www.perl.org">Perl</a>
- <a href="https://www.nlnetlabs.nl/projects/nsd">NSD</a> 和 <a href="https://www.nlnetlabs.nl/projects/unbound">Unbound</a>
- <a href="https://www.gnu.org/software/ncurses">ncurses</a>
- <a href="https://www.gnu.org/software/binutils">binutils</a>
- <a href="https://www.gnu.org/software/gdb/gdb.html">gdb</a>
- <a href="https://github.com/Yubico/libfido2">libfido2</a>
- <a href="https://libexpat.github.io/">Expat</a>
- <a href="https://www.zlib.net/">zlib</a>

OpenBSD 团队经常对第三方产品进行修补，通常是为了提高代码的安全性或质量。系统中也包含了许多<a href="https://www.openbsd.org/innovations.html">自研软件</a>。额外的应用程序可以通过 <a href="./04_package.md">软件包 (packages)</a> 获取。

<h3 id="HowAbout">为什么包含/不包含某产品？</h3>

人们经常问为什么 OpenBSD 包含或不包含某个特定产品。答案基于两点：开发者的意愿以及与项目<a href="https://www.openbsd.org/goals.html">目标</a>的兼容性。许可证通常是最大的问题：我们希望 OpenBSD 能够被世界上任何人在任何地方用于任何目的。

<h3 id="Next">下一个版本何时发布？</h3>

OpenBSD 团队大约每六个月发布一个新版本，目标发布日期在 5 月和 11 月。关于开发周期的更多信息可以在<a href="./05_building-from-source.md#Flavors">这里</a>找到。

<h2 id="Platforms">硬件支持</h2>

OpenBSD 运行在以下<a href="https://www.openbsd.org/plat.html">平台</a>上：
<!-- XXXrelease - verify platform list is current -->

- <a href="https://www.openbsd.org/alpha.html">alpha</a>
- <a href="https://www.openbsd.org/amd64.html">amd64</a>
- <a href="https://www.openbsd.org/arm64.html">arm64</a>
- <a href="https://www.openbsd.org/armv7.html">armv7</a>
- <a href="https://www.openbsd.org/hppa.html">hppa</a>
- <a href="https://www.openbsd.org/i386.html">i386</a>
- <a href="https://www.openbsd.org/landisk.html">landisk</a>
- <a href="https://www.openbsd.org/loongson.html">loongson</a>
- <a href="https://www.openbsd.org/luna88k.html">luna88k</a>
- <a href="https://www.openbsd.org/macppc.html">macppc</a>
- <a href="https://www.openbsd.org/octeon.html">octeon</a>
- <a href="https://www.openbsd.org/powerpc64.html">powerpc64</a>
- <a href="https://www.openbsd.org/riscv64.html">riscv64</a>
- <a href="https://www.openbsd.org/sparc64.html">sparc64</a>

具体的硬件支持细节请参阅各自的平台页面。

<h2 id="ManPages">手册页 (Manual Pages)</h2>

OpenBSD 附带了以手册页 (man pages) 形式存在的详尽文档。它们是 OpenBSD 的权威信息来源，我们投入了大量精力确保其更新和准确。开发者在更改系统代码时，应同步更新手册页。我们期望用户在寻求帮助之前先查阅手册页。

以下是一些对新用户有用的手册页列表：

- <a href="https://man.openbsd.org/afterboot">afterboot(8)</a> - 首次完整启动后需要检查的事项
- <a href="https://man.openbsd.org/help">help(1)</a> - 给新用户和管理员的帮助信息
- <a href="https://man.openbsd.org/hier">hier(7)</a> - 文件系统的布局
- <a href="https://man.openbsd.org/man">man(1)</a> - 显示手册页
- <a href="https://man.openbsd.org/adduser">adduser(8)</a> 和 <a href="https://man.openbsd.org/rmuser">rmuser(8)</a> - 添加或删除新用户
- <a href="https://man.openbsd.org/reboot">reboot(8), halt(8)</a> 和 <a href="https://man.openbsd.org/shutdown">shutdown(8)</a> - 停止和重启系统
- <a href="https://man.openbsd.org/syspatch">syspatch(8)</a> - 应用<a href="./03_system.md#Patches">安全和可靠性更新</a>
- <a href="https://man.openbsd.org/sysupgrade">sysupgrade(8)</a> - 升级到下一个 OpenBSD 发行版或更新的快照
- <a href="https://man.openbsd.org/dmesg">dmesg(8)</a> - 重新显示内核启动信息
- <a href="https://man.openbsd.org/doas">doas(1)</a> - 以其他用户身份运行命令
- <a href="https://man.openbsd.org/tmux">tmux(1)</a> - 终端复用器
- <a href="https://man.openbsd.org/ifconfig">ifconfig(8)</a> - 配置网络接口参数
- <a href="https://man.openbsd.org/ftp">ftp(1)</a> - 从互联网下载文件（支持 FTP/HTTP/HTTPS）
- <a href="https://man.openbsd.org/login.conf">login.conf(5)</a> - 登录类别配置文件的格式
- <a href="https://man.openbsd.org/sendbug">sendbug(1)</a> - 报告你发现的 Bug

所有的 OpenBSD 手册页都可以在 <a href="https://man.openbsd.org">man.openbsd.org</a> 网站上找到，也可以在 <code>man78.tgz</code> 文件集中找到。

通常，如果你知道命令或手册页的名称，可以通过执行 <code>man command</code> 来阅读它。如果你不知道命令的名称，或者 <code>man command</code> 找不到手册页，你可以通过执行 <code>apropos something</code> 或 <code>man -k something</code> 来搜索手册页数据库，其中 <code>something</code> 是可能出现在你要查找的手册页标题中的词。

```shell
$ apropos "time zone"
tzfile(5) - time zone information
zdump(8) - time zone dumper
zic(8) - time zone compiler
```

括号中的数字表示该页面所在的<a href="https://man.openbsd.org/man#SECTIONS">手册章节</a>。在某些情况下，你可能会发现同名的手册页存在于不同的章节中。例如，假设你想知道 cron 守护进程配置文件的格式。一旦你知道了所需页面的章节号，就可以执行 <code>man n command</code>，其中 n 是手册章节号。

```shell
$ man -k cron
cron(8) - clock daemon
crontab(1) - maintain crontab files for individual users
crontab(5) - tables for driving cron
$ man 5 crontab
```

<h2 id="MailLists">邮件列表</h2>

OpenBSD 项目维护了几个邮件列表，用户可以订阅并关注。
一些较受欢迎的列表包括：

- **announce** - 公告和安全建议
- **bugs** - 通过 <a href="https://man.openbsd.org/sendbug">sendbug(1)</a> 接收到的 Bug 及其讨论
- **misc** - 一般用户问答
- **ports** - 关于 <a href="../003_ports/00_index.md">ports 树</a>的讨论
- **source-changes** - CVS 源码树变更的自动邮件通知
- **tech** - 面向 OpenBSD 开发者和高级用户的技术话题讨论

在任何邮件列表上提问之前，请检查存档，因为大多数常见问题已经被反复问过。虽然这可能是你第一次遇到这个问题，但邮件列表上的其他人可能在过去一周内已经看到过同样的问题好几次了，并且可能不喜欢再次看到它。如果询问可能与硬件有关的问题，请务必附上完整的 <a href="https://man.openbsd.org/dmesg">dmesg(8)</a>。

你可以在<a href="https://www.openbsd.org/mail.html">邮件列表页面</a>找到多个存档、其他准则和更多信息。可以通过 <a href="https://lists.openbsd.org">Web 界面</a>轻松管理订阅。

<h2 id="OtherUnixes">迁移到 OpenBSD</h2>

如果你通过阅读任何关于通用 Unix 的<a href="https://www.openbsd.org/books.html">好书</a>学习了 Unix，理解了 Unix 哲学，然后将知识扩展到特定平台，你会发现 OpenBSD 非常熟悉。

以下是 OpenBSD 与其他 Unix 变体之间常见的一些区别：

- OpenBSD 是 BSD 风格的 Unix，紧密遵循 4.4BSD 设计。Linux 和 Solaris 是 System V 风格的系统。一些类 Unix 操作系统混合了 System V 和 BSD 的特性。这通常会在<a href="./03_system.md#rc">启动脚本</a>方面引起混淆。OpenBSD 使用 <a href="https://man.openbsd.org/rc">rc(8)</a> 系统。
- OpenBSD 是一个完整的系统，旨在保持同步。它不是一个内核加上可以彼此独立升级的实用程序。
- OpenBSD 维护一个 <a href="../003_ports/00_index.md">ports 树</a>来提供第三方软件。预编译的 <a href="./04_package.md">软件包 (packages)</a> 由 OpenBSD ports 团队创建和分发。
- OpenBSD 使用 CVS 来跟踪源代码变更。OpenBSD 开创了 <a href="https://www.openbsd.org/anoncvs.html">匿名 CVS</a>，允许任何人在任何时间提取任何版本 OpenBSD 的完整源代码树。还有一个 <a href="https://cvsweb.openbsd.org">Web 界面</a>。
- OpenBSD 经历了繁重且持续的安全审计，以确保代码的质量和安全性。
- OpenBSD 不支持日志文件系统 (journaling filesystems)。
- OpenBSD 附带 <a href="../002_pf/00_index.md">Packet Filter (PF)</a>。这意味着网络地址转换 (NAT)、排队和过滤是通过 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a>、<a href="https://man.openbsd.org/pf">pf(4)</a> 和 <a href="https://man.openbsd.org/pf.conf">pf.conf(5)</a> 处理的。
- OpenBSD 的默认 Shell 是 <a href="https://man.openbsd.org/ksh">ksh</a>，它基于公共领域的 Korn shell。bash 等其他 Shell 可以从 <a href="./04_package.md">软件包</a> 中添加。
- 设备按驱动程序命名，而不是按类型命名。换句话说，没有 <code>eth0</code> 和 <code>eth1</code> 这样的设备名。Intel PRO/1000 以太网卡可能是 <code>em0</code>，Broadcom BCM57xx 或 BCM590x 以太网设备可能是 <code>bge0</code>，RaLink 无线设备可能是 <code>ral0</code> 等。
- OpenBSD/i386、amd64 和其他几个平台使用两层磁盘分区系统，第一层是 <a href="./06_disk-setup.md#fdisk">fdisk</a> BIOS 可见分区，第二层是 <a href="./06_disk-setup.md#disklabel">disklabel</a>。
- 一些其他操作系统鼓励你为你的机器定制内核。OpenBSD 鼓励用户直接使用开发者提供并测试的标准 GENERIC 内核。

<h2 id="Bugs">报告 Bug</h2>

报告 Bug 是最终用户最重要的责任之一。诊断大多数严重问题需要非常详细的信息。例如，以下是一个合适的 Bug 报告：

```text
From: user@example.com
To: bugs@openbsd.org
Subject: 3.3-beta panics on a SPARCStation2

OpenBSD 3.2 installed from an official CD-ROM installed and ran fine
on this machine.

After doing a clean install of 3.3-beta from a mirror, I find the
system randomly panics after a period of use, and predictably and
quickly when starting X.

This is the dmesg output:

[...]

This is the panic I got when attempting to start X:

panic: pool_get(mclpl): free list modified: magic=78746572; page 0xfaa93000;
 item addr 0xfaa93000
Stopped at      Debugger+0x4:   jmpl            [%o7 + 0x8], %g0
https://www.openbsd.org/ddb.html describes the minimum info required in bug
reports. Insufficient info makes it difficult to find and fix bugs.
ddb> trace
[...]

Thank you!
```

请参阅<a href="https://www.openbsd.org/report.html">此页面</a>了解有关创建和提交 Bug 报告的更多信息。请提供关于发生了什么、系统的确切配置以及如何重现该问题的详细信息。只要可能，请使用 <a href="https://man.openbsd.org/sendbug">sendbug(1)</a> 来报告你的问题。否则，请至少包含你系统的 <a href="https://man.openbsd.org/dmesg">dmesg(8)</a> 输出。<a href="https://man.openbsd.org/sendbug">sendbug(1)</a> 命令要求你的系统能够发送电子邮件。

OpenBSD 邮件服务器使用 <a href="https://man.openbsd.org/spamd">spamd(8)</a> 进行灰名单 (greylisting) 过滤，因此邮件服务器可能需要半小时左右才会接受你的 Bug 报告。请耐心等待。

提交 Bug 报告后，开发者可能会联系你以获取更多信息或提供需要测试的补丁。你也可以监控 <a href="mailto:bugs@openbsd.org">bugs@openbsd.org</a> 邮件列表的存档 - 详情见<a href="https://www.openbsd.org/mail.html">邮件列表页面</a>。

<h2 id="Support">支持本项目</h2>

我们非常感谢为 OpenBSD 项目做出贡献的个人和组织。

OpenBSD 持续需要社区提供多种类型的支持。如果你觉得 OpenBSD 有用，强烈建议你寻找一种方式来做出贡献。

- <a href="https://www.openbsd.org/donations.html">捐赠资金</a>。项目持续需要现金来支付设备、网络连接等费用。即使是小额捐款也会产生深远的影响。
- <a href="https://www.openbsd.org/want.html">捐赠设备和零件</a>。项目持续需要通用和特定的硬件。
- <a href="./05_building-from-source.md#Diff">贡献你的时间和技能</a>。喜欢编写操作系统的程序员自然总是受欢迎的，但人们还有许多其他方式可以发挥作用。
- 关注<a href="https://www.openbsd.org/mail.html">邮件列表</a>并帮助回答其他用户的问题。
- 通过向 <a href="mailto:misc@openbsd.org">misc@openbsd.org</a> 提交新的 FAQ 材料来帮助维护文档。
- 组建本地<a href="https://www.openbsd.org/groups.html">用户组</a>，让你的朋友迷上 OpenBSD。
- 向你的雇主推荐在工作中使用 OpenBSD。如果你是学生，与你的教授讨论将 OpenBSD 作为计算机科学或工程课程的学习工具。