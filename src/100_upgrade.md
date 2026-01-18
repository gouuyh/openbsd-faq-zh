## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">升级指南：7.7 到 7.8</span>

---

<small>
<a href="./000_MainPage.md">[FAQ 索引]</a> |
<a href="https://www.openbsd.org/faq/upgrade77.html">[7.6 -> 7.7]</a>
</small>

---

> **注意**：本文档仅涵盖 OpenBSD 最新版本的升级指南。如果您需要跨版本升级或查找旧版本的升级说明，请访问 [OpenBSD 官方网站](https://www.openbsd.org/faq/upgrade.html)。

> *仅支持从一个发行版升级到紧随其后的下一个发行版。*
>
> **_在尝试之前，请通读并理解此过程。对于关键或物理上远程的机器，请先在相同的本地系统上进行测试。_**

<h3 id="BeforeUpdate">在使用任何升级方法之前</h3>

- **检查 /usr 中的可用磁盘空间。**
  验证 `/usr` 分区的大小至少为 1.1G。如果空间不足，升级可能会失败，你应该考虑重新安装系统。

- **阅读配置和语法变更以及软件包升级说明。**
  有一些 <a href="#ConfigChanges">配置变更</a> 和 <a href="#SpecialPackages">软件包变更</a> 可能需要在开始升级之前进行规划。

<h3 id="UpgradeMethods">升级方法</h3>

- **无人值守升级：**
  最简单的方法是使用 <a href="https://man.openbsd.org/OpenBSD-7.8/sysupgrade.8">sysupgrade(8)</a> 进行无人值守升级。该程序将下载所有安装集，验证其签名，并重新启动以自动执行升级。无人值守升级完成后，请继续阅读 <a href="#AfterSets">下文</a>。

- **交互式升级：**
  如果你坚持要省略某些安装集，你将需要执行 <a href="#InteractiveUpgrade">交互式升级</a>。（sysupgrade 会升级所有安装集。）

- **手动升级：**
  最后的选项是使用 <a href="#NoInstKern">手动升级过程</a>。（不推荐这样做，因为这是最容易出错的方法。）

---

<h3 id="InteractiveUpgrade">交互式升级</h3>

- **获取并验证 `bsd.rd`。**
  下载适用于你架构的内存盘内核和加密签名的校验和文件。

  <dl>
    <dt><code>bsd.rd</code></dt>
    <dd>
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/alpha/bsd.rd">alpha</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/bsd.rd">amd64</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/arm64/bsd.rd">arm64</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/armv7/bsd.rd">armv7</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/hppa/bsd.rd">hppa</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/i386/bsd.rd">i386</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/landisk/bsd.rd">landisk</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/luna88k/bsd.rd">luna88k</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/macppc/bsd.rd">macppc</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/octeon/bsd.rd">octeon</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/powerpc64/bsd.rd">powerpc64</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/riscv64/bsd.rd">riscv64</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/sparc64/bsd.rd">sparc64</a>]
    </dd>
    <dt><code>SHA256.sig</code></dt>
    <dd>
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/alpha/SHA256.sig">alpha</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/SHA256.sig">amd64</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/arm64/SHA256.sig">arm64</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/armv7/SHA256.sig">armv7</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/hppa/SHA256.sig">hppa</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/i386/SHA256.sig">i386</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/landisk/SHA256.sig">landisk</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/luna88k/SHA256.sig">luna88k</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/macppc/SHA256.sig">macppc</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/octeon/SHA256.sig">octeon</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/powerpc64/SHA256.sig">powerpc64</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/riscv64/SHA256.sig">riscv64</a>]
    [<a href="https://cdn.openbsd.org/pub/OpenBSD/7.8/sparc64/SHA256.sig">sparc64</a>]
    </dd>
  </dl>

  使用 <a href="https://man.openbsd.org/OpenBSD-7.8/signify">signify(1)</a> 验证 `bsd.rd` 和 `SHA256.sig`：

  ```shell
  $ signify -C -p /etc/signify/openbsd-78-base.pub -x SHA256.sig bsd.rd
  Signature Verified
  bsd.rd: OK
  ```

