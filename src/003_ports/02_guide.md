## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">Ports - 移植指南</span>

---

- <a href="#Overview">概览</a>
- <a href="#PortsChecklist">移植检查清单</a>
- <a href="#PortsComplex">处理复杂情况</a>
    - <a href="#Know">了解软件</a>
    - <a href="#Figure">找出重要选项</a>
    - <a href="#Ideal">理想情况：MULTI_PACKAGES 和 PSEUDO_FLAVORS</a>
    - <a href="#Inter">子包之间的相互依赖</a>
    - <a href="#True">真正的 FLAVORS 和 PKGNAMES</a>
- <a href="#PortsUpdate">更新 Ports</a>
    - <a href="#UpdateChecklist">更新检查清单</a>
    - <a href="#CommitUpdates">提交 Port 更新</a>
- <a href="#PortsPolicy">OpenBSD 移植策略</a>
- <a href="#PortsSecurity">安全建议</a>
- <a href="#PortsGeneric">通用移植提示</a>
- <a href="#PortsOther">其他有用的提示</a>
- <a href="#PortsAvail">附加信息</a>

---

<h2 id="Overview">概览</h2>

所以你刚刚在你的 OpenBSD 机器上编译了你最喜欢的软件包，你想通过把它变成一个标准 port 来分享你的努力。该怎么办？

最重要的事情是**沟通**。在 <a href="mailto:ports@openbsd.org">ports@openbsd.org</a> 上询问是否有人正在开发同一个 port。*告诉原始软件作者这件事*，包括你可能发现的问题。如果许可信息看起来不正确，告诉他们。如果你不得不费尽周折才能使 port 构建成功，告诉他们可以修复什么。如果他们只在 Linux 上开发并且觉得可以忽略 Unix 世界的其他部分，试着让他们改变观点。

沟通决定了一个成功的 port 和一个慢慢被所有人抛弃的 port 之间的区别。

首先，查看本页上的移植信息。测试，再测试，最后再测试！

OpenBSD 完全支持更新。这意味着必须考虑<a href="#PortsUpdate">一些问题</a>。

提交 port。创建 port 目录的 gzip 压缩 tar 包。然后你可以把它放在公共 HTTP 服务器上，将 URL 发送到 <a href="mailto:ports@openbsd.org">ports@openbsd.org</a>，或者将 MIME 编码的 port 发送到同一地址。

移植一些新软件需要时间。随着时间的推移维护它更难。移植软件并在之后让其他人处理它是完全可以的。帮助其他人更新和维护其他 ports 也是可以的，只要你们沟通以避免做同样的事情两次。

在 OpenBSD 文化中，`MAINTAINER`（维护者）身份不是一种地位象征，而是一种责任。我们有 CVS 和注释来归功于做这项工作的人。port `MAINTAINER` 是另一回事：一个对 port 的工作承担责任的人，并且愿意花一些时间确保它尽可能好地工作。

<h2 id="PortsChecklist">移植检查清单</h2>

下面的列表是一个有用的提醒，列出了要做的事情。这既不完全准确也不完美。将评论和问题直接发送到 <a href="mailto:ports@openbsd.org">ports@openbsd.org</a>。

1. 如果你想成为维护者，请订阅 <a href="mailto:ports@openbsd.org">ports@openbsd.org</a>。

    - 这是所有 ports 讨论发生的地方。
    - 阅读此列表很重要，因为许多公告都在那里发布。
    - 你会在那里找到很多精通移植的人。他们经常可以给你很好的建议或为你测试 ports。

2. 成为维护者意味着**不仅仅是**提交 ports。这也意味着尝试保持它们最新，并回答有关它们的问题。

3. 从 CVS 检出 ports 树的副本。你可以在 <a href="https://www.openbsd.org/anoncvs.html">AnonCVS 页面</a> 找到如何执行此操作的说明。作为 porter，你应该保持你的基础操作系统、ports 树和已安装的软件包最新。

4. 从 `/usr/ports/` 的一级子目录名称中，为你的 port 选择一个主要类别。在 `/usr/ports/<category>/` 或 `/usr/ports/mystuff/<category>/` 下创建一个新目录，并在那里创建基本的基础设施。从 `/usr/ports/infrastructure/templates/Makefile.template` 复制模板 `Makefile`。用你选择的主要类别填写 `CATEGORIES`。

5. 添加 Makefile 的获取部分。

    - 如果不是 .tar.gz，请填写 `EXTRACT_SUFX`。其他示例是 .tar.Z 或 .tgz。
    - 用文件名减去提取后缀填写 `DISTNAME`。如果你有 `foo-1.0.tar.gz`，`DISTNAME` 就是 `foo-1.0`。
    - 用保存 distfile 的位置的 URL 列表填写 `SITES`，例如 https://ftp.openbsd.org/pub/OpenBSD/distfiles/ 。**不要忘记末尾的斜杠**。尽量至少有三个不同的站点。将最容易访问的放在第一位，因为它们是按顺序遍历的。
    - 请记住，fetch 引用文件为 `${SITES}${DISTNAME}${EXTRACT_SUFX}`。所有三个都被使用。不要将 `DISTNAME` 设置为直接指向文件。
    - 你可以通过键入 `make makesum` 来检查你是否正确填写了这些值。

    对于更复杂的 ports，你有更多可用的选项和工具：

    - 你还有变量 `PATCHFILES` 可用。这是 port 的供应商（非 OpenBSD）补丁列表。常见的用途是像安全或可靠性修复这样的东西，它们偶尔作为补丁而不是完整版本分发。
    - 如果你的 ports 可通过大型公共镜像（如 GNU、Sourceforge 或 CPAN）获得，我们已经在 `/usr/ports/infrastructure/db/network.conf` 中为你提供了一个站点列表。将 `SITES` 设置为 `${SITE_GNU}` 或 `${SITE_SOURCEFORGE}` 等。为了简化此过程，使用构造 `${SITE_FOO:=subdir/}` 来附加分发子目录。
    - 对于从 GitHub 分发源代码的 ports，有几种可能性。
        - 在某些情况下，例如 icinga 或 darktable，已经准备好了一个分发 tar 文件，并在 /releases/ 目录中可用。如果可用，请优先使用这些选项，因为它们通常包含正确的构建基础设施（配置脚本等），这些在“标记”版本中通常缺失。像往常一样指定，使用类似 `SITES=https://github.com/acct/project/releases/download/relname/` 的东西。
        - 在其他情况下，文件已被标记，但它们依赖于 github 的即时归档创建。这些可以使用以下 <a href="https://man.openbsd.org/bsd.port.mk">bsd.port.mk(5)</a> 变量指定：`GH_ACCOUNT`、`GH_PROJECT`、`GH_TAGNAME`。
        - 偶尔别无选择，只能让 port 仅引用提交 id。在这种情况下，应使用 `GH_COMMIT` 代替 `GH_TAGNAME`。
    - Ports 通常对应于给定的软件版本。一旦它们被检索，文件将被校验和并与 distinfo 中记录的校验和进行比较。因此，为了避免混淆，`DISTFILES` 和 `PATCHFILES` 应该有清晰可见的版本号：如果是指向 `foo-1.0.5.tar.gz` 的链接，不要检索 `foo-latest.tar.gz`。文件名不应 **仅** 包含版本号 (`1.0.5.tar.gz`)；如果需要，可以在获取期间重命名文件，参见 <a href="https://man.openbsd.org/bsd.port.mk#DISTFILES">bsd.port.mk(5)</a>。如果某些东西仅在未版本化的文件中可用，请温和地要求原始程序作者明确这种区别。与此同时，如果你必须使用这样的文件，请将 `DIST_SUBDIR` 设置为合理的 `DISTNAME`，如 `foo-1.0.5`，以便不同、同名的 distfile 版本不会在分发文件镜像或构建机器上发生冲突。
    - 如果给定的 port 需要大约五个以上的 `DISTFILES` 和 `PATCHFILES` 才能工作，请使用 `DIST_SUBDIR` 以避免过度混乱 `DISTDIR`（默认为 `/usr/ports/distfiles`）。在这种情况下，`DIST_SUBDIR` 必须不包括版本号。当 port 更新到更高版本时，一些 distfiles 可能不会改变，但如果更改了 `DIST_SUBDIR`，它们将被重新获取。即使所有 distfiles 都改变了，用户也更容易跟踪垃圾。
    - 所有 `DISTFILES` 和 `PATCHFILES` 不一定来自同一组 `SITES`。可以使用后缀定义补充站点：`DISTFILES.alt = foo-1.0.5.tar.gz` 将来自 `SITES.alt`。
    - 某些 ports 并不总是需要在所有情况下检索所有文件。例如，某些 ports 可能有一些编译选项，以及仅在这种情况下才需要的相关文件。或者它们可能仅在某些架构上需要某些文件。在这种情况下，那些补充的可选文件必须在 `SUPDISTFILES` 变量中提及。诸如 `makesum` 或 `fetch-all` 之类的目标将获取普通用户不需要的那些补充文件。

