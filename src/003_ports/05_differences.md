## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">Ports - 与其他 BSD 项目的不同</span>

---

- <a href="#Extra">额外支持</a>
- <a href="#Generic">通用基础设施问题</a>
- <a href="#Make">正确使用 make</a>
- <a href="#Fetch">获取源代码</a>
- <a href="#wrkdir"><code>WRKDIR</code> 基础设施</a>
- <a href="#Fake">伪 Port 安装</a>
    - <a href="#Introduction">简介</a>
    - <a href="#Advantages">优势</a>
    - <a href="#How">如何做</a>
    - <a href="#Pitfalls">陷阱</a>
- <a href="#Tools">打包工具</a>
- <a href="#Flavors">Flavors</a>

---

<h2 id="Extra">额外支持</h2>

移植基础设施在 `infrastructure/bin` 下包含几个脚本，以促进新 ports 的创建：

<dl>
<dt>check-lib-depends</dt>
<dd>通过 <code>make lib-depends-check</code> 调用，用于验证共享库依赖项。</dd>
<dt>update-patches</dt>
<dd>通过 <code>make update-patches</code> 调用，应 <b>始终</b> 用于重新生成补丁。</dd>
<dt>update-plist</dt>
<dd>通过 <code>make update-plist</code> 调用。这处理了制作准确装箱单的大部分细节。OpenBSD 装箱单与其他 BSD 项目的装箱单有很大不同，部分原因是软件包工具已被完全重写。</dd>
</dl>

检查 `infrastructure/bin` 目录以获取更多有用的脚本。它们大多数都有手册页。

<h2 id="Generic">通用基础设施问题</h2>

OpenBSD 的 <a href="https://man.openbsd.org/make">make(1)</a> 支持 `${VAR:U}` 和 `${VAR:L}` 将变量的值转换为大写或小写。因此，make 测试应以不区分大小写的方式编写。
例如：

```text
.if ${NEED_XXX:L} == "yes"
do stuff if yes
.else
do other stuff
.endif
```

理论上，`bsd.port.mk` 识别的所有布尔变量都应始终定义，因此像 `defined(USE_FOO)` 这样的代码应该是不必要的，而 `${USE_FOO:L} != "no"` 应该可以工作。

主 `bsd.port.mk` 文件已经过大量精简和修复。特别是，它已准备好并行 make。`scripts/{pre,do,post}-*` 功能在此过程中已丢失。要替换该功能，请从 `Makefile` 手动调用脚本。

<h2 id="Make">正确使用 make</h2>

请注意，如果你以 `make VAR=value` 的方式调用 make，该赋值将覆盖 `VAR` 可能从 `Makefile` 获得的任何值。这意味着许多 `Makefile` 补丁是不必要的。正确设置 `MAKE_FLAGS` 要好得多，这减少了维护负担。

<h2 id="Fetch">获取源代码</h2>

有两种源代码归档：`DISTFILES` 和 `PATCHFILES`。OpenBSD 以统一的方式处理它们，默认情况下从 `MASTER_SITES` 检索所有内容。**没有** `PATCH_SITES` 或 `PATCH_SITES_SUBDIR`。

如果所有要获取的文件并非来自同一组站点，OpenBSD 允许扩展名 `filename:0` 到 `filename:9`，在这种情况下，它将使用 `MASTER_SITES0` 到 `MASTER_SITES9` 来检索文件。

某些架构可能需要特定的 distfiles。过去，这在镜像 distfiles 方面引起了麻烦。OpenBSD 支持第三组文件：`SUPDISTFILES`。这些文件仅用于创建校验和及镜像目的。请注意，`SUPDISTFILES` 可能与 `DISTFILES` 或 `PATCHFILES` 重叠。
例如：

```text
DISTFILES=foo-1.0.tgz
.if ${ARCH} == "i386"
DISTFILES+=foo-i386.tgz
.elif ${ARCHI} == "amd64"
DISTFILES+=foo-amd64.tgz
.endif
SUPDISTFILES=foo-i386.tgz foo-amd64.tgz
```

<h2 id="wrkdir"><code>WRKDIR</code> 基础设施</h2>

我们不希望 ports 使用 `NO_WRKDIR`。所有 OpenBSD ports 必须有一个工作目录。这些工作目录的命名细节不应是 porter 关心的问题。如果你需要找出这样的名称，请询问 `Makefile`：

```shell
$ cd that_ports_dir && make show=WRKDIR
```

