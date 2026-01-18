## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">Ports - 特殊内容</span>

---

- <a href="#SharedLibs">共享库</a>
- <a href="#Autoconf">GNU autoconf</a>
- <a href="#Config">配置文件</a>
- <a href="#Audio">音频应用程序</a>
- <a href="#Mandoc">手册页</a>
- <a href="#RcScripts">rc.d(8) 脚本</a>

---

<h2 id="SharedLibs">共享库</h2>

<h3 id="UnderstandingSharedLibraryNumberRules">理解共享库编号规则</h3>

由于各种原因，共享库有点棘手。你必须理解库命名方案：`libfoo.so.major.minor`。

当你链接程序时，链接器 ld 将该信息嵌入到创建的二进制文件中。你可以使用 <a href="https://man.openbsd.org/ldd">ldd(1)</a> 查看它。稍后，当你运行该程序时，动态链接器 <a href="https://man.openbsd.org/ld.so">ld.so(1)</a> 使用该信息来查找正确的动态库：

- 需要具有完全相同主版本号 (major number) 的库。
- 需要具有相等或更高次版本号 (minor number) 的库。

因此，这意味着 **所有** 具有相同主版本号和相等或更高次版本号的库 **必须满足程序期望的二进制 API**。如果它们不满足，那么你的 port 就坏了。具体来说，当用户尝试更新他们的系统时，它会坏掉。

共享库的规则非常简单。

- 如果向库中添加了函数，你必须增加库的次版本号：需要这些函数的程序除了显式要求至少此版本外，无法要求它。
- 如果现有 API 发生更改，即如果任何函数签名被更改，或者如果有效的调用序列不再有效，如果类型以不兼容的方式更改，则库的主版本号 **必须增加**。
- 这包括删除旧函数。任何函数删除都应触发主版本号增加。
- 一个好的提示是比较

  ```shell
  $ nm -g oldlib.so.X.Y | cut -c10- | grep -e^T
  ```

  和

  ```shell
  $ nm -g newlib.so.X.Y | cut -c10- | grep -e^T
  ```

  的输出。这不会显示函数参数类型是否更改，但至少你会很快看到是否添加和/或删除了某些函数。

有时，库被写成几个文件，并且内部函数碰巧可见以便在这些文件之间进行通信。这些函数名称传统上以下划线开头，并且不是 API 本身的一部分。

<h3 id="TweakingPortsBuildsToAchieveTheRightNames">调整 Ports 构建以获得正确的名称</h3>

无论如何，相当多的 ports 需要调整才能正确构建共享库。请记住，构建共享库应该使用

```shell
$ cc -shared -fpic|-fPIC -o libfoo.so.4.5 obj1 obj2
```

尝试事后重命名库以调整版本号是行不通的：ELF 库使用一些额外的魔法来设置库内部名称，因此你必须第一次就使用正确的版本链接它。

另一方面，请记住你可以通过在 port 的 `Makefile` 中使用 `MAKE_FLAGS` 从命令行覆盖 `Makefile` 变量。在某些情况下，你正在移植的程序将有一个简单的变量，你可以通过在 MAKE_FLAGS 中设置库版本来覆盖它，例如 `MAKE_FLAGS= SO_VERSION=${LIBfoo_VERSION}`。在其他情况下，port 将需要修补以利用此类变量。

Ports 基础设施已经在基于 libtool 和基于 CMake 的 ports 中处理了这些细节。对于 libtool，默认使用基础操作系统的版本，但在某些情况下这还不够，可以设置 `USE_LIBTOOL=gnu`。CMake 通过使用 `cmake.port.mk` 模块处理：`MODULES += devel/cmake`。在这些情况下，大多数细节都会自动处理：

- `SHARED_LIBS` 被检查，版本号被自动替换。
- 共享库构建记录在 `${WRKBUILD}/shared_libs.log` 中，可以直接包含在 port 的 `Makefile` 中。

对于 CMake，初始构建通常会生成没有版本号的库（`lib/libXXX.so` 文件）。在这种情况下，将这些库的 `SHARED_LIBS` 行添加到 Makefile 中，设置为版本 0.0，清理并重建 port，当你重新生成 PLIST 时，你应该会看到它开始使用版本号。

<h3 id="TryPuttingAllUserVisibleLibrariesIntoUsrLocalLib">尝试将所有用户可见的库放入 /usr/local/lib</h3>

作为规则，要求用户将目录添加到他们的 <a href="https://man.openbsd.org/ldconfig">ldconfig(8)</a> 路径是一个非常糟糕的主意：所有直接链接到程序的共享库都应该出现在 `/usr/local/lib` 中。但是，完全可以使用指向实际库的符号链接。你应该了解库查找规则：

- 在构建时，<a href="https://man.openbsd.org/ld">ld(1)</a> 使用 `-L` 标志来设置查找库的路径。一旦找到符合其要求的库，它就会停止查找。
- 在运行时，<a href="https://man.openbsd.org/ld.so">ld.so(1)</a> 使用通过 <a href="https://man.openbsd.org/ldconfig">ldconfig(8)</a> 缓存的信息来查找所需的库。

因此，让我们假设你有两个 ports 提供给定库的两个主要版本，比如 `qt.1.45` 和 `qt.2.31`。由于两个 ports 可以同时安装，为了确保给定程序将链接到 `qt.1`，该库作为 `/usr/local/lib/qt/libqt.so.1.45` 提供，并且程序将使用以下命令链接

```shell
$ ld -o program program.o -L/usr/local/lib/qt -lqt
```