6. 通过键入 `make makesum` 在 `distinfo` 中创建校验和。

7. `DISTFILES*` 中的所有文件通常在 `make extract` 期间处理。`EXTRACT_ONLY` 可用于将提取限制为文件子集（可能为空）。此变量的惯常用法是自定义提取：例如，如果某些 `DISTFILES` 需要特殊处理，它们将从 `EXTRACT_ONLY` 中移除并在 `post-extract` 阶段手动处理。由于历史原因，`make extract` 确实会在提取文件的同时首先设置工作目录。因此，提供 `pre-extract` 或 `do-extract` 目标是非常不寻常的（并且是相当可疑的行为，表明 port 中存在高度混淆）。

8. 需要特定处理的补丁应在 `DISTFILES` 中提及，并从 `EXTRACT_ONLY` 中移除，这是出于历史原因。

9. 使用 `make extract` 提取 port。注意源代码的基础在哪里。通常，它是 `/usr/ports/pobj/${PKGNAME}${FLAVOR_EXT}/${DISTNAME}`。如果不同，你可能需要修改 `Makefile` 的 `WRKDIST` 变量。

10. 阅读安装文档并记下构建 port 必须做什么以及可能需要的任何特殊选项。

11. 现在也是弄清楚什么样的许可限制适用于你的 port 的好时机。许多是可以自由重新分发的，但也有不少不是。我们需要回答两个问题才能正确分发 ports。这些是 `Makefile` 中的 `PERMIT_*` 值。

    `PERMIT_PACKAGE` 告诉我们是否可以将软件包放在镜像上。
    `PERMIT_DISTFILES` 告诉我们是否可以镜像 distfiles。

    如果两者都被允许，只需设置 `PERMIT_PACKAGE=Yes`。否则，将每个变量设置为 `Yes` 或指示不允许分发原因的注释字符串。注意以后可能需要满足的任何特殊条件。例如，某些 ports 需要安装许可证副本。我们建议你将许可证放在 `/usr/local/share/doc/<name>/` 中。

    除了 `PERMIT_*` 值之外，在它们上面放一个像 `# License` 这样的许可证标记作为注释，这样我们就知道为什么 `PERMIT_*` 值被设置为那样。对于 GPL，指定适用的版本，例如 "GPLv2 only"、"GPLv2+"（意思是“v2 或更新版本”）、"GPLv3 only" 等。

12. 向 `Makefile` 添加配置选项和/或创建配置脚本。

    - 如果使用 GNU configure，运行 `./configure --help` 查看有哪些选项可用。特别是，寻找你的系统上可能不存在，但在另一个系统上或在批量构建中可能会被拾取的可选依赖项。
    - 任何你可能想要覆盖的东西都可以通过将 `--option` 标志添加到 `Makefile` 中的 `CONFIGURE_ARGS` 参数来更改。
    - 使用 `CONFIGURE_ARGS+=` 追加到变量。`CONFIGURE_ARGS=` 将覆盖它。

13. 尝试使用 `make build` 构建 port。

    - 如果你幸运的话，port 将一路通过没有错误。
    - 如果它以错误退出，你需要为你的 port 生成补丁。弄清楚需要更改什么并为其制作补丁。
    - 为文件制作补丁的顺序通常是：
        - `cd `make show=WRKSRC` ; cp foo/bar.c{,.orig.port}`
        - 编辑 `foo/bar.c` 以修复错误。
        - `cd - ; make update-patches`
        - 这将创建带有你修改的 `patches/patch-foo_bar_c`。
    - 重置 port 并测试补丁的最简单方法是 `make clean patch`。这将删除工作目录，重新提取并修补你的 port。

14. 开始 `make build`、编辑源代码、使用 `make update-patches` 生成补丁和 `make clean patch` 的循环。

    - 补丁放在 `patches/` 目录中，应该命名为 patch-*，其中 * 是有意义的东西。首选命名是 `patch-FILENAME`，其中 `FILENAME` 是它正在修补的文件的名称。（`make update-patches` 会自动为你做这件事。）
    - 应用 `PATCHFILES` 是 `make patch` 阶段的前半部分。它可以作为 `make distpatch` 单独调用，这对 porters 来说是一个方便的目标。如果你没有设置它，请忽略此项。
    - 请每个补丁文件只修补一个源文件。
    - 使用 `make update-patches` 生成补丁。
    - 所有补丁必须相对于 `${WRKDIST}`。
    - 检查补丁**不要**包含 cvs 会替换的标签（<a href="https://man.openbsd.org/update-patches">update-patches(1)</a> 会警告你）。如果包含，你的补丁在你签入后将无法应用。你可以使用 `-kk` 签入你的更改以避免这种情况。
    - 在补丁文件的开头写一段关于其目的的简短说明（不是强制性的，但例如对于安全补丁通常会这样做）。
    - 如果你的补丁在 ports 树之外使用也有意义，**请**将它们反馈给该软件的作者。

15. 尝试设置 `SEPARATE_BUILD`。

    - 如果 port 可以在其源代码树之外使用目标文件构建，这会更干净（许多使用 `CONFIGURE_STYLE=gnu` 的程序可以）。
    - 这也可以为你节省一些精力，因为你大多数时候可能能够在 configure 处重新开始循环。

