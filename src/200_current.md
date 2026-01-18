## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">跟随 -current 并使用快照</span>

OpenBSD 的活跃开发分支被称为 <a href="./001_faq/05_building-from-source.md#Flavors">-current</a> 分支。这些源代码经常被编译成称为*快照 (snapshots)* 的发布版本。

有时激进的更改会被推送到此分支，在构建最新代码或从以前的时间点升级时可能会出现并发症。本页解释了克服这些障碍的一些步骤。在使用 -current 和以下说明之前，请确保你已阅读并理解如何 <a href="./001_faq/05_building-from-source.md">从源代码构建系统</a>。

通常，使用快照要容易得多，因为开发者已经为你解决了大部分麻烦。

你应该**始终**使用快照作为运行 -current 的起点。此过程通常包括运行带 `-s` 标志的 <a href="https://man.openbsd.org/sysupgrade">sysupgrade(8)</a>。或者，从你首选 <a href="https://www.openbsd.org/ftp.html">镜像站</a> 的 `/snapshots/` 目录下载（并验证）适当的 <a href="./001_faq/02_installation.md#bsd.rd">bsd.rd</a> 文件，从中引导，并在提示符下选择 `(U)pgrade`。引导进入新系统后，应 <a href="./001_faq/04_package.md#PkgUpdate">升级</a> 任何已安装的软件包。

除专家外，不建议通过编译自己的源代码升级到 -current，因为构建时的困难交叉点经常发生，并且不提供任何帮助。如果失败，请使用快照进行恢复。

大多数这些更改必须以 root 身份执行。

<h3 id="r20241122">2024/11/22 - [packages] PostgreSQL 主要更新</h3>

PostgreSQL 有一个到 17.2 的主要更新。按照 <a href="https://cvsweb.openbsd.org/cgi-bin/cvsweb/~checkout~/ports/databases/postgresql/pkg/README-server?rev=1.37&amp;content-type=text/plain">postgresql-server pkg-readme</a> 中的描述使用 `pg_upgrade`，或者执行转储/恢复 (dump/restore)。

<h3 id="r20241124">2024/11/24 - [packages] minio server: 移除网关和文件系统模式</h3>

MinIO 网关和相关的文件系统模式在 2020 年 7 月进入了功能冻结期。2022 年 2 月，MinIO 宣布弃用 MinIO 网关。随着弃用公告，MinIO 还宣布该功能将在六个月后移除。

MinIO port 已更新到一个移除了 MinIO 网关和相关文件系统模式代码的版本。仍在使用独立或文件系统 MinIO 模式的部署如果升级到该最新版本将无法启动。

支持文件系统模式的最后一个 MinIO 版本 (0.20221024) 仍然作为软件包提供。如果你执行 "pkg_add minio"，将会提供版本选择。

上游项目提供了一个 <a href="https://min.io/docs/minio/linux/operations/install-deploy-manage/migrate-fs-gateway.html">迁移到新的单节点单驱动器部署的过程</a>。

<h3 id="r20241216">2024/12/16 - [packages] courier-imap: 移除了 -noidentlookup 选项</h3>

在 courier-imap 和 courier-pop3 上，`-noidentlookup` 选项已被移除，必须相应地调整配置文件。

<h3 id="r20241218">2024/12/18 - [packages] ttyd: 默认更改为只读</h3>

ttyd 已从默认可写的 Web 终端更改为只读。如果你需要它接受用户输入，请使用 `-W` 标志。

<h3 id="r20250127">2025/01/27 - [perl] PerlIO::scalar 不再是 XS 模块</h3>

Perl 已更新至 5.40.1，某些文件不再包含在内，可以移除。

```shell
# rm -rf /usr/libdata/perl5/*/PerlIO/scalar.pm \
         /usr/libdata/perl5/*/auto/PerlIO/scalar
```

<h3 id="r20251023">2025/10/23 - OpenBSD/luna88k 编译器切换</h3>

<a href="https://www.openbsd.org/luna88k.html">OpenBSD/luna88k</a> 移植已将编译器从 gcc 3.3.6 切换到 gcc 4.2.1。强烈建议使用快照进行更新。要从源代码更新，需要执行以下步骤：

1. 安装新的 Makefile 机制。

   ```shell
   # cd /usr/src/share/mk && make install
   ```

2. 构建并安装新的编译器。

   ```shell
   # cd /usr/src/gnu/usr.bin/cc && su build -c 'exec make' && make install
   ```

3. 构建并安装预处理器。

   ```shell
   # cd /usr/src/usr.bin/cpp && su build -c 'exec make' && make install
   ```

4. 像往常一样重建系统。

   ```shell
   # cd /usr/src && make build
   ```

---

<small>
$OpenBSD: current.html,v 1.1145 2025/11/23 03:16:10 jeremy Exp $
</small>