同样，链接到 `qt.2` 的程序将使用 `/usr/local/lib/qt2/libqt.so.2.31` 文件，并使用

```shell
$ ld -o program program.o -L/usr/local/lib/qt2 -lqt
```

为了在运行时解决这些库，提供了一个名为 `/usr/local/lib/libqt.so.1.45` 的链接和一个名为 `/usr/local/lib/libqt.so.2.31` 的链接。这足以满足 <a href="https://man.openbsd.org/ld.so">ld.so(1)</a>。

使用以下命令链接使用 `qt1` 的程序是错误的

```shell
$ ld -o program program.o -L/usr/local/lib -lqt
```

此代码假设未安装 `qt.2.31`，这是一个错误的假设。

这种技巧仅在极少数非常普遍的库的情况下才需要，其中必须提供主要版本之间的过渡期。一般来说，确保护库出现在 `/usr/local/lib` 中就足够了。

<h3 id="WritingLibraryDependenciesCorrectly">正确编写库依赖项</h3>

新的依赖代码确实需要完整的库依赖项。你必须使用 `make lib-depends-check` 或 `make port-lib-depends-check` 来验证 port 是否提到了它需要的所有库。你只需像这样将它们写入 `LIB_DEPENDS`/`WANTLIB`：

```text
LIB_DEPENDS += x11/gtk+
WANTLIB += gtk>=1.2 gdk>=1.2
```

在 `WANTLIB` 行上指定静态库也不是错误。`WANTLIB` 在包构建时被完全评估：生成的包将嵌入库依赖信息作为 ld.so 的行，其中包含用于构建的实际 major.minor 版本号，对于静态库则没有任何内容。

事实上，即使对于静态库也提供 `LIB_DEPENDS` 行是一个好主意。如果给定的依赖项从静态库变为共享库，这将简化 port 更新。

`WANTLIB` 行必须指定与 ld 使用的路径相同的路径。使用与上面相同的示例，标准的 `qt2` 依赖片段将说 `WANTLIB += lib/qt2/qt.=2`。这允许依赖检查代码在遇到同一库的多个版本时做正确的事情。

<h3 id="UpdatingPortsCorrectly">正确更新 Ports</h3>

因此，当你更新或添加涉及共享库的 port 时，必须正确处理一些细节。

- 确保共享库的 major.minor 版本号正确。
- 验证所有依赖于你的 port 的 ports。验证它们是否使用你的更改正确构建。通知相应的维护者更新，以便他们可以验证他们的 ports 仍然正确运行。
- 你可能需要调整依赖 ports 的 `WANTLIB` 和 `LIB_DEPENDS`。如果你引入了新的共享库，请注意需要转换为 `LIB_DEPENDS` 的 `BUILD_DEPENDS`。
- 每当你引入一个新 port 时，你应该验证你没有创建一个与现有库冲突的库：来自两个具有相同名称的 ports 的库是致命的，因为它们的版本编号方案没有机会匹配。你应该尝试与软件作者一起解决这种情况（例如，名为 libnet 的库绝对是命名不当的）。
- 查看 <a href="./02_guide.md#PortsUpdate">ports 更新指南</a>以获得更广泛的讨论。

<h2 id="Autoconf">GNU autoconf</h2>

Autoconf 是一个 GNU 工具，旨在帮助编写可移植程序。它通常与 automake（可移植 makefile）和 libtool（可移植共享库）一起使用。

这些工具并不那么好用，并且经常在将软件移植到 OpenBSD 时产生特定的挑战。

<h3 id="DetectingTheUseOfAutoconfInAPieceOfSoftware">检测软件中 autoconf 的使用</h3>

相当多的软件项目都有 configure 脚本，在大多数情况下，这些脚本是由 autoconf 生成的。此类脚本在顶部附近有一行说：

```text
# Generated automatically using autoconf version 2.13
```

或类似的内容。生成过程将在下一节中介绍。大多数情况下，autoconf ports 带有生成的脚本，以及生成这些脚本的源脚本。下一节涵盖了你只想运行生成的脚本而不修改它的简单情况。确保你也阅读了关于特洛伊木马的部分。

<h3 id="RunningAnAutoconfConfigureScript">运行 autoconf Configure 脚本</h3>

此脚本通常在 ports 构建的 configure 阶段运行。要调用 configure 脚本，只需设置 `CONFIGURE_STYLE=gnu`，这将自动调用 `${WRKSRC}/configure`。

如果你的 configure 脚本位于其他位置，只需将 `CONFIGURE_SCRIPT` 设置为正确的值。

Configure 脚本通常接受许多参数。Ports 树的默认处理只会将 `--prefix` 和 `--sysconfdir` 传递给这些脚本。非常旧的 configure 脚本不理解 `--sysconfdir`；在这种情况下，你可以设置 `CONFIGURE_STYLE=gnu old`。

同样，某些 ports 不知道 `DESTDIR`。这些 ports 通常会接受设置 `prefix=${DESTDIR}/usr/local` 而没有任何问题，这可以通过 `CONFIGURE_STYLE=gnu dest` 完成。

使用 autoconf 和 automake 的 Ports 将具有特定格式的 `Makefile`，该格式以几个标准位置开头：

- `bindir`：二进制文件的位置
- `sysconfdir`：配置的位置
- `includedir`：包含目录的位置

如果 configure 脚本不允许你覆盖这些，你仍然可以在稍后的构建或 `fake` 阶段这样做。当然，这假设对此类目录的唯一引用是在生成的 `Makefile` 中。