16. 细读输出（如果有）并调整 `Makefile` 中的任何选项。要重复，发出命令 `make clean configure`。

    配置文件通常将安装在 `/usr/local` 下的某个位置，并由 <a href="https://man.openbsd.org/pkg_add">pkg_add(1)</a> 的 `@sample` 注释处理。安装软件包后，如果存在 `pkg/MESSAGE`，将显示其内容。

    OpenBSD 文件位置是：

    ```text
    用户可执行文件:			/usr/local/bin
    系统管理可执行文件:		/usr/local/sbin
    程序可执行文件:			/usr/local/libexec
    库:				/usr/local/lib
    架构相关数据:		/usr/local/lib/<name>
    安装的包含文件:		/usr/local/include 或
    					/usr/local/include/<name>
    单机数据:			/etc 或 /etc/<name>
    本地状态:				/var/run
    游戏得分文件:			/var/games
    GNU info 文件:				/usr/local/info
    手册页:				/usr/local/man/...
    只读架构无关数据:	/usr/local/share/<name>
    杂项文档:			/usr/local/share/doc/<name>
    示例文件:				/usr/local/share/examples/<name>
    ```

17. 开始 make 循环，直到 port 准备就绪。修补（见上文）清理，并 make 直到生成 port。尽可能摆脱所有警告，特别是与安全相关的警告。

18. 在 `Makefile` 中添加 `COMMENT`。`COMMENT` 是 port 的**简短**单行描述（最多 60 个字符）。**不要**在注释中包含软件包名称（或软件版本号）。**不要**以大写字母开头，除非语义上很重要，并且**不要**以句号结尾。**永远不要以不定冠词如 "a" 或 "an" 开头 - 完全删除冠词。** `COMMENT` 的主要用途是来自 <a href="https://man.openbsd.org/pkg_info">pkg_info(1)</a> 的单行信息：
    ```text
    py3-cairo-1.24.0    cairo bindings for Python
    ```

19. 将 port 的较长描述放入 `pkg/DESCR` 中。一到几段简明扼要地解释 port 的作用就足够了。行长不应超过 80 个字符。这可以通过首先编辑 `DESCR` 文件然后通过 `fmt` 运行它来完成。

20. 如果应用程序需要创建用户或组，请从 `/usr/ports/infrastructure/db/user.list` 中选择最低的空闲 id 供你的 port 使用，并且不要忘记将该文件包含在你的提交中。

21. 使用 `make fake` 安装应用程序。库永远不应该被剥离 (stripped)。可执行文件默认被剥离，除非你设置了 `DEBUG_PACKAGES`；这由 `${INSTALL_STRIP}` 控制。`${INSTALL_PROGRAM}` 自动遵守这一点，优于无条件剥离（例如，通过 `install-strip` 目标或从 `Makefile` 运行 `strip`）。你可以使用 `objdump --syms` 来确定二进制文件是否被剥离。被剥离的文件在 `SYMBOL TABLE` 中没有符号。

22. **再次检查 port 是否存在安全漏洞**。这对于面向网络的程序尤为重要。为此请参阅<a href="#PortsSecurity">我们的安全建议</a>。

23. 确保你的 `/etc/mtree` 目录是最新的。（下一步使用 mtree 列表从生成的装箱单中删除现有目录）。记住 OpenBSD `(U)pdate` 不会触及 `/etc`... 对于 `/etc` 的自动更新，<a href="https://man.openbsd.org/sysmerge.8">sysmerge(8)</a> 可能会有所帮助。

24. 创建 `pkg/PLIST`。在使用 `make fake` 安装完成后，使用开发者命令 `make update-plist`，它会在 `pkg` 目录中创建或更新文件 `PLIST`。此文件是候选装箱单。

    细读 `PLIST` 并验证所有内容是否已安装以及是否安装在正确的位置。任何未安装的内容都可以添加到 port `Makefile` 的 `post-install` 规则中。请注意，`PLIST` 注释记录在 <a href="https://man.openbsd.org/pkg_create">pkg_create(1)</a> 手册中，并且 <a href="https://man.openbsd.org/update-plist">update-plist(1)</a> 也有记录。

25. 可能有些目录不需要在 `PLIST` 中，因为它们已由依赖项安装；如果你添加到了 `LIB_DEPENDS` 或 `RUN_DEPENDS`，运行 `make update-plist` 以删除这些目录。

26. 使用 `make package` 测试打包，使用 `make install` 测试生成的软件包的安装，并使用 `make uninstall` 测试其移除。在处理多软件包时，直接使用 <a href="https://man.openbsd.org/pkg_add">pkg_add(1)</a> 和 <a href="https://man.openbsd.org/pkg_delete">pkg_delete(1)</a> 可能会更方便，在环境中将 `TRUSTED_PKG_PATH` 设置为 `/usr/ports/packages/%a/all/`。

27. 验证依赖项。细读你的日志以验证 port 是否确实检测到了 `*DEPENDS` 中提到的内容，且仅此而已。检查名称，特别是在 `make configure` 阶段，以查找隐藏的依赖项（存在于 ports 树中其他地方且如果用户首先安装了其他 ports 可能会被检测到的东西）。

28. 验证共享库依赖项。运行 `make port-lib-depends-check` 并添加所需的每个 `LIB_DEPENDS` 或 `WANTLIB` 注释，直到它运行干净为止。你可能想阅读<a href="#PortsUpdate">更新指南</a>以了解为什么这如此重要。

29. 检查回归测试，以及它们是否运行干净。如果 port 没有测试基础设施，设置 `NO_TEST=Yes`。如果运行测试需要依赖项，但构建 port 不需要，请在 `TEST_DEPENDS` 中列出它们。请注意：如果 port 有空的回归测试基础设施，不要设置 `NO_TEST`。

30. 测试 `make clean` 是否成功。有时 `make fake` 阶段会在构建目录中创建导致此操作失败的文件。

31. 在你的 port 目录中运行 <a href="https://man.openbsd.org/portcheck">`/usr/ports/infrastructure/bin/portcheck`</a> 实用程序，并处理它发现的问题（如果有）。

32. 发送邮件给 <a href="mailto:ports@openbsd.org">ports@openbsd.org</a>，附上描述、主页（如果有）以及请求评论和测试的简短说明。确保将 port/补丁附加到此电子邮件（或包含可以找到它的 URL）并发送出去。

    尝试让其他人在各种平台上为你测试它。

    - AMD64 系统很好，因为它们很快，而且在这个平台上 `sizeof(int) != sizeof(long)`。
    - sparc64 系统很好，因为它们相当普遍，可以很快，而且它们的字节序与 i386 相反；当然，如果你在 sparc64 上开发，你会希望在 i386 上测试它。

33. 整合你得到的任何有用反馈。在你的平台上再次测试。让那些给你反馈的人从你的新 port 再次测试。