- 接下来，从上一步获取的安装内核 `bsd.rd` 启动。将其放在文件系统的根目录中，并指示引导加载程序引导此内核。一旦此内核启动，选择 `(U)pgrade` 选项并按照提示操作。

- 安装完文件集后，系统将使用升级后的内核重新启动。现在 <a href="#AfterSets">继续下一步</a>：

---

<h3 id="AfterSets">升级之后</h3>

升级安装集后，系统将使用升级后的内核重新启动，并在引导期间运行 <a href="https://man.openbsd.org/OpenBSD-7.8/sysmerge">sysmerge(8)</a>。在某些情况下，配置文件无法自动修改。运行

```shell
# sysmerge
```

来检查并执行这些 <a href="#ConfigChanges">配置变更</a>。

接下来 <a href="#RmFiles">移除旧文件</a>。
最后使用 `pkg_add -u` 升级软件包。

你可能希望查看 <a href="https://www.openbsd.org/errata78.html">勘误表页面</a> 以获取任何发布后的修复。

---

<h2 id="NoInstKern">手动升级（不使用安装内核）</h2>

<b style="color:#e00000">这不是推荐的过程。如果可能，请使用无人值守或交互式升级方法！</b>

有时，你需要对无法进行正常无人值守或交互式升级过程的机器执行升级。

<h3>准备工作</h3>

- **将安装文件放在一个好的位置。**
  确保你有足够的空间！在远程升级时空间不足可能会……很不幸。建议在 `/usr` 上至少有 500MB 的可用空间。

- **成为 root。**
  虽然在每个命令前使用 <a href="https://man.openbsd.org/OpenBSD-7.8/doas">doas(1)</a> 通常是一个好习惯，但该命令可能会被最后几步破坏，因此你应该在开始此过程之前成为 root。此时最好使用 doas 以外的方法验证你的 root 访问权限，即直接登录或使用 <a href="https://man.openbsd.org/OpenBSD-7.8/su">su(1)</a>。

- **停止和/或禁用任何适当的应用程序。**
  在此过程中，所有用户空间应用程序都将被替换，但可能无法运行，结果可能会发生奇怪的事情。在第一次重启期间，你可能还会遇到 DNS 解析问题，因此依赖于 DNS 的 PF 规则和 NFS 挂载可能会导致启动问题。可能还有其他应用程序你希望在升级后立即停止运行；也请停止并禁用它们。

- **安装新的引导块。**
  这实际上应该在任何升级结束时完成。如果忽略了这一点，那么现在不这样做可能会破坏串口控制台或其他东西，具体取决于你的平台。使用 <a href="https://man.openbsd.org/OpenBSD-7.8/amd64/installboot">installboot(8)</a>，假设 `sd0` 是你的引导磁盘：

  ```shell
  # installboot sd0
  ```

<h3>手动升级</h3>

- **安装新内核。**
  复制主内核的额外步骤是为了确保磁盘上始终有一个有效的内核。

  如果使用多处理器内核：
  ```shell
  # cd /usr/rel    # 你放置发布文件的位置
  # ln -f /bsd /obsd && cp bsd.mp /nbsd && mv /nbsd /bsd
  # cp bsd.rd /
  # cp bsd /bsd.sp
  ```

  如果使用单处理器内核：
  ```shell
  # cd /usr/rel    # 你放置发布文件的位置
  # ln -f /bsd /obsd && cp bsd /nbsd && mv /nbsd /bsd
  # cp bsd.rd bsd.mp /    # 可能会给出一个无害的警告
  ```

- **启用 KARL。**
  存储内核的校验和：
  ```shell
  # sha256 -h /var/db/kernel.SHA256 /bsd
  ```

