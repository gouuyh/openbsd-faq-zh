## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">Ports - 测试指南</span>

---

- <a href="#Introduction">简介</a>
- <a href="#Testing">测试</a>
- <a href="#Commenting">注释</a>
- <a href="#More">更多测试</a>

---

<h2 id="Introduction">简介</h2>

<a href="./01_ports.md">ports 树</a>是一项巨大的工程，它允许 OpenBSD 用户使用第三方程序，而无需浪费时间单独修补、配置和安装每一个程序。这项工作由一群志愿者完成，他们花费时间在各种 OpenBSD 平台上移植和测试应用程序。

许多人认为他们因为知识不足而无法帮助这一过程，但这是错误的，因为他们可以帮助 porters 更好、更快地工作。测试发布在 <a href="https://www.openbsd.org/mail.html">ports 邮件列表</a>上的提交更新或新 ports。通过这样做，你可以减少提交的延迟，并增加要提交的 ports 数量。许多 ports 因为缺乏测试而未被提交。

Ports 树是针对 <a href="../001_faq/05_building-from-source.md#Flavors">-current</a> 开发的；不能保证新 ports 或更新能在其他分支上正常工作。这意味着你应该将系统和 ports 树升级到 -current。有关如何执行此操作的说明，请参阅<a href="../200_current.md">跟随 current</a> 页面。还建议你订阅 ports 和 ports-changes 邮件列表。这样，你将收到有关新 ports 或更新 ports 以及 ports 树中更改的通知。

<h2 id="Testing">测试</h2>

邮件列表上有两种类型的提交：新 ports 和更新。新 ports 通常作为 tar 包附件或 URL 发布。一个好主意是将它们解压到 `/usr/ports/mystuff/` 目录并从那里进行测试。更新通常是针对 -current ports 树的 diff，因此最好将 port 复制到 `mystuff/` 并应用 diff 以防止树损坏。如果已安装该 port 的旧版本，最好在构建新版本之前将其删除，以避免副作用，例如旧版本库中缺少符号。

需要逐步构建以验证每个目标（参见 <a href="https://man.openbsd.org/ports">ports(7)</a>）是否正确实现：

- *fetch*
    - 需要验证 distfile(s) 是否已正确下载。尝试测试指定的所有 `MASTER_SITES`，以确保它们都是有效的源。

- *checksum*
    - 验证下载的 distfiles 是否与 `distinfo` 文件中记录的校验和匹配。你可以重新运行 `make clean=dist && make makesum` 并验证没有任何更改。

- *extract*
    - `prepare` 目标会在 `extract` 之前自动调用，并且应该安装此 port 的所有 `{BUILD,LIB}_DEPENDS`（例如 bzip2）。

- *patch*
    - 补丁应该干净地应用，没有任何警告。
    - `patches/` 目录中不应留下任何 ".orig" 文件。
    - 另一个常见的错误是在补丁中包含 RCS 标签；当 port 检入存储库并扩展 RCS 标签时，这将中断。

- *configure*
    - 检查 configure 脚本是否正确检测到你平台上的功能。
    - configure 脚本不应在未在 port 中设置显式依赖项的情况下检测到系统上已安装的杂散应用程序。
    - 如果 port 无法使用基础系统中的 libtool 构建，你可以尝试设置 `USE_LIBTOOL=gnu`，它使用来自 ports 的修补版本。GNU libtool 在 OpenBSD 上因不需要的“功能”而臭名昭著，因此如果 port 无法使用基础系统中的 libtool 构建，则应修复该问题，并且在 ports `Makefile` 中应有关于原因的 XXX 注释。
    - 如果 port 使用 CMake 并且你看到 configure 阶段的不良结果，请确保另外检查或发送以下文件：
        - `${WRKBUILD}/CMakeCache.txt`
        - `${WRKBUILD}/CMakeFiles/CMakeError.log`
        - `${WRKBUILD}/CMakeFiles/CMakeOutput.log`

- *build*
    - 检查构建错误和可疑警告。
    - 关于 <a href="https://man.openbsd.org/tmpnam">tmpnam(3)</a> 问题的警告应通过使用 <a href="https://man.openbsd.org/mkstemp">mkstemp(3)</a> 解决。
    - 尝试将 `SEPARATE_BUILD` 变量设置为 'Yes' 并测试构建是否仍然有效。
    - 确保对 GNU make 的依赖确实是必要的。

- *test*
    - 检查错误（空测试意味着没问题）。
    - 如果测试具有除 `{BUILD,RUN}_DEPENDS` 中指定的依赖项之外的依赖项，请检查 `TEST_DEPENDS` 是否正确定义。

- *fake*
    - 此目标将应用程序安装到伪工作目录中，以确保所有文件都可以轻松打包而不影响基础系统。
    - port **绝不**应该将文件安装到伪目录之外，例如安装到 `/usr/local`。
    - GNU libtool 偶尔会在某些架构上的 fake 过程中遇到重新链接库的问题。
    - 检查所有文件是否以正确的所有权和权限安装。

- *port-lib-depends-check*
    - 这将检查 port 依赖的所有库是否可以通过 `LIB_DEPENDS` 或 `WANTLIB` 到达。结果应该为空。当你看到以 "Extra" 或 "Missing" 开头的行时，应检查上述变量。

- *package*
    - 如果 `pkg/PLIST*` 和/或 `pkg/PFRAG*` 错误，软件包创建可能会中断。

- *install*
    - 软件包应成功安装其打包列表中的所有文件，并具有正确的权限。特别注意设置了 setuid 位的文​​件。
    - 确保软件包 `INSTALL` 脚本正常工作，并且不覆盖 `/etc` 中的任何文件。

- *deinstall*
    - 这应该删除软件包安装的所有文件，除了 `/etc` 中的文件。
    - 软件包在运行时创建的系统文件应在 `pkg/PLIST*` 中标记为 `@extra` 和/或 `@extraunexec` 注释。

剩余的 `pkg/` 文件如 `DESCR` 和 `MESSAGE` 应检查语法和拼写错误。段落应使用 <a href="https://man.openbsd.org/fmt">fmt(1)</a> 格式化并在 80 个字符处换行。

<h2 id="Commenting">注释</h2>

在测试结束时，真正重要的事情来了：注释。即使 port 工作正常，也必须进行注释。如果我们有十个帖子说该 port 在不同架构下运行良好，那么提交就会更快完成。如果它不起作用，那么必须给出一些信息。有些工具可以帮助完成此任务，例如 <a href="https://man.openbsd.org/portslogger">portslogger(1)</a>，它就像一个“智能 tee”，将输出重定向到日志文件。

示例：

```shell
# make install 2>&1 | /usr/ports/infrastructure/bin/portslogger .
```

这会将输出重定向到位于当前目录中的日志文件。

最后，一旦发现 port 没问题，也应该测试依赖它的其他 ports，以检查它们是否仍然正常工作。`show-required-by` make 目标将有助于找到依赖于当前 port 的其他 ports。

<h2 id="More">更多测试</h2>

检查 port 的 `Makefile` 是否有正确的依赖项、拼写错误、不正确的链接、无用或缺失的变量、正确的许可和类别。那些更有技能的人可以通过检查补丁以及提供 diff 来纠正错误、添加 flavors 或其他增强功能来提供帮助。

这些 diff 应该使用 `-uNprx CVS` 选项完成。`cvs diff -uNp` 也可以用来生成针对 CVS 存储库的补丁。