34. 将 port 提交到 CVS。

    如果你不是拥有 CVS 访问权限的开发人员，那么你将不得不找一位 OpenBSD 开发人员为你执行以下步骤（在 <a href="mailto:ports@openbsd.org">ports@openbsd.org</a> 上询问）。

    在你导入任何东西之前，至少获得另一位 ports 开发人员的一个 "OK"（越多越好）。

    如果在 PLIST 文件中使用 `@newuser` 或 `@newgroup`，请检查自 port 创建以来没有向 `/usr/ports/infrastructure/db/user.list` 添加用户。

    对于新 ports，我们使用 `cvs import`，而不是单独添加新文件。例如，要导入一个新的 `lang/kaffe1` port，首先进行试运行导入：

    ```shell
    $ cd /usr/ports/lang/kaffe1
    $ cvs -ndcvs.openbsd.org:/cvs import ports/lang/kaffe1 username username_yyyymmdd
    ```

    其中 `username` 是你的帐户用户名，`yyyymmdd` 是当前日期。如果这成功了，那么你可以删除 `-n` 进行真正的导入。你的编辑器将被调用，以便你可以输入提交消息。在提交消息中，至少说明你正在导入什么以及哪些开发人员提供了 OK。确保导入路径正确；它总是以 `ports/` 开头。

    或者，你可以使用 <a href="https://man.openbsd.org/portimport">`ports/infrastructure/bin/portimport`</a> 来导入新 ports。

35. 最后但同样重要的一点是，在其父目录的 `Makefile` 中为新 port 添加一行条目，例如，对于 `ports/lang/kaffe1`，将其添加到 `ports/lang/Makefile`。不要忘记提交对 `/usr/ports/infrastructure/db/user.list` 所做的任何更改。

36. 维护 port！随着时间的推移，可能会出现问题，或者可能会发布该软件的新版本。你应该努力保持你的 port 最新。换句话说 - 迭代，测试，测试，迭代...

37. 更新 port 时，记得处理依赖项！你不应该破坏任何依赖于你的 port 的 port。`sqlports` 软件包包括 `show-reverse-deps`，这使得了解直接或间接依赖于给定 port 的完整 ports 树变得微不足道。如果出现问题，请与这些 ports 的维护者沟通。同样，警惕依赖项更新，并检查维护者是否尽职尽责。

<h2 id="PortsComplex">处理复杂情况</h2>

假设你已经设法构建了软件，提供了所需的补丁，并且你想完成 port。

<h3 id="Know">了解软件</h3>

<dl>
<dt>识别选项</dt>
<dd>第一步通常是识别构建选项。你经常需要阅读配置日志，看看你的 port 自动检测到了什么。阅读 configure 脚本选项。阅读 port 文档以获取额外内容。</dd>
<dt>使选项工作</dt>
<dd>使用各种选项重新编译你的 port。安装额外的依赖项。确保你的 port 正确检测到它们。添加补充补丁以确保编译。测试结果，并验证额外的东西是否有效。</dd>
<dt>识别缺失的软件</dt>
<dd>一些依赖项将无法满足，因为缺失的软件尚未被移植。强烈建议显式禁用这些选项。如果不这样做，会一直破坏批量构建：人们移植新软件并导入它，不久之后，旧 ports 停止构建，因为它们检测到依赖项，尝试使用它，然后构建或打包失败。</dd>
<dt>检查运行时依赖项与构建依赖项</dt>
<dd>使用 <code>make update-plist</code> 更新你的装箱单。使用 <code>make port-lib-depends-check</code> 查看你的软件需要什么库（这通常会最终出现在 <code>LIB_DEPENDS</code> 或 <code>WANTLIB</code> 中）。识别依赖项中必须存在才能使 port 工作的各种文件和二进制文件。</dd>
</dl>

到目前为止，你应该对你的 port 的工作原理有了相当的了解。

<h3 id="Figure">找出重要选项</h3>

你不会关心某些选项。如果某些东西总是有效，并且依赖项非常小，那么禁用它是没有意义的。特别注意依赖项的许可证，尤其是 `PERMIT*` 的东西。通常，即使依赖项非常小，如果它影响生成的软件包的许可，你也必须显式处理它。

考虑到所有可能的选项，你应该只剩下一组更小的 port 选项，主要取决于运行软件需要什么软件包。目前，不要担心构建依赖项。请记住，OpenBSD ports 系统专注于最终用户，最终用户将希望安装二进制软件包，所以如果你需要一大块软件来构建你的 port，只要它不作为库或运行时依赖项出现，这就没关系。

<h3 id="Ideal">理想情况：<code>MULTI_PACKAGES</code> 和 <code>PSEUDO_FLAVORS</code></h3>

现在，你应该对以下内容有了一个相当好的了解：

- 当你激活每个选项时会出现哪些新文件
- 当你激活每个选项时需要哪些库/运行时文件

在理想情况下，构建选项只会创建具有新依赖项的新文件，而不会影响其他东西。这是插件框架的一个相当常见的情况：你添加一个库，你得到一个新的插件。这也经常发生在带有图形前端的核心应用程序中：控制台应用程序每次都构建，而 x11 界面显示为一个单独的二进制文件。

在这种情况下，尝试将 `MULTI_PACKAGES` 变量设置为 -sub 软件包列表，添加相应的 `COMMENTS`，并查看你的打包。基本上，`MULTI_PACKAGES` 仅影响打包：如果你有 `MULTI_PACKAGES=-s1 -s2`，所有与软件包相关的东西都将存在两个变体：

第一个软件包的 `COMMENT-s1`，第二个软件包的 `COMMENT-s2`，`PLIST-s1`，`PLIST-s2`，`DESCR-s1`，`DESCR-s2`。你需要在 `Makefile` 中编写那些 `COMMENT-s1` 和 `COMMENT-s2`，将你的 `PLIST` 分成两部分，并创建 `DESCR-s1/DESCR-s2`。

从所需的最小框架工作开始是一个好主意：只需复制现有的描述和注释，因为在完善它之前，你将不得不摆弄 `MULTI_PACKAGES` 和其他东西。

一旦你正确分离了文件，你需要检查依赖项：`LIB_DEPENDS`、`WANTLIB` 和 `RUN_DEPENDS` 将为每个子包拆分。通常是时候检查你的多重打包是否“有效”，以及你不想强加给用户的那些讨厌的依赖项是否确实被归类到特定的子包中。

假设一切正常，你基本上完成了。只需为各种软件包选择合理的名称，并填写注释和描述。最终用户将能够只安装他们想要的软件包。

但是等等。你说构建呢？好吧，在构建期间有很多依赖项不是问题。大多数软件包由 OpenBSD 团队使用特殊的构建运行（称为批量构建）构建，开发人员只需在专用机器（或几台，对于慢速架构）上构建所有可能的软件包。由于所有东西都会被构建，拥有大的依赖项不是问题。但是，多次构建同样的东西是一个问题，这就是为什么 `MULTI_PACKAGES` 是处理选项的最佳方式（如果可能）：一次构建，一组要测试的软件包，总体质量更好...

如果你还想帮助那些自己构建软件包的人，你可以考虑添加 `PSEUDO_FLAVORS`。伪 flavor 是一种调整选项（比如，禁用图形界面）的方法，与实际 flavors 非常相似。事实上，最大的区别是功能上的区别：伪 flavor 应该只影响构建的软件包集，但绝不允许修改实际的软件包内容。

例如，假设你将图形界面分离到一个单独的子包中（`MULTI_PACKAGES=-main -x11`），你可以创建一个伪 flavor `no_x11` 来避免构建 -x11 子包。关键点是这个 flavor 不应该以任何方式影响 -main 软件包。