例如，一个巧妙的技巧是在 `fake` 阶段将 `sysconfdir` 切换到 `${PREFIX}/share/example/pkgname`，以获取要打包的默认配置文件（因为软件包通常不将文件存储在 `/etc` 下）。

完全使用 autoconf 和 automake 的 Ports 可能支持在不同目录下构建：尝试设置 `SEPARATE_BUILD=flavored` 看看是否有效。这将允许你在不擦除源树的情况下擦除构建树，方法是给你单独的 `${WRKSRC}` 和 `${WRKBUILD}` 位置。在少数情况下，单独构建可能需要使用 gmake，而 port 的其余部分对 bsd-make 很满意，在这种情况下这不值得。

Automake 将生成一些规则，如果任何内容发生更改，则重建所有生成的脚本。这经常妨碍 OpenBSD 特定的补丁。因此，只要 `CONFIGURE_STYLE` 对应于 autoconf 使用，`post-patch` 就会按特定顺序 touch 各种文件，以便稍后不会触发 automake 依赖项。依赖项列表以 <a href="https://man.openbsd.org/tsort">tsort(1)</a> 顺序在 `REORDER_DEPENDENCIES` 中提到的文件中给出（默认为 `${PORTSDIR}/infrastructure/mk/automake.dep`）。

<h3 id="TheMechanicsOfConfigureChecks">Configure 检查的机制</h3>

Configure 脚本首先运行一个名为 `config.guess` 的固定脚本，该脚本将确定 configure 正在哪个系统上运行。`config.guess` 不会因 port 而异，是一个固定脚本，因此 OpenBSD ports 树将其替换为一个知道某些特定 OpenBSD 架构的固定版本。由于大多数软件包都带有捆绑的 `config.guess`，而且其中一些非常旧，这是一个必要的步骤。如果软件包包含多个 `config.guess`，你可以通过将 `MODGNU_CONFIG_GUESS_DIRS` 设置为要处理的目录的完整列表来覆盖它们。

由 autoconf 生成的 configure 脚本随后通过查找编译器并通过它运行简单的测试程序来简单地检查现有系统上的所有功能。由于其中一些测试相当漫长，ports 树为 configure 准备了一个 `CONFIG_SITE=config.site` 文件。Configure 将在运行测试之前首先查看该文件的内容。一些 configure 脚本可能有错误，导致它们在存在 `config.site` 的情况下无法正确运行。将 `CONFIG_SITE` 设置为空将消除此类问题。

大多数 configure 会自动检测相当多的条件。查看 configure 的选项、configure 的输出以及生成的 `config.log` 文件非常重要：这些将告诉你找到了哪些选项，以及未找到哪些选项。这将允许你找出 configure 何时未找到已安装的软件包。

这也将告诉你 configure 会找到哪些可选软件包。在 ports 树中，这些被称为隐藏依赖项。这是一件坏事：隐藏依赖项是如果安装了 configure 就会拾取的一些额外软件包。然后它将继续构建一个变异的软件包。在某些情况下，构建会因为 OpenBSD 的特性而失败。在某些情况下，软件包创建将失败，因为某些文件将具有不同的名称。在某些情况下，生成的软件包将不正确，因为它无法记录对可选软件包的任何依赖关系。所以查看 configure 的输出是 ports 维护者最重要的职责之一。注意级联测试：检测给定功能可能会导致 configure 脚本尝试并找到某些依赖功能，因此除非触发第一个功能，否则你不会在 configure 输出中看到第二个功能。

如果发现一些隐藏依赖项，应该采取一些行动。最简单的行动是安装可选软件包，看看 configure 会做什么。如果它检测到该软件包，你可以禁用检测（通过使用 configure 选项、环境变量或修补 configure 脚本），或者验证构建是否顺利并将依赖项添加到依赖软件包列表中。更好的选择是找出合理的默认依赖项集，然后添加一些 flavors 以涵盖其他常见功能。

<h3 id="ReGeneratingConfigureScripts">重新生成 Configure 脚本</h3>

Configure 脚本通常从 `configure.in` 文件生成（较新版本的 autoconf 使用 `configure.ac` 文件）。标准定义库通常在 `aclocal.m4` 中可用。

在大多数情况下，直接修补 configure 是个坏主意。最好修补 `configure.in` 文件并让 ports 树调用 autoconf。优秀的 porters 将努力编写可以反馈给软件作者的 `configure.in` 更改。

不同版本的 autoconf 将产生不同的 configure 脚本。`autoconf-2.13` 很特别：它使用了相当长的一段时间，并且有广泛使用的 `autoconf-2.13` 变体版本（实际上是较新 autoconf 的测试版）。因此，使用 `autoconf-2.13` 通常不会产生完全相同的 configure 脚本。

由于同时拥有多个 autoconf 版本很有用，ports 树中实际可用的 autoconf 脚本是名为 metaauto 的 port 的一部分。实际调用哪个 autoconf 脚本通过环境变量 `AUTOCONF_VERSION` 控制。如果你设置 `CONFIGURE_STYLE=autoconf` 以及设置 `AUTOCONF_VERSION`，就会发生调用 autoconf。在大多数情况下，识别用于生成分发的 configure 脚本的 autoconf 版本（通常在阅读脚本时很明显）并自己使用相同的版本。