- **安装新的用户空间。**
  保存 `reboot(8)` 的副本，提取并安装发布版 tar 包，重启。最后安装 `base78.tgz`，因为新的基础系统，特别是 <a href="https://man.openbsd.org/OpenBSD-7.8/tar">tar(1)</a>、<a href="https://man.openbsd.org/OpenBSD-7.8/gzip">gzip(1)</a> 和 <a href="https://man.openbsd.org/OpenBSD-7.8/reboot">reboot(8)</a>，将无法与旧内核一起工作。

  手动解压所需的文件集：
  ```shell
  # cp /sbin/reboot /sbin/oreboot
  # tar -C / -xzphf xshare78.tgz
  # tar -C / -xzphf xserv78.tgz
  # tar -C / -xzphf xfont78.tgz
  # tar -C / -xzphf xbase78.tgz
  # tar -C / -xzphf man78.tgz
  # tar -C / -xzphf game78.tgz
  # tar -C / -xzphf comp78.tgz
  # tar -C / -xzphf base78.tgz    # 最后安装！
  # /sbin/oreboot
  ```

  或者，如果你使用 <a href="https://man.openbsd.org/OpenBSD-7.8/ksh">ksh(1)</a>，你可以这样做：
  ```shell
  # cp /sbin/reboot /sbin/oreboot
  # for _f in [!b]*78.tgz base78.tgz; do tar -C / -xzphf "$_f" || break; done
  # /sbin/oreboot
  ```
  注意 <a href="https://man.openbsd.org/OpenBSD-7.8/tar">tar(1)</a> 每次调用只能展开一个归档文件，所以简单的通配符不起作用。

- **重启后，更新 `/dev`。**
  运行 <a href="https://man.openbsd.org/OpenBSD-7.8/amd64/MAKEDEV">MAKEDEV(8)</a>：
  ```shell
  # cd /dev
  # ./MAKEDEV all
  ```

- **更新引导加载程序。**
  仍然假设 `sd0` 是你的引导磁盘：
  ```shell
  # installboot sd0
  ```

- **更新系统配置文件。**
  运行 <a href="https://man.openbsd.org/OpenBSD-7.8/sysmerge">sysmerge(8)</a>：
  ```shell
  # sysmerge
  ```

- **更新固件。**
  你的系统可能有新的固件。使用 <a href="https://man.openbsd.org/OpenBSD-7.8/fw_update">fw_update(8)</a> 更新它：
  ```shell
  # fw_update
  ```

- **完成。**
  查看引导时的控制台输出（使用 `dmesg -s`）并在必要时更正任何故障。下面 <a href="#ConfigChanges">配置变更</a> 之后的所有步骤也适用于手动升级。最后，移除 `/sbin/oreboot` 并更新软件包：`pkg_add -u`。再次重启以确保你使用的是最新的固件文件，并运行由 KARL 生成的你自己的内核。

---

<h3 id="ConfigChanges">配置和语法变更</h3>

- `build` 用户需要其自己的 `build` 登录类别而不是 `default` 类别才能构建 LLVM 19。
  <a href="https://man.openbsd.org/OpenBSD-7.8/sysmerge">sysmerge(8)</a> 实用程序将登录类别添加到 <a href="https://man.openbsd.org/OpenBSD-7.8/login.conf">login.conf(5)</a>，但需要手动将 build 用户添加到此类：
  ```shell
  # user mod -L build build
  ```

<h3 id="RmFiles">需要移除的文件</h3>

- 本次发布没有特别需要注意的文件。

<h3 id="SpecialPackages">特殊软件包</h3>

- **freetds.**
  FreeTDS 1.5 增加了对 TDS 协议版本 8.0 的支持。如果你之前在 /etc/freetds.conf 中有 `"tds version = 8.0"`，它会回退到旧版本，更新后可能会有问题。你可能希望在更新之前将其更改为特定版本或 `"tds version = auto"`，并在尝试 "8.0" 之前进行测试。

- **php.**
  Ports 中的默认 PHP 版本已从 8.2 更改为 8.3。请按照 /usr/local/share/doc/pkg-readmes/php-8.3 的“在 PHP 发布分支之间移动”部分中的说明更新 .ini 文件符号链接等。

- **rbw.**
  从 OpenBSD 7.8 开始，对 Yubico OTP 认证的支持已被移除。在添加或同步帐户之前，你必须添加或使用不依赖 OTP 功能的其他身份验证器。