你最终会得到一个看起来像这样的 `Makefile`：

```text
COMMENT-main =	foo core application
COMMENT-x11 =	foo graphical interface

V =		1.0
DISTNAME =	foo-$V
CATEGORIES =	app

MULTI_PACKAGES =	-main -x11
PSEUDO_FLAVORS =	no_x11
FLAVOR ?=

WANTLIB = c m crypto ssl
WANTLIB-x11 = ${WANTLIB} X11 Xt

RUN_DEPENDS-x11 =	${BASE_PKGPATH},-main>=$V

CONFIGURE_STYLE =	gnu

.include <bsd.port.arch.mk>

.if !${BUILD_PACKAGES:M-x11}
CONFIGURE_ARGS += --disable-x11
.endif

.include <bsd.port.mk>
```

请注意，你只需要在 `Makefile` 中编写一个非常小的条件部分：系统根本不关心你定义了额外的变量。

<h3 id="Inter">子包之间的相互依赖</h3>

`MULTI_PACKAGES` 设置曾经是不对称的，有一个 -main 子包和其他子包，-main 子包总是被构建，其他子包可能依赖于它。现在的情况是完全对称的：任何子包都可以依赖于任何其他子包。基础设施有特定的规定来避免无限循环。

基础设施特别关注库的相互依赖性：它可以检测哪些 `WANTLIB` 来自外部依赖项，哪些来自相互依赖项。虽然外部 `LIB_DEPENDS` 和 `WANTLIB` 在构建开始时检查，但引用当前正在构建的子包之一的 `LIB_DEPENDS` 和 `WANTLIB` 将仅在打包时检查（因此可能会以特定顺序创建软件包以满足相互依赖性）。

基础设施提供了特定的变量来帮助编写相互依赖性：`BUILD_PKGPATH` 包含构建当前软件包期间使用的 `PKGPATH`，考虑了 flavors 和伪 flavors。强烈建议使用此变量而不是自己编写：如果不这样做，通常会在有趣的 flavors 情况下触发重建。例如：

```text
...
FLAVORS = a b
FLAVOR ?=
MULTI_PACKAGES = -p1 -p2
WANTLIB-p1 = foo
LIB_DEPENDS-p1 = some/pkgpath,-p2
...
```

如果你继续并在 some/pkgpath 中使用 `FLAVOR=a` 构建，那么为 `-p1` 创建子包将触发使用 `FLAVOR=''` 的重建。你应该这样写：

```text
LIB_DEPENDS-p1 = ${BUILD_PKGPATH},-p2
```

还有一个 `BASE_PKGPATH` 变量，它不考虑伪 flavors。此变量的适用性有限：它对应于旧 `MULTI_PACKAGES` 和新 `MULTI_PACKAGES` 之间的过渡，其中旧的 `main` 子包在其 pkgpath 中没有任何标记，因此相应的软件包在其装箱单中需要一个 `@pkgpath ${BASE_PKGPATH}`。（一般来说，伪 flavors 是构建信息，不应该进入软件包和装箱单）。

<h3 id="True">真正的 <code>FLAVORS</code> 和 <code>PKGNAMES</code></h3>

在某些情况下，配置选项过于具有侵入性，你必须向 `Makefile` 添加真正的 flavors：这些 flavors 将命令一些配置选项，通常还会添加到各种依赖项中。请注意，软件包命名大多是自动的：`PKGNAME` 将通过将其名称附加指定的 flavors 来构建扩展名。所以，如果

```text
PKGNAME = foo-1.0
FLAVORS = f1 f2 f3
```

并且你使用 `FLAVOR='f3 f1'` 构建 port，那么 `FULLPKGNAME=foo-1.0-f1-f3`（`FLAVORS` 用于以规范方式重新排序指定的 flavors）。

有时会出现混合情况，其中一些软件包确实依赖于 `FLAVOR`，而另一些则不依赖。例如，某些 ports 包含大量不依赖于 `FLAVOR` 的文档，以及一些依赖于 `FLAVOR` 的实际程序。在这种情况下，你可以显式指定文档子包的 `FULLPKGNAME`。像这样：

```text
CATEGORIES = app
COMMENT-core = foo application
COMMENT-doc = foo documentation
V = 1.0
DISTNAME = foo-1.0
PKGNAME-core = foo-$V
FULLPKGNAME-doc = foo-doc-$V
FLAVORS = crypto

MULTI_PACKAGES = -core -doc
WANTLIB-core = c m

.if ${FLAVOR:L:Mcrypto}
WANTLIB-core += ssl crypto
CONFIGURE_ARGS += --enable-crypto
.endif
```

如文档中所述，所有软件包名称都具有相同的结构：`stem-version-flavor_extension`。

默认情况下，具有相同词干 (stem) 的软件包确实会冲突，更新路径将查看具有相同词干的候选者。正确的软件包将是来自完全相同的 `PKGPATH` 的软件包，或者匹配装箱单中的 `@pkgpath` 注释。

通常，`MULTI_PACKAGES` 不应冲突，因此它们必须具有不同的名称（并且基础设施无法构建这些名称）。另一方面，flavors 应该冲突，因此具有相同的名称。flavor 信息应在软件包名称的末尾结束，除了伪 flavors，它们不会改变软件包的构建方式。

就依赖项而言，默认情况下，指定 `PKGPATH` 将仅创建 `stem-*` 依赖项，这意味着任何具有正确词干的软件包都将匹配依赖项。默认情况下，任何 flavor 都会匹配。如果只需要特定的 flavors，你必须将它们包含在你的规范中，例如 `stem-*-flavor`。如果不需要某些 flavors，你可以从匹配的软件包中删除它们，例如 `stem-*-!flavor`。

自 OpenBSD 4.9 以来，要求至少某个版本的依赖项可以直接附加到 `PKGPATH`，例如 `dir/foo>=2.0`。

<h2 id="PortsUpdate">更新 Ports</h2>

软件包工具可以进行更新，因此维护者必须意识到一个简单的事实：更新不是瞬间完成的。即使用户只是从一个版本升级到另一个版本，每次运行 `pkg_add -u` 时，系统都会用新版本替换每个软件包，一次一个软件包。所以用户将运行一个混合系统，即使只有几分钟。

基本上有两种维护者必须意识到的更新模型。

- 一些用户松散地跟随 current。他们每隔几天/几周更新一次软件包。要么所有软件包，要么只是一些选定的软件包。更新机制应该对他们有效：它可以强制他们更新某些特定的软件包，或者安装一些新东西，但它不应该静默失败。微更新必须经过测试。这些用户通常能够应对小的变化，如程序名称更改或配置调整。
- 一些用户每六个月随新版本更新一次。他们还将下载稳定更新（安全和关键更新）。在六个月期间，ports 树的很大一部分发生了变化。这些用户不关心单个软件的更改。他们只想要一个能工作的系统。更改二进制名称对这些用户来说完全不友好。调整数百个配置文件对他们来说将是一个巨大的痛苦。维护者可以通过提供平滑的更新路径，并在重要事项发生变化时提供主要提示来提供帮助。