Autoconf 依赖于标准 unix 预处理器 <a href="https://man.openbsd.org/m4">m4(1)</a>。通常，autoconf 依赖于 GNU 版本 m4 (gm4) 的某些功能。幸运的是，OpenBSD 的 m4 也有足够的功能来运行 autoconf，只需要用 `-g` 调用它来处理 autoconf。极少数情况下，使用 OpenBSD 的 m4 运行 autoconf 会产生虚假的 configure 脚本。OpenBSD 开发人员将修复此类问题。

<h3 id="TrojanHorses">特洛伊木马</h3>

Configure 脚本是巨大的生成文件。它们是特洛伊木马的理想藏身之处，这在过去确实发生过。这是在树中拥有大多数 autoconf 版本的主要原因：优秀的 porter 应该检查生成的 configure 脚本是否与 ports 树 autoconf 构建的内容匹配。

<h3 id="InteractionWithOtherPrograms">与其他程序的交互</h3>

Autoheader 是另一个与 autoconf 相关的程序，通常运行它来创建 `config.h.in` 文件。设置 `CONFIGURE_STYLE=autoconf` 也会运行 autoheader。一些 ports 不使用 autoheader。设置 `CONFIGURE_STYLE=autoconf no-autoheader` 将解决该问题。

libtool 在 `configure.in` 中有一些特定的钩子。通常有一个 `libtool.m4` 脚本与之配合。让 libtool 做正确的事情超出了本文档的范围。

<h2 id="Config">配置文件</h2>

软件包应该只在 `${PREFIX}` 下安装文件，默认为 `/usr/local`。另一方面，OpenBSD 的策略是将大多数配置文件安装在 `${SYSCONFDIR}` 下，默认为 `/etc`。

*注意，二进制软件包硬编码 `${PREFIX}` 和 `${SYSCONFDIR}` 是完全可以接受的：`PREFIX` 和 `SYSCONFDIR` 主要是影响软件包构建的用户设置。*

<h3 id="SampleExplained"><code>@sample</code> 解释</h3>

装箱单包含特定的 `@sample` 机制来处理配置文件：

- 在伪安装期间，port 应安装一个示例配置文件，例如 `${PREFIX}/share/examples/PKGNAME/foo.rc`。
- 装箱单应在示例配置文件正下方包含一个 `@sample ${SYSCONFDIR}/foo.rc`。
- 在安装过程中，默认示例配置文件将被复制到配置文件应该存在的实际位置。
- 在更新和卸载期间，现有配置文件将与默认示例文件进行比较。如果它们不同，软件包工具将通知用户并让他自己执行更新/删除。如果它们相同，软件包工具知道它们可以继续更新/删除配置文件而无需任何进一步的预防措施。

<h3 id="MoreSampleSpecificities">更多 <code>@sample</code> 特性</h3>

与装箱单中的其他文件不同，`@sample` 条目可以具有绝对路径名。

一些大型软件包也需要它们自己的配置目录，`@sample ${SYSCONFDIR}/directory/` 将处理此问题。

使用 `@sample directory/` 创建不包含任何配置文件的 port 特定目录是完全好的风格。`@sample` 正确解释正确的 `@mode`、`@owner`、`@group` 注释。这可能有点麻烦，因为你经常需要在默认模式和配置文件特定模式之间来回切换。

<h3 id="SpecialTricks">特殊技巧</h3>

`make update-plist` 知道如何复制 `@sample` 注释，但它不知道如何创建它们，因此必须首先编写它们。

注意配置文件和示例配置文件之间的区别：port 必须配置为在 `${SYSCONFDIR}` 下查找其文件。只有伪安装阶段必须将东西放在 `${PREFIX}/share/examples` 下。处理此问题的一种简单方法是在 `post-install` 中复制文件。

一个经常有效的巧妙技巧是查看程序的 `Makefile`，并通过使用特定的 `FAKE_FLAGS` 在伪安装阶段覆盖配置目录，例如：

```text
FAKE_FLAGS=	DESTDIR=${WRKINST} \
		sysconfdir=${WRKINST}${TRUEPREFIX}/share/examples/PKGNAME
```

你只需要注意那些在安装阶段将配置目录写入特定文件的程序。

<h3 id="Examples">示例</h3>

- `security/integrit` port 使用带有几个文件的配置目录。其装箱单如下所示：

  ```text
  @comment $OpenBSD$
  @bin bin/i-ls
  @info info/integrit.info
  @man man/man1/i-ls.1
  @man man/man1/i-viewdb.1
  @man man/man1/integrit.1
  @bin sbin/i-viewdb
  @bin sbin/integrit
  share/doc/integrit/
  share/doc/integrit/README
  share/doc/integrit/crontab
  share/doc/integrit/install_db
  share/doc/integrit/integrit_check
  share/doc/integrit/viewreport
  share/examples/integrit/
  @sample ${SYSCONFDIR}/integrit/
  share/examples/integrit/root.conf
  @sample ${SYSCONFDIR}/integrit/root.conf
  share/examples/integrit/src.conf
  @sample ${SYSCONFDIR}/integrit/src.conf
  share/examples/integrit/usr.conf
  @sample ${SYSCONFDIR}/integrit/usr.conf
  ```

- `mail/dovecot` port 使用 `@sample dir/` 创建私有目录。

  ```text
  ...
  @extraunexec rm -rf /var/dovecot/*
  @sample ${SYSCONFDIR}/dovecot/
  @sample ${SYSCONFDIR}/dovecot/conf.d/
  @mode 755
  @sample /var/dovecot/
  @mode 750
  @group _dovenull
  @sample /var/dovecot/login/
  ```