这将产生该 port 对 `WRKDIR` 的看法。

此禁令背后的主要原因是 OpenBSD 的 `bsd.port.mk` 就像一个真正的 `Makefile` 一样，具有依赖关系。`fetch` 阶段依赖于 distfiles 和 patchfiles，所有其他阶段都依赖于存在于工作目录中的真实文件（cookies），因此如果没有工作目录，它们就无法存在。

如果 `DISTFILES` 提取是特殊的，请设置

```text
EXTRACT_ONLY=
```

并在 `post-extract` 中进行提取。

<dl>
<dt><code>WRKDIR</code></dt>
<dd>Port 工作目录，它在其中放置自己的 cookies。</dd>

<dt><code>WRKDIST</code></dt>
<dd><code>WRKDIR</code> 的子目录，port 实际解包的位置。它也是 patch 的基础目录。其他 BSD 目前没有 <code>WRKDIST/WRKSRC</code> 的区别，只有 <code>WRKSRC</code>。</dd>

<dt><code>WRKSRC</code></dt>
<dd><code>WRKDIST</code> 的子目录，实际源代码所在的位置。</dd>

<dt><code>WRKBUILD</code></dt>
<dd><code>WRKDIR</code> 的子目录，port 配置和构建将在此处发生。其他 BSD 没有 <code>WRKBUILD/WRKSRC</code> 的区别。基于 autoconf 的程序（大多数）通常可以设置 <code>SEPARATE_BUILD</code> 以让 port 构建发生在与 <code>WRKSRC</code> 不同的 <code>WRKBUILD</code> 中。</dd>

<dt><code>WRKCONF</code></dt>
<dd><code>WRKDIR</code> 的子目录，configure 脚本应在此处运行。默认为 <code>WRKBUILD</code>，这在 99% 的情况下是正确的。</dd>

<dt><code>WRKINST</code></dt>
<dd>在打包之前 port 将被安装到的目录（见下文“伪 Port 安装”）。</dd>
</dl>

*注意 `NO_WRKSUBDIR` 已被移除：其功能可以通过设置 `WRKDIST=$(WRKDIR)` 来实现。*

<h2 id="Fake">伪 Port 安装</h2>

<h3 id="Introduction">简介</h3>

构建完成后，其他 BSD 继续安装 port，然后从已安装的 port 构建软件包。OpenBSD 改用伪装 (faked) 安装。

- OpenBSD port 被正常配置和构建（例如，安装在 `PREFIX` 下，通常是 `/usr/local`）。
- 但它被告知安装在其他地方，即 `WRKINST` 下，通常是 `WRKDIR` 的子目录。
- 然后使用 pkg_create 的 `-B` 选项打包虚假安装。
- 最后，可以使用 pkg_add 安装生成的软件包。

<h3 id="Advantages">优势</h3>

- 对于软件包构建者来说，这意味着大多数 ports 不必实际安装，这消除了大量潜在的危害和来自安装不良 ports 的普遍恶心。它还允许在同一台机器上构建多个冲突的软件包。最后，它允许构建一组新的未经测试的软件包而不破坏正确的安装。
- 对于 port 编写者来说，它极大地简化了在装箱单中查找问题的任务，因为伪安装区域在 port 安装之前是空的。此外，如果 port 安装了太多文件，不再需要调整 port 安装：只需不在装箱单中记录多余的文件即可。
- 对于最终用户来说，它提高了软件包的质量：由于最终 port 是使用 pkg_add 安装的，最终用户得到的软件与 porter 机器上准备的软件 *完全* 相同。

<h3 id="How">如何做</h3>

为 `make fake` 调用的目标是通常的 install 目标，除了几个区别：

- 使用 `FAKE_FLAGS` 代替 `MAKE_FLAGS`。默认情况下，`FAKE_FLAGS` 设置 `DESTDIR=${WRKINST}`。
- 使用 `FAKE_TARGET` 代替 `INSTALL_TARGET`。
- `{pre,do,post}-install` 片段被调用时，`TRUEPREFIX` 设置为 `$(PREFIX)`，`PREFIX` 设置为 `$(WRKINST)$(PREFIX)`，`DESTDIR` 设置为 `$(WRKINST)`。

使用 imake 的 Ports 应该按原样工作，因为 imake 片段配置为使用 `DESTDIR`。同样，最近的 GNU configure ports 应该不需要更改。

另一个好的技术是“后期绑定”技巧：配置 ports 使用 `$(DESTDIR)/usr/local` 的前缀，以便生成的 `Makefile` 具有以下设置：