你应该注意，更新过程的一部分，特别是针对每六个月更新一次的用户的宏观变化，尚未自动化。全局更新机制仍在进行中，pkg_add 将来能够处理更多问题。目前，你应该专注于确保系统正确更新，一次一个 port，并且就冲突和其他问题而言，一个 port 的更新会考虑到其他 ports。

<h3 id="UpdateChecklist">更新检查清单</h3>

部分工作是在制作 port 本身时完成的。

- 确保软件作者了解使其在 OpenBSD 上运行所需的更改。有些更改仅与 ports 策略有关，这些通常没有反馈的意义，但除此之外，将更改包含在上游可以使作者更容易在 OpenBSD 上进行测试（他们通常希望在 ports 框架之外进行此操作），并减少以后自己合并冲突补丁的需要。更新时，检查补丁是否仍然有意义。也许同一个问题以不同的方式修复了，在这种情况下，补丁可能仍然适用但应该被删除。
- 确保软件作者了解版本编号。有时 distfiles 的名称不包含版本号，或者引用一个分支 - 也就是说，它们不是代码中的固定点，而是一个移动的目标。如果发生这种情况，请与作者沟通并尝试获得永久 URL。
- 记录变通方法。这将帮助你在尝试更新 port 时记住检查它们是否仍然必要。
- 记录依赖项，尤其是你不使用的东西。某些 ports 可以使用移植时可能不可用的外部软件。确保你不拾取它，并记录下来，以便稍后当此软件可用时你可以更新你的 port。
- 如果 port 使用 libtool，请从文件 `${WRKBUILD}/shared_libs.log` 逐字复制其日志作为 `SHARED_LIBS` 设置的基础。这将有助于更新。
- `PLIST_REPOSITORY` 默认设置，并在你的系统上构建软件包时构建装箱单数据库。这对于找出被遗忘的软件包名称提升很有用，也可以检查与旧软件包的冲突。
- 确保依赖项尽可能宽松。默认情况下，`RUN_` 和 `LIB_DEPENDS` 将由任何版本的软件包满足。除非必须，否则不要坚持给定的版本。尽可能使用最低版本。

Ports 经常需要在上游没有新版本的情况下进行小更新。

- 每个 port 更新都需要提升软件包名称。否则，二进制软件包的更新机制将不起作用。任何影响二进制软件包的东西都意味着提升。这包括 `HOMEPAGE`、`MAINTAINER` 或描述更改，补丁或构建标志的更改。如果上游版本没有更改，则通过增加 `REVISION`（如果已存在）来完成软件包名称提升，否则在 Makefile 顶部添加 `REVISION = 0`。如果使用了 `MULTI_PACKAGES`，增加 `REVISION` 以设置所有子包的默认修订版，或根据需要增加单个 `REVISION-subpkg`（这将覆盖 `REVISION`）。如果有疑问，请在进行更改之前和之后使用 `make show=PKGNAMES` 进行检查。
- 版本号总是向前走的。如果发生意外情况并且你不得不恢复更新，或者如果上游编号完全改变以至于版本号看起来倒退，你必须使用 `EPOCH` 来确保 pkg_add(1) 将软件包视为较新的软件包。这将把 `v${EPOCH}` 添加到 `FULLPKGNAME` 以形成符合 <a href="https://man.openbsd.org/packages-specs.7">packages-specs(7)</a> 的完整软件包名称。
- 如果软件包从未用特定版本构建过，则不需要提升。
- 确保 port 不拾取/拾取外部依赖项的更改需要提升。
- 现有依赖项的更改通常不影响 port 版本号。软件包系统使用签名机制，使得二进制软件包由其软件包名称、构建它的依赖项以及它包含的所有共享库的版本号完全标识。因此，如果你的 port 依赖于一个库，该库版本号的更改将触发从你的 port 构建的软件包的更新。
- 版本比较顺序可以概括为：`0.1 < 0.1p0 < 0.2 < 0.1v0 < 0.1p0v0 < 0.2v0 < 0.1v1`。此外还处理了包括 `beta`、`rc` 和 `pre` 在内的附加词。有关更多详细信息，请参阅 <a href="https://man.openbsd.org/packages-specs.7">packages-specs(7)</a>，并使用 <a href="https://man.openbsd.org/pkg_check-version.1">pkg_check-version(1)</a> 检查版本号的顺序。

部分工作将在更新本身之前进行。

- 运行 `make patch` 以在更新之前拥有 port 的初始副本。

然后是更新。

- 编辑 port 的 `Makefile` 以获取新版本，运行 `make makesum` 和 `make patch` 作为起始基础。
- 一旦你修复了失败的补丁，在旧版本和新版本之间运行完整的 diff 以确切弄清楚发生了什么变化。做笔记，准备好稍后重新审视事情。
- 做任何必要的移植工作以使新版本在 OpenBSD 下运行：调整依赖项，更改软件包内容等。
- 再次检查共享库版本。对于基于 libtool 的 ports，你有 `shared_libs.log` 来检查软件作者是否做了重大更改。*请注意这还不够。* 许多软件作者并不真正理解共享库问题。你必须阅读旧版本和新版本之间的完整 diff，并相应地提升库版本。有疑问时，提升主版本。
- 注意与已构建软件包的冲突。对于简单的更新，无需执行任何操作。如果你将一个软件包拆分为几个子包，请确保子包具有适当的冲突注释：它们通常应该与“旧的”单个软件包冲突。
- 帮助用户。如果除了 `pkg_add -u` 之外还需要采取某些特定步骤，请确保它们在软件包中可见。当你的打包策略发生变化时，在软件包描述中记录下来。例如，如果你因空间限制将文档移动到单独的软件包中，主软件包描述应提及文档现在可作为单独的软件包使用。

<h3 id="CommitUpdates">提交 Port 更新</h3>

当 port 更新准备好后，将其提交到 CVS。如果你没有 CVS 帐户，那么你需要找一位开发人员为你做以下事情（在 <a href="mailto:ports@openbsd.org">ports@openbsd.org</a> 上询问）。

1. 至少获得另一位 ports 开发人员的一个 "OK"。
2. 使用 `cvs add` 添加任何新文件。
3. 使用 `cvs rm` 删除不再需要的文件（例如已上游化的补丁）。
4. 检查 `cvs -n up -d` 的输出。新文件应标记为 `A`，已删除文件应标记为 `D`，已更改文件应标记为 `M`。寻找标记为 `?` 的文件 - 你是否打算 `cvs add` 它们？
5. 如果一切正常，使用 `cvs commit` 提交新/已删除/已更改的文件。调用 `cvs commit` 时，你可以单独列出文件，或者如果你不提供文件名，CVS 将递归提交（小心使用此功能）。你将被要求输入提交消息，其中你应该说明正在更新哪个 port，以及谁提供了 OK。

<h2 id="PortsPolicy">OpenBSD 移植策略</h2>