- `sysutils/nut` port 为其配置文件使用特定的所有者。

  ```text
  @comment $OpenBSD$
  @conflict upsd-*
  @newuser ${NUT_USER}:${NUT_ID}:daemon:UPS User:/var/empty:/sbin/nologin
  ...
  share/examples/nut/
  @sample ${SYSCONFDIR}/nut/
  @owner ${NUT_USER}
  share/examples/nut/ups.conf
  @sample ${SYSCONFDIR}/nut/ups.conf
  share/examples/nut/upsd.conf
  @mode 600
  @sample ${SYSCONFDIR}/nut/upsd.conf
  @mode
  share/examples/nut/upsd.users
  @mode 600
  @sample ${SYSCONFDIR}/nut/upsd.users
  @mode
  share/examples/nut/upsmon.conf
  @mode 600
  @sample ${SYSCONFDIR}/nut/upsmon.conf
  @mode
  share/examples/nut/upssched.conf
  @sample ${SYSCONFDIR}/nut/upssched.conf
  @mode 700
  @sample /var/db/nut/
  @mode
  @owner
  share/ups/
  share/ups/cmdvartab
  share/ups/driver.list
  ```

<h2 id="Audio">音频应用程序</h2>

本文档目前仅处理采样声音问题。欢迎提供有关合成器和波形表的贡献。

音频应用程序往往很难移植，因为这是一个接口完全没有标准化的领域，尽管操作系统之间的方法差异不大。

<h3 id="libsndio">libsndio</h3>

OpenBSD 有自己的音频层，由 sndio 库提供，记录在 <a href="https://man.openbsd.org/sio_open">sio_open(3)</a> 中。在它合并到此页面之前，你可以在指南<a href="https://sndio.org/tips.html">关于编写和移植音频代码的提示</a>中找到有关为此 API 编程的更多信息。sndio 允许用户进程以统一的方式访问 <a href="https://man.openbsd.org/audio.4">audio(4)</a> 硬件和 <a href="https://man.openbsd.org/sndiod">sndiod(8)</a> 音频服务器。它支持全双工操作，并且当与 <a href="https://man.openbsd.org/sndiod">sndiod(8)</a> 服务器一起使用时，它支持动态重采样和格式转换。

<h3 id="HardwareIndependence">硬件独立性</h3>

**你不应该假设关于所使用的音频硬件的任何事情。**
错误的代码是仅根据 8 或 16 位检查 `a_info.play.precision` 字段，并根据 soundblaster 行为假设无符号或有符号样本的代码。你应该显式检查样本类型，并据此编写代码。简单示例：

```c
AUDIO_INIT_INFO(&a_info);
a_info.play.encoding = AUDIO_ENCODING_SLINEAR;
a_info.play.precision = 16;
a_info.play.sample_rate = 22050;
error = ioctl(audio, AUDIO_SETINFO, &a_info);
if (error)
    /* deal with it */
error = ioctl(audio, AUDIO_GETINFO, &a_info);
switch(a_info.play.encoding)
    {
case AUDIO_ENCODING_ULINEAR_LE:
case AUDIO_ENCODING_ULINEAR_BE:
    if (a_info.play.precision == 8)
        /* ... */
    else
        /* ... */
    break;
case ...

default:
    /* don't forget to deal with what you don't know !!! For instance, */
    fprintf(stderr,
        "Unsupported audio format (%d), ask ports@ about that\n",
            a_info.play.encoding);

    }
    /* now don't forget to check what sampling frequency you actually got */
```

这是处理大多数问题的最小代码片段。

<h3 id="16bitFormatsAndEndianness">16 位格式和字节序</h3>

在正常使用中，你只要求一种编码类型（例如，`AUDIO_ENCODING_SLINEAR`），然后你检索带有字节序的编码（例如，`AUDIO_ENCODING_SLINEAR_LE`）。考虑到声卡不必使用与你的平台相同的字节序，你应该准备好处理这个问题。最简单的方法可能是准备一个完整的音频缓冲区，并在需要更改字节序时使用 <a href="https://man.openbsd.org/swab">swab(3)</a>。处理外部样本通常相当于：

1. 解析样本格式
2. 获取样本
3. 如果不是你的本机格式，则交换字节序
4. 计算要输出到缓冲区的内容
5. 如果声卡不是你的本机格式，则交换字节序
6. 播放缓冲区

显然，如果你只是播放恰好是声卡本机格式的声音样本，你可能能够删除步骤 3 和 5。

<h3 id="AudioQuality">音频质量</h3>

硬件可能有一些奇怪的限制，例如无法在立体声中超过 22050 Hz，但在单声道中高达 44100。在这种情况下，你应该给用户一个陈述其偏好的机会，然后尽力提供尽可能好的性能。例如，因为你要输出立体声而将频率限制为 22050 Hz 是愚蠢的。如果用户没有将立体声声音系统连接到他的音频卡输出怎么办？

将类似 soundblaster 的限制硬编码到你的程序中也是愚蠢的。你应该意识到这些，但一定要尝试突破 22050 Hz/立体声障碍并检查结果。

<h4 id="SamplingFrequency">采样频率</h4>

你绝对应该检查你的卡返回给你的采样频率。5% 的差异已经相当于半音，有些人的听力比这更准确，尽管我们大多数人不会注意到任何事情。你的应用程序应该能够动态执行重采样，可能是天真地，或者是通过香农重采样公式的迂回应用（如果可以的话）。

<h4 id="DynamicRange">动态范围</h4>