```text
prefix=$(DESTDIR)/usr/local
```

当 port 构建时，由于 `DESTDIR` 设置为空，因此使用 `/usr/local`。伪安装会将所有内容放入 `${WRKINST}/usr/local`（例如，对于 GNU configure，使用 `CONFIGURE_STYLE= gnu dest`）。

<h3 id="Pitfalls">陷阱</h3>

- 一些 ports 在 `DESTDIR` 处理上不一致：port 的大部分对设置 `DESTDIR` 很满意，除了一个或两个违规者。修补问题。
- 小心区分 port 安装的实际位置和软件包配置文件中记录的位置。这很容易被忽视，但使用 `TRUEPREFIX` 很容易修复。
- 绝对符号链接总是需要调整。幸运的是，`bsd.port.mk` 会注意到该领域的问题。
- 一些 ports 不想在 configure 阶段不理会 `$(DESTDIR)`。需要一个 `post-configure` 片段来调整所有 Makefiles 以添加 `DESTDIR`。
- 极少数情况下，port 会抵制所有使用 FAKE 的合理尝试。蛮力方法应该有效：使用 `pre-fake` 链接或复制 port 想要在 `WRKINST` 区域中找到的所有内容，然后在 chroot 下执行安装。

<h2 id="Tools">打包工具</h2>

软件包工具了解相当多的文件类型，并且可以自动做很多事情：在大多数情况下，不需要 `@exec` 命令或 `INSTALL` 脚本。

请注意，所有不需要的脚本都应该被禁止，因为它们有可扩展性问题。调试单个软件包基础设施比修改数百个脚本来处理新问题要容易得多。例如：

- 不需要 `@exec ldconfig`，因为共享库用 `@lib libfoo.so.1.0` 注释，`ldconfig` 仅在需要时运行，并优雅地处理 chroot。
- 不需要 `@exec install-info`，因为 info 文档文件用 `@info file.info` 注释。这也处理了多个 info 文件，并消除了对 `makeinfo --no-split` 的需求。
- 字体通过 `@font` 和 `@fontdir` 自动集成。
- 新用户和组使用 `@newuser` 和 `@newgroup` 而不是安装脚本处理。它们也被尽早创建，以便进一步的软件包提取可以使用它们。
- 大多数第三方数据库处理通过 `@tag` 处理，这会在安装结束时触发运行诸如 update-desktop-database 之类的工具一次。
- 配置文件通过 `@sample` 而不是安装脚本处理。

有关更多详细信息，请参阅 <a href="https://man.openbsd.org/pkg_create">pkg_create(1)</a>。在大多数情况下，`make update-plist` 将编写一个非常好的完整装箱单近似值，并将手工调整从一个版本带到下一个版本。

<h2 id="Flavors">Flavors</h2>

选项已被合理化为 flavors (风味/变体)，以便软件包构建可以保持一致。带有选项的 port 应将 `FLAVORS` 设置为对该 port 有意义的所有选项列表（例如，`FLAVORS=foo bar zoinx`），然后使用 `FLAVOR` 测试实际选择了哪些选项（例如，`FLAVOR=zoinx foo`）。`bsd.port.mk` 提供了一些支持：

- `PKGNAME` 被调整为包含破折号分隔的选项（例如，`package-foo-zoinx`）。
- `WRKDIR` 被调整，以便可以并发构建不同的 flavors 而不会冲突。
- `%%flavor%%` 形式的构造将触发包含 `PFRAG.flavor`。
- `bsd.port.subdir.mk` 理解扩展名 `SUBDIR=directory,opt1,opt2`，表示“使用 `FLAVOR=opt1 opt2` 在 `directory` 中构建 port”。

检查是否选择了给定的 flavor 非常简单：

```text
.if ${FLAVOR:Mzoinx}
```

还有一个额外的扩展，称为 `MULTI_PACKAGES`。一般来说，`MULTI_PACKAGES` 和 `FLAVORS` 是正交机制。它们共同解释了为什么 OpenBSD ports 树比其他 BSD 小一些，因为它们允许一个单独的 port 目录构建许多不同的软件包。<a href="https://man.openbsd.org/bsd.port.mk">bsd.port.mk(5)</a> 有<a href="https://man.openbsd.org/bsd.port.mk#FLAVORS_AND_MULTI_PACKAGES">完整的一节</a>专门介绍 FLAVORS 和 MULTI_PACKAGES。