- OpenBSD 不使用 `/usr/local/etc/rc.d`。由于 NFS，`/usr/local` 经常在几台机器之间共享。由于这个原因，特定于给定机器的配置文件不能存储在 `/usr/local` 下。`/etc` 是每台机器配置文件的中央存储库。此外，OpenBSD 的策略是永远不要自动更新 `/etc` 下的文件。需要某些特定引导设置的 Ports 应该建议管理员该做什么，而不是盲目地安装文件。
- OpenBSD 不压缩手册页。
- OpenBSD 不需要 `-lcrypt`、`-ldl` 或 `-lrt`。这些库提供的功能是 `libc` 的一部分。`crypt()` 函数不支持 DES，只支持 bcrypt。
- OpenBSD 为 ports 创建的用户和组有一个单独的命名空间。有关详细信息，请参阅 `/usr/ports/infrastructure/db/user.list`。
- OpenBSD 强烈面向安全。你应该阅读并理解本页的 <a href="#PortsSecurity">安全部分</a>。
- 目标是让所有移植的应用程序都支持 OpenBSD。为了实现这一目标，将支持在 OpenBSD 上运行的补丁反馈给应用程序维护者。（如果你不是 port 维护者，请先与他们核实。他们可能是有意不这样做的）。

<h2 id="PortsSecurity">安全建议</h2>

有许多安全问题需要担心。如果你不完全确定自己在做什么，请向 <a href="mailto:ports@openbsd.org">ports</a> 邮件列表寻求帮助。

- 在准备 port 时 *不要* 使用 alpha 或 beta 代码。使用最新的常规或补丁版本。
- 任何要作为服务器安装的软件都应扫描缓冲区溢出，尤其是不安全使用 `strcat/strcpy/strcmp/sprintf`。一般来说，`sprintf` 应该替换为 `snprintf`。
- 永远不要使用文件名代替真正的安全性。有很多竞争条件是你无法正确控制的。例如，已经在你的机器上拥有用户权限的攻击者可能会用指向更具战略意义的文件（如 `/etc/master.passwd`）的符号链接替换 `/tmp` 中的文件。
- 例如，`fopen` 和 `freopen` 都 **创建新文件或打开现有文件** 进行写入。攻击者可能会创建一个从 `/etc/master.passwd` 到 `/tmp/addrpool_dump` 的符号链接。你打开它的那一刻，你的密码文件就毁了。是的，即使就在之前有一个 `unlink`。你只是缩小了机会窗口。改用带有 `O_CREAT|O_EXCL` 的 `open` 和 `fdopen`。
- 另一个非常常见的问题是 `mktemp` 函数。注意 bsd 链接器关于其使用的警告。**这些必须修复**。这不像 `s/mktemp/mkstemp/g` 那么简单。有关更多信息，请参阅 <a href="https://man.openbsd.org/mktemp.3">mktemp(3)</a>。使用 `mkstemp` 的正确代码包括 `ed` 或 `mail` 的源代码。在 `rsync` port 中可以找到正确使用 `mktemp` 的罕见代码实例。
- 仅仅因为你可以读取它并不意味着你应该读取。这种性质的一个著名漏洞是 `startx` 问题。作为一个 setuid 程序，你可以将 startx 与任何文件作为脚本一起启动。如果文件不是有效的 shell 脚本，随后会出现语法错误消息，以及违规文件的第一行，而没有任何进一步的权限检查。考虑到这些通常以 root 条目开头，抓取影子密码文件的第一行非常方便。不要打开你的文件，然后在打开的描述符上执行 `fstat` 以检查你是否应该能够打开它（否则攻击者会玩弄 `/dev/rst0` 并倒带你的磁带）——使用正确的 uid/gid/grouplist 设置打开它。
- 在放弃权限之前，不要在 setuid 程序中使用任何会 fork shell 的东西。这包括 `popen` 和 `system`。改用 `fork`、`pipe` 和 `execve`。
- 将打开的描述符而不是文件名传递给其他程序以避免竞争条件。即使程序不接受文件描述符，你仍然可以使用 `/dev/fd/0`。
- 访问权限附加到文件描述符。如果你需要 setuid 权限来打开文件，打开该文件，然后放弃你的权限。你仍然可以访问打开的描述符，但你需要担心的更少。这是双刃剑：即使在放弃权限后，你仍然应该注意那些描述符。
- 尽可能避免 root setuid。基本上，root 可以做任何事情，但很少需要 root 权限，除了可能创建编号低于 1024 的 socket 端口。你必须知道编写守护进程的适当魔法才能实现这一点。可以说，如果你不知道怎么做，你就没有资格编写 setuid 程序。
- 使用 setgid 代替 setuid。除了 setgid 程序所需的那些特定文件外，大多数文件都不是组可写的。因此，setgid 程序中的安全问题不会对你的系统造成太大损害：只有组权限，入侵者能做的最坏的事情就是黑掉游戏得分表之类的东西。有关此类更改的实例，请参阅 `xkobo` port。
- 不要信任组可写文件。即使它们经过审计，setgid 程序也不被视为重要的潜在安全漏洞。因此，它们可以篡改的东西不应被视为敏感信息。
- 不要信任你的环境！这涉及简单的事情，如你的 `PATH`（永远不要使用带有非限定名称的 `system`，避免 `execvp`），但也涉及更微妙的项目，如你的 locale、timezone、termcap 等。注意传递性：即使你采取了充分的预防措施，你直接调用的程序也不一定会。**永远不要**在特权程序中使用 `system`，构建你的命令行，一个受控环境，并直接调用 `execve`。`perlsec` 手册页是关于此类问题的好教程。
- 永远不要使用 setuid shell 脚本。这些本质上是不安全的。将它们包装在一些确保适当环境的 C 代码中。另一方面，OpenBSD 具有安全的 perl 脚本。
- 提防动态加载器。如果你正在运行 setuid，它将只使用 ldconfig 扫描过的受信任库。Setgid 是不够的。
- 动态库很棘手。C++ 代码设置了类似的问题。基本上，库可能会在你的主程序甚至有机会检查其 setuid 状态之前根据你的环境采取一些行动。OpenBSD `issetugid` 从库编写者的角度解决了这个问题。除非你彻底理解这个问题，否则不要尝试移植库。
- 在 OpenBSD 上，<a href="https://man.openbsd.org/rand">rand(3)</a>、<a href="https://man.openbsd.org/random.3">random(3)</a> 和 <a href="https://man.openbsd.org/drand48">drand48(3)</a> 系列默认返回非确定性随机数。任何指定的种子值都会被关联的种子函数忽略，并改用 `arc4random`。如果必须保留确定性（即：可重复）行为，请使用 OpenBSD 扩展：`srand_deterministic`、`srandom_deterministic`、`srand48_deterministic`、`seed48_deterministic` 和 `lcong48_deterministic`。

<h2 id="PortsGeneric">通用移植提示</h2>

- 应该谨慎使用 `__OpenBSD__`，如果非要用的话。看起来像这样的结构

  ```text
  #if defined(__NetBSD__) || defined(__FreeBSD__)
  ```

  通常是不合适的。不要盲目地向其中添加 `__OpenBSD__`。相反，尝试弄清楚发生了什么，以及需要什么实际功能。手册页通常很有用，因为它们包含历史注释，说明特定功能何时并入 BSD。根据已知版本检查 `BSD` 的数值通常是正确的方法。有关更多信息，请参阅 <a href="https://www.netbsd.org/Documentation/pkgsrc/">NetBSD pkgsrc 指南</a>。