样本并不总是使用它们可以使用的全部值范围。首先，以低增益录制的样本在机器上听起来不会很响，迫使用户调高音量。其次，在音频隔离不好的机器上，低声音输出意味着你主要听到的是机器的心跳声，而不是你预期的声音。最后，从 16 位到 8 位的愚蠢转换可能会让你只剩下 4 位可用音频，这会导致极其糟糕的质量。

如果可能的话，最好的解决方案可能是提前扫描你要播放的整个流，并对其进行缩放以使其适合完整的动态范围。如果你负担不起，但你可以设法对你要播放的内容进行一点前瞻，你可以动态调整音量增强，你只需要确保增强因子相对于你要播放的声音保持在低频，并且你绝对**没有溢出**——这些听起来总是比你试图实现的改进糟糕得多。
由于声音音量感知是对数的，使用算术移位通常就足够了。如果你的数据是有符号的，你应该显式地将移位编码为除法，因为 C `>>` 运算符在有符号数据上不可移植。

如果所有其他方法都失败了，你至少应该尝试为用户提供音量缩放选项。

<h3 id="AudioPerformance">音频性能</h3>

低端应用程序通常没有什么可担心的。

不要忘记运行基准测试。理论优化只是理论上的。应该收集一些硬数据来检查什么是显着的改进，什么不是。

对于高性能音频应用程序，例如 mpegI-layer3，应考虑以下几点：

- 音频接口确实为你提供了自然的硬件块大小。将倍数用于输出缓冲区至关重要。请记住，`write` 作为系统调用，与内部音频处理相比会产生很高的成本。
- 处理音频时，带宽是一个非常重要的因素。优化音频播放器的一种有用方法是将其视为解压缩器。通常，保留压缩数据的时间越长越好。做很少处理的非常短的循环通常是个坏主意。通常最好将所有处理合并到一个循环中。
- 某些格式确实比其他格式产生更多的开销。应使用 `AUDIO_GETENC` `ioctl` 检索音频设备提供的所有格式。特别注意 `AUDIO_ENCODINGFLAG_EMULATED` 标志。如果你的应用程序已经能够输出各种奇怪的格式，并且为此进行了合理的优化，请不惜一切代价尝试使用本机格式。另一方面，音频设备中存在的仿真代码可以被认为是相当优化的，所以不要用快速破解的代码替换它。

为了获得最佳结果，你可能必须遵循的一个模型是首先编译一个小的测试程序，询问可用的特定音频硬件，然后继续配置你的程序，以便它以最佳方式处理此硬件。你可以合理地期望想要良好音频性能的人在更换硬件时重新编译你的 port，前提是这会有所不同。

<h3 id="RealTimeOrSynchronized">实时或同步</h3>

考虑到 OpenBSD 不是实时的，你可能仍然希望编写主要是实时的音频应用程序，例如游戏。在这种情况下，你将不得不降低块大小，以便声音效果不会与当前游戏不同步。这样做的问题是音频设备可能会因为数据不足而产生可怕的结果。

如果你只是想让音频与某些图形输出同步，但你的程序的行为是可预测的，那么同步更容易实现。你只需播放音频样本，并使用 `AUDIO_GETOOFFS` 询问音频设备当前正在播放什么，然后使用该信息对图形进行后期同步。只要你询问得足够频繁（比如每十分之一秒），并且只要你有足够的马力来运行你的应用程序，你就可以通过这种方式获得非常好的同步。你可能必须通过恒定偏移量调整数字，因为音频报告的内容、当前正在播放的内容以及 X Window 显示内容所需的时间之间存在一些滞后。

<h3 id="ContributingCodeBack">回馈代码</h3>

在音频应用程序的情况下，与原始程序的作者合作非常重要。例如，如果他们的代码仅适用于 soundblaster 卡，那么他们很有可能很快就要应对其他技术。

**如果你不把你的评论发送给作者，你的工作将是无用的**。

作者也可能已经注意到你当前正在处理的任何问题，并且正在他当前的开发树中解决它们。如果你正在编写的补丁超过几行，合作几乎肯定是一个非常好的主意。

<h2 id="Mandoc">手册页</h2>

本节提供了关于如何在 ports 中处理 groff 与 <a href="https://man.openbsd.org/mandoc">mandoc(1)</a> 问题的指南，以及如何处理非英语手册页。

<h3 id="ShouldICheckAnything">我应该检查什么吗？</h3>

当创建新 port 或更新现有 port 时，请检查 port 是否可以使用 mandoc 格式化其手册。下面描述的自动和手动检查都是必需的。这可能会使手册对 port 的用户更有用，并且会减少 port 的构建时间。

如果新 port 或未标记为 `USE_GROFF` 的现有 port 无法使用 mandoc 工作，请向 schwarze@ 报告，他可能会修复 mandoc。

<h3 id="WhichToolsDoINeed">我需要哪些工具？</h3>

除了基础系统中包含的 <a href="https://man.openbsd.org/mandoc.1">mandoc(1)</a> 实用程序外，不需要任何工具。在非常不寻常的情况下，如果你怀疑 mandoc 的最近更改对 port 很重要，你可以轻松更新 mandoc，甚至无需更新系统的其余部分：

```shell
$ cd /usr/src/usr.bin/mandoc/
$ cvs -q up -Pd
$ make cleandir
$ make obj
$ make
$ doas make install
```

或者，你也可以获取 <a href="https://mandoc.bsd.lv/cgi-bin/cvsweb/gmdiff">gmdiff</a> 实用程序脚本的副本，它有助于比较 groff 和 mandoc 的输出。gmdiff 脚本不是严格要求的，手动进行必要的检查也是完全可以接受的。

<h3 id="HowDoIReportTheResults">我如何报告结果？</h3>

以下段落要求在某些特定情况下向 mandoc 维护者发送报告。在发送此类报告之前，请务必勾选以下清单：

1. 将有问题的 <a href="https://man.openbsd.org/mdoc.7">mdoc(7)</a> 或 <a href="https://man.openbsd.org/man.7">man(7)</a> 源文件附加到邮件中。这可以是分发 tar 包中包含的文件，也可以是构建过程中生成的文件。如果多个文件表现出问题，请选择一个显示所有问题的文件。如果不同的文件表现出你想报告的不同问题，请附加尽可能多的文件。重点是为 mandoc 维护者节省下载分发 tar 包、搜索源文件、有时甚至在能够开始构建之前安装软件的工作，而你手头正好有这些信息。
2. 简要描述你想报告的所有问题，以及可以在哪个文件的哪里看到它们。过去我们不止一次花时间想知道报告者的确切意思是什么。
3. 如果你的报告与 mandoc 实用程序打印的错误或警告有关，请将 `mandoc -Tlint`（或当警告无关紧要时 `mandoc -Tlint -Werror`）的输出复制到你的邮件正文中。通常，这很容易重现，但也发生过无法重现的情况，造成不必要的困惑。
4. 如果你谈论的 port 版本尚未提交，请附上构建未提交 port 所需的内容：更新时的针对 -current 的 diff，或者是全新 port 时的 port 目录 tar 包。很多时候，源文件足以识别问题；然而，在那些不足以识别的情况下，来回发送邮件或搜索邮件列表存档只是为了获得所需的额外信息是浪费时间。
5. 发送邮件给 schwarze@。除非你是 port 的维护者，否则请抄送给他或她。除非你是 OpenBSD 开发人员，如果你经常与一位正在提交你的 ports 并且你知道他对这个 port 感兴趣的开发人员合作，抄送给他或她也可能有用。

<h3 id="HowDoIDoAutomaticChecking">我如何进行自动检查？</h3>

要进行自动部分的检查，请对 port 中包含的所有 <a href="https://man.openbsd.org/mdoc.7">mdoc(7)</a> 和 <a href="https://man.openbsd.org/man.7">man(7)</a> 手册源文件运行以下命令：

```shell
$ mandoc -Tlint -Werror *
```

如果你收到任何 `UNSUPPORTED` 消息，手册页的相应位置需要仔细审查。很可能该页面使用 mandoc 格式化错误，并且 port 需要 `USE_GROFF`。如果你确信所有与不支持的功能相关的格式错误都是次要的并且不妨碍读者，你可以删除 `USE_GROFF`；但在有疑问的情况下，当有 `UNSUPPORTED` 消息时保留 `USE_GROFF`。

如果有任何 `ERROR` 消息，也应该简要查看一下。在不寻常的情况下，如果它们与 mandoc 发生的但 groff 未发生的格式错误有关，则应报告；mandoc 维护者可能会选择让 mandoc 在其他情况下发出 `UNSUPPORTED` 消息或修复格式。

如果手册页使用 groff 看起来很好，切勿为了摆脱 mandoc 错误而修补它们。这仅仅是一个无用的项目，对任何人都没帮助：它既无助于改进上游手册，也无助于改进 mandoc。

<h3 id="HowDoIDoManualChecking">我如何进行手动检查？</h3>

如果没有错误或错误与 mandoc 的严重格式错误无关，请继续进行检查的手动部分。查看由 mandoc 格式化的手册。它们看起来好吗？如果是，你不需要 `USE_GROFF`，也不需要报告任何事情。

如果没有错误，但 mandoc 输出有严重问题，即相关信息丢失或部分输出乱码，请务必报告你的发现，即使你碰巧知道这是由于 mandoc 的已知问题造成的。我们确实想知道哪些问题在实践中会导致严重问题，以便我们可以首先解决最紧迫的问题。

如果 mandoc 输出有严重问题，而 groff 输出看起来也很糟糕，那么手册可能在上游就坏了。在这种情况下，你在移植损坏的软件时有通常的选项：放弃 port、忽略问题、向上游报告和/或通过补丁修复错误。如果你需要后者的帮助，请与 schwarze@ 交谈。

如果没有错误，但 mandoc 输出有小问题，并不真正妨碍用户阅读手册，欢迎你也报告这些问题。在这种情况下，更欢迎你先检查 mandoc <a href="https://mandoc.bsd.lv/cgi-bin/cvsweb/TODO">TODO</a> 列表，以避免一次又一次地报告相同的小问题 - 但在有疑问的情况下，报告重复总比让问题被忽视要好。

如果只有极少数错误，特别是如果你得到的印象是 mandoc 输出无论如何都很好，你通常不需要 `USE_GROFF=Yes`。如有疑问，请寻求建议。此类问题通常有助于改进 mandoc 错误报告，特别是识别和删除虚假的 mandoc 错误消息。

为了加快手动检查，特别是如果你经常在 OpenBSD ports 上进行 mandoc 检查，并为了减少忽视问题的风险，请考虑使用 <a href="https://mandoc.bsd.lv/cgi-bin/cvsweb/gmdiff">gmdiff</a> 实用程序脚本。它接受任意数量的手册源文件名作为参数，依次对所有文件运行 groff 和 mandoc，并比较两个程序的输出。但是，请记住，你仍在进行手动检查，最终目标是判断 mandoc 输出的质量：即使你在使用 gmdiff 脚本帮助你的工作，上述所有要点仍然适用。另请注意，gmdiff 通常会发现两个程序之间的细微格式差异，特别是在空格方面。如果 mandoc 输出看起来不错，即使它与 groff 输出略有不同，也不需要 `USE_GROFF`。