- 定义 `BSD` 是个坏主意。尝试包含 `sys/param.h`。这不仅定义了 `BSD`，还给它一个正确的值。正确的代码片段应该看起来像：

  ```text
  #if (defined(__unix__) || defined(unix)) && !defined(USG)
  #include <sys/param.h>
  #endif
  ```

- 测试功能，而不是特定的操作系统。从长远来看，测试 `tcgetattr` 是否工作比测试你是否在 BSD 4.3 或更高版本或 SystemVR4 下运行要好得多。这类测试只会混淆问题。解决这个问题的方法是，例如，测试一个特定系统，设置一系列 `HAVE_TCGETATTR` 定义，然后继续下一个系统。这种技术将功能测试与特定操作系统分开。匆忙中，另一个 porter 可以只将整套 `-DHAVE_XXX` 定义添加到 Makefile 中。也可以编写或添加到 configure 脚本以检查该功能并自动添加它。作为一个不该效仿的例子，检查 nethack 3.2.2 源代码：它根据系统类型假设了很多东西。这些假设大多已过时，不再反映现实：POSIX 函数比旧的 BSD 与 SystemV 差异更有用，以至于一些传统的 bsd 函数现在仅通过兼容性库支持。

- 避免包含文件包含其他包含文件，而那些文件又包含... 首先，因为这效率低下。你的项目最终会包含一个包含所有内容的文件。另外，因为这使得一些问题难以修复。在给定点 *不* 包含特定文件变得更加困难。

- 避免奇怪的宏技巧。取消定义由头文件定义的宏是个坏主意。定义宏以从包含文件中获得某些特定行为也是个坏主意：编译模式应该是全局的。如果你想要 POSIX 行为，请说出来，并在整个项目中 `#define POSIX_C_SOURCE`，而不是在你觉得需要的时候。

- 永远不要编写系统函数原型。所有现代系统，包括 OpenBSD，都有这些原型的标准位置。可能的位置包括 `unistd.h`、`fcntl.h` 或 `termios.h`。手册页经常说明可以在哪里找到原型。你可能需要另一系列 `HAVE_XXX` 宏来获取正确的文件。不要担心包含同一个文件两次；包含文件有防止各种恶心情况的守卫。如果某个损坏的系统需要你编写原型，不要强加给所有其他系统。自己编写的原型会因为它们可能使用在你的系统上等效但在其他系统上不可移植的类型（`unsigned long` 而不是 `size_t`），或者弄错一些 `const` 状态而中断。此外，如果你包含正确的头文件，一些编译器，如 OpenBSD 自己的 gcc，能够对一些非常频繁的函数（如 `strlen`）做得更好。

- 仔细检查构建日志是否有任何编译器警告。

    - `implicit declaration of function foo()` 表示缺少函数原型。这意味着编译器将假设返回类型为整数。如果函数实际上返回指针，在 64 位平台上它将被截断，通常会导致分段错误。

- 不要将标准系统函数的名称用于其他任何事情。如果你想编写自己的函数，给它自己的名字，并在任何地方调用该函数。如果你希望恢复到默认系统函数，你只需要注释掉你的代码，并将你自己的名字定义为系统函数。不要反过来做。代码应该看起来像这样：

  ```text
  /* prototype part */
  #ifdef USE_OWN_GCVT
  char *foo_gcvt(double number, size_t ndigit, char *buf);
  #else
  /* include correct file */
  #include <stdlib.h>
  /* use system function */
  #define foo_gcvt  gcvt
  #endif

  /* definition part */
  #ifdef USE_OWN_GCVT
  char *foo_gcvt(double number, size_t ndigit, char *buf)
     {
     /* proper definition */
     }

  /* typical use */
  s = foo_gcvt(n, 15, b);
  ```

<h2 id="PortsOther">其他有用的提示</h2>

- `bsd.port.mk` 的最新版本设置了安装程序的路径。具体来说，它们设置 `/usr/bin` 和 `/bin` 在 `/usr/local/bin` 和 `/usr/X11R6/bin` *之前* 被搜索。
- 依赖 `bsd.port.mk` 最新版本中出现的功能是可以的，因为人们应该更新他们的整个 ports 树，包括 `bsd.port.mk`。
- 最好使用 `update-plist` 来生成和更新装箱单，而不是手动操作。你可以用 `@comment` 注释掉不需要的行。`update-plist` 可以检测大多数文件类型并正确复制大多数额外注释。它不是绝对正确的；审查它所做的更改。检查它们是否有意义，并检查任何新的示例配置文件是否应该跟随 `@sample` 注释。
- 在 OpenBSD 中，`curses.h/libcurses/libtermlib` 是“新 curses”。更改：
  `ncurses.h ==> curses.h`
  “旧 (BSD) curses”可以通过在包含 `curses.h` 之前定义 `_USE_OLD_CURSES_`（通常在 Makefile 中）并链接 `-locurses` 来获得。
- 在 OpenBSD 中，终端规则已从旧的 BSD `sgtty` 升级到更新的 POSIX `tcgetattr` 系列。避免在新代码中使用旧样式。一些代码可能将 `tcgetattr` 定义为旧 `sgtty` 的同义词，但这在 OpenBSD 上充其量只是权宜之计。`xterm` 源代码是一个不该做什么的很好的例子。尝试使你的系统功能正确：你想要一种记住终端状态的类型（可能是 typedef），你想要一个提取当前状态的函数，以及一个设置新状态的函数。修改此状态的函数更困难，因为它们往往因系统而异。此外，不要忘记你必须处理未连接到终端的情况，并且你需要处理信号：不仅是终止，还有后台 (`SIGTSTP`)。你应该始终让终端处于理智状态。在较旧的 shell（如 sh）下进行测试，该 shell 在程序终止时不会以任何方式重置终端。
- OpenBSD 包含的较新的 termcap/terminfo 和 curses 包括用于控制彩色终端的标准序列。你应该尽可能使用这些，在其他系统上恢复到标准 ANSI 颜色序列。但是，某些终端描述尚未更新，你可能需要能够手动指定这些序列。这就是 vim 处理它的方式。这并非绝对必要。除了特权程序外，通常可以通过 `TERMCAP` 变量覆盖 termcap 定义并使其正常工作。
- 信号语义很棘手，并且因系统而异。使用 `sigaction` 确保特定的语义，以及相应手册页中引用的其他系统调用。

<h2 id="PortsAvail">附加信息</h2>

- 手册页 <a href="https://man.openbsd.org/bsd.port.mk">bsd.port.mk(5)</a>。这记录了包含在每个单独 port makefile 末尾的 ports 基础设施 makefile。文件本身开头还有一些注释，但大部分有用信息现在已记录在案。
- 手册页 <a href="https://man.openbsd.org/packages-specs">packages-specs(7)</a> 和 <a href="https://man.openbsd.org/library-specs">library-specs(7)</a>。这些分别解释了软件包和库依赖项的确切语法。
- 手册页 <a href="https://man.openbsd.org/port-modules">port-modules(5)</a>。这记录了整个 ports 基础设施中使用的不同 port 模块，例如 cpan、java、python 等。