为了便于使用，可以从 mk.conf 中的自定义目标调用 gmdiff：

```text
gmdiff:
	@make fake; cd ${WRKINST}${TRUEPREFIX}; find man -type f -path 'man/man*' -print0 | xargs -0r gmdiff | less
```

<h3 id="WhatAboutWarnings">关于警告呢？</h3>

你可能想知道 mandoc 警告，相对于 mandoc 错误。简而言之，区别在于错误可能会严重影响输出的可用性，而警告最多可能导致轻微的格式故障，如果有的话。如果 mandoc 警告似乎与严重乱码的输出有关，那可能是 mandoc 中的错误，应始终报告。

也就是说，很明显警告与决定是否为给定 port 使用 mandoc 无关。它们是给手册作者的，以帮助提高手册质量，而不是给 porters 的。

<h3 id="HowCanIHelpUpstream">我如何帮助上游？</h3>

如果你是 port 的上游开发人员之一，或者知道他们关心手册的高质量并乐意接受补丁，那么使用 `mandoc -Tlint` 来识别潜在的格式问题并生成要提交给上游的补丁可能是有意义的。通常，无需将此类补丁放入 ports 树中。

与任何类型的 linting 一样，在更改 <a href="https://man.openbsd.org/mdoc.7">mdoc(7)</a> 或 <a href="https://man.openbsd.org/man.7">man(7)</a> 源代码或发送补丁之前，首先确保你正在追踪手册中的实际问题。mandoc 实用程序并不完美。它可能会产生虚假警告。我们正在努力修复这个问题，但总会有改进的空间。如有疑问，请报告问题并寻求建议。

<h3 id="NonEnglishManualPages">非英语手册页</h3>

以下是经验法则，并非一成不变的法律。如果你发现你的 port 有特殊需求，你可以把它们放在一边；目标是使 port 对用户有用。如果你这样做了，请考虑告诉 schwarze@，也许我们可以从你的 port 中学到一些东西。

1. 如果上游提供非英语手册页，如果可能的话，在不费周折的情况下安装它们，除非有特定的理由不这样做。“它们已过时”不是排除它们的好理由。
2. 永远不要安装除 UTF-8 以外的任何编码。如果上游提供 UTF-8，太好了。否则，设置 `BUILD_DEPENDS = converters/libiconv` 并在 `post-build` 目标中使用 iconv(1)。
3. 如果 mandoc 可以处理，你可以用与英语手册完全相同的方式检查，只需将 UTF-8 源代码安装到 `man/language/manN/*.N` 并且不要 `USE_GROFF`。
4. 如果 mandoc 无法处理，正确的操作顺序是 iconv(1) -t UTF-8，然后 preconv(1)，然后 nroff(1)，绝不是其他方式。
5. 如果可能，安装到 `man/language/manN`，不带任何 "_" 或 "@" 字符。切勿在路径名中包含编码，并确保 `/language/` 部分永远不包含 "."（点）。
6. 作为例外，使用 `zh_CN` 和 `zh_TW` 而不仅仅是 `zh`。此外，保留上游的 `pt` 和 `pt_BR`，如果可用，两者都安装。

如果遵循上述规定，人们可以在不更改默认配置的任何部分的情况下执行以下操作：

```shell
$ doas pkg_add mc
$ export LC_CTYPE=en_US.UTF-8
$ alias ruman="man -m /usr/local/man/ru"
$ ruman mc
```

<h2 id="RcScripts">rc.d(8) 脚本</h2>

本节旨在提供有关编写和安装 <a href="https://man.openbsd.org/rc.d">rc.d(8)</a> 脚本的一些信息。

安装守护进程的 Ports 极大地受益于拥有 <a href="https://man.openbsd.org/rc.d">rc.d(8)</a> 脚本。它允许用户轻松检查守护进程是否正在运行，并提供一种简单且一致的方式来启动和停止它。

<h3 id="WritingRcD8Scripts">编写 rc.d(8) 脚本</h3>

由于 <a href="https://man.openbsd.org/rc.subr">rc.subr(8)</a> 系统简洁的设计，编写 <a href="https://man.openbsd.org/rc.d">rc.d(8)</a> 脚本既直接又简单。虽然有几件事需要考虑。

1. 脚本必须放置在 `${PKGDIR}` 中，扩展名为 `.rc`，如 `mpd.rc`。这将允许软件包工具拾取它。
2. 务必测试脚本的所有功能，特别是 *reload* 功能。
3. 编写守护进程路径时使用 `${TRUEPREFIX}`。

<h3 id="ExampleScript">示例脚本</h3>

下面是一个典型脚本的示例。

```shell
#!/bin/ksh

daemon="${TRUEPREFIX}/sbin/munin-node"

. /etc/rc.d/rc.subr

pexp="/usr/bin/perl -wT ${daemon}${daemon_flags:+ ${daemon_flags}}"

rc_pre() {
        install -d -o _munin /var/run/munin
}

rc_cmd $1
```

在 ports 树的 templates 目录中也可以找到<a href="https://cvsweb.openbsd.org/ports/infrastructure/templates/rc.template?rev=HEAD">模板脚本</a>。
