# [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">FAQ - 键盘与显示控制</span>

---

- <a href="#Consoles">在控制台工作</a>
    - <a href="#Keyboard">重映射键盘</a>
    - <a href="#ConsoleMouse">控制台鼠标支持</a>
    - <a href="#SwitchConsole">切换控制台</a>
- <a href="#VGAConsoles">VGA 硬件上的控制台</a>
    - <a href="#Scrollback">回滚缓冲区</a>
    - <a href="#AddConsole">添加更多虚拟控制台</a>
    - <a href="#Size80x50">更改控制台字体和分辨率</a>
- <a href="#Blanker">非活动控制台黑屏</a>
- <a href="#SerCon">配置串口控制台</a>

---

<h2 id="Consoles">在控制台工作</h2>

<h3 id="Keyboard">重映射键盘</h3>

<a href="https://man.openbsd.org/kbd">kbd(8)</a> 实用程序可用于更改键盘编码。大多数键盘选项可以使用 <a href="https://man.openbsd.org/wsconsctl">wsconsctl(8)</a> 实用程序进行控制。例如，要使用 <a href="https://man.openbsd.org/wsconsctl">wsconsctl(8)</a> 更改键映射，可以执行以下操作：

```shell
# wsconsctl keyboard.encoding=uk
```

将 `[Caps Lock]` 重映射为 `[Control L]`（左 Control 键）：

```shell
# wsconsctl keyboard.map+="keysym Caps_Lock = Control_L"
```

要使更改永久生效，请使用 <a href="https://man.openbsd.org/wsconsctl.conf">wsconsctl.conf(5)</a> 文件。

<h3 id="ConsoleMouse">控制台鼠标支持</h3>

对于 <a href="https://www.openbsd.org/alpha.html">alpha</a>、<a href="https://www.openbsd.org/amd64.html">amd64</a> 和 <a href="https://www.openbsd.org/i386.html">i386</a> 平台，OpenBSD 提供了 <a href="https://man.openbsd.org/wsmoused">wsmoused(8)</a>。它可以使用 <a href="https://man.openbsd.org/rcctl">rcctl(8)</a> 实用程序启用，这在 <a href="./03_system.md#rc">rc 脚本</a>的 FAQ 中也有描述。

<h3 id="SwitchConsole">切换控制台</h3>

在大多数 alpha、macppc、amd64 和 i386 系统上，OpenBSD 默认提供六个虚拟终端：`/dev/ttyC0` 到 `/dev/ttyC5`。你可以使用 `[CTRL]+[ALT]` 加上 `[F1]` 到 `[F6]` 在它们之间切换。虚拟终端 `ttyC4` 保留供 <a href="./09_X.md">X 窗口系统</a>使用。

<h2 id="VGAConsoles">VGA 硬件上的控制台</h2>

**注意：** 本节讨论 <a href="https://man.openbsd.org/vga">vga(4)</a> 驱动程序的功能。并非所有显卡都支持它们。以下说明**不适用于使用 <a href="https://man.openbsd.org/drm">drm(4)</a> 驱动程序的现代图形硬件**。

<h3 id="Scrollback">回滚缓冲区</h3>

在少数平台和硬件组合上，OpenBSD 提供控制台回滚缓冲区。这允许你查看已经滚过屏幕的信息。要在缓冲区中上下移动，请使用 `[SHIFT]+[PGUP]` 和 `[SHIFT]+[PGDN]`。你可以向上移动和查看的页面数为 8。切换控制台将清除回滚缓冲区。由于空间限制，安装内核不具备此功能。

<h3 id="AddConsole">添加更多虚拟控制台</h3>

如果你希望拥有比默认数量更多的虚拟控制台，请使用 <a href="https://man.openbsd.org/wsconscfg">wsconscfg(8)</a> 命令为 `ttyC6`、`ttyC7` 及以上创建屏幕。例如：

```shell
# wsconscfg -t 80x25 6       # 这在从使用 drm(4) 的系统上不起作用
```

这将为 `ttyC6` 创建一个虚拟终端，通过 `[CTRL]+[ALT]+[F7]` 访问。要在新创建的虚拟控制台上获得 `login:` 提示符，你需要在 <a href="https://man.openbsd.org/ttys">ttys(5)</a> 中将其设置为 `on`，然后重启或使用 <a href="https://man.openbsd.org/kill">kill(1)</a> 向 <a href="https://man.openbsd.org/init">init(8)</a> 发送 HUP 信号。如果你希望下次启动计算机时拥有额外的屏幕，请将此命令添加到 <a href="https://man.openbsd.org/rc.local">rc.local(8)</a>。

<h3 id="Size80x50">更改控制台字体和分辨率</h3>

alpha、amd64 和 i386 上的许多 VGA 显卡能够显示 50 行而不是通常的 25 行的更高文本分辨率。标准的 80x25 文本屏幕使用 8x16 像素字体。要使行数加倍，我们首先使用 <a href="https://man.openbsd.org/wsfontload">wsfontload(8)</a> 加载 8x8 像素字体。然后我们使用 <a href="https://man.openbsd.org/wsconscfg">wsconscfg(8)</a> 删除并重新创建具有所需屏幕分辨率的虚拟控制台。这可以通过将以下命令添加到 <a href="https://man.openbsd.org/rc.local">rc.local(8)</a> 脚本的末尾来在引导时自动完成：

```shell
wsfontload -h 8 -e ibm /usr/share/misc/pcvtfonts/vt220l.808	# 加载 8x8 字体
wsconscfg -dF 5			# 删除通过 [CTRL]+[ALT]+[F6] 访问的屏幕 5
wsconscfg -t 80x50 5		# 添加具有 50 行 80 个字符的屏幕 5
```

如果你希望修改其他屏幕，只需对你希望以 80x50 分辨率运行的任何屏幕重复删除和添加屏幕步骤。无法更改通过 `[CTRL]+[ALT]+[F1]` 访问的主控制台设备 `ttyC0` 的分辨率。避免更改屏幕 4，它被 X 用作图形屏幕。

<h2 id="Blanker">非活动控制台黑屏</h2>

如果你希望在不使用 X 的情况下在一段时间不活动后使控制台黑屏，请修改以下 <a href="https://man.openbsd.org/wscons">wscons(4)</a> 变量：

- **`display.screen_off`** 决定黑屏时间（以毫秒为单位）。
- **`display.kbdact`** 如果设置为 `on`，键盘活动将唤醒屏幕。
- **`display.msact`** 如果设置为 `on`，<a href="#ConsoleMouse">控制台鼠标</a>活动将唤醒屏幕。
- **`display.outact`** 如果设置为 `on`，屏幕输出将唤醒屏幕。
- **`display.vblank`** 如果设置为 `on`，将禁用垂直同步脉冲。这将导致许多显示器进入节能模式。

例如：

```shell
# wsconsctl display.screen_off=60000
display.screen_off -> 60000
```

通过编辑 <a href="https://man.openbsd.org/wsconsctl.conf">wsconsctl.conf(5)</a> 来永久设置它们。当 `display.kbdact` 或 `display.outact` 设置为 `on` 时，黑屏程序被激活。请注意，这两个必须有一个是 `off`。

<h2 id="SerCon">配置串口控制台</h2>

OpenBSD 在大多数平台上支持串口控制台，但各平台之间的细节差异很大。除了允许用户登录外，它们还用于记录控制台输出。

要在 OpenBSD 系统上获得功能齐全的串口控制台，需要两个部分：

- 启用串口用作交互式终端，以便用户在运行多用户模式时可以登录。
- 配置 OpenBSD 使用你的串口作为状态和单用户模式的控制台。这部分非常依赖于平台。

<h3 id="SerContty">更改 <code>/etc/ttys</code> 以获得登录提示符</h3>

终端会话由 <a href="https://man.openbsd.org/ttys">ttys(5)</a> 文件控制。在 OpenBSD 在设备上给你 `login:` 提示符之前，必须在 `/etc/ttys` 中启用它。在通常连接键盘和屏幕的平台上，串口终端默认是禁用的。我们将以 amd64 平台为例。在这种情况下，你必须编辑如下行：

```text
tty00   "/usr/libexec/getty std.9600"   unknown off
```

改为类似：

```text
tty00   "/usr/libexec/getty std.9600"   vt220   on secure
```

这里，`tty00` 是我们用作控制台的串口，`vt220` 是与你的终端匹配的 <a href="https://man.openbsd.org/termcap.5">termcap(5)</a> 条目。其他可能的选项可能包括 `vt100`、`xterm` 等。`on` 位通过为该串口激活 <a href="https://man.openbsd.org/getty">getty(8)</a> 来启用登录提示符。`secure` 位允许 `root` 在此控制台登录。`9600` 位是终端波特率。

请注意，你可以在不做此步骤的情况下使用串口控制台进行安装，因为系统运行在单用户模式下，不使用 `getty` 进行登录。

在某些平台和某些配置上，如果串口控制台是你唯一可用的控制台，你必须将系统启动到单用户模式来进行此更改。

<h3>amd64 和 i386</h3>

要配置引导过程使用串口作为控制台，你的 <a href="https://man.openbsd.org/amd64/boot.conf">boot.conf(5)</a> 文件应包含以下行：

```text
set tty com0
```

此文件放在你的引导驱动器上，这也可能是你的安装介质。如果你需要非 9600bps 的波特率，请使用 `stty` 选项。

有些系统可能能够在机器中没有显卡的情况下运行，但肯定不是全部——许多系统认为这是一种错误状态。其他系统能够通过配置选项将所有 BIOS 键盘和屏幕活动重定向到串口，因此可以通过串口完全维护机器。结果可能会有所不同。使用此功能时，某些 BIOS 实现可能会阻止引导加载程序看到串口，从而导致内核不会被告知使用它。可能有一个 BIOS 选项 "Continue Console Redirection after POST"（POST 后继续控制台重定向）。这应该设置为 OFF，以便引导加载程序和内核可以处理它们自己的控制台。

要在多用户模式下使用机器，你需要按照<a href="#SerContty">上文所述</a>编辑 `/etc/ttys`。

<h3>sparc64</h3>

这些机器设计为完全可以通过串口控制台进行维护。只需从机器上移除键盘，系统就会运行串口模式。

在某些系统上，串口标记为 `ttya`、`ttyb`、`ttyh0` 或 `ttyh1`。在多用户模式下使用串口控制台无需对 `/etc/ttys` 进行任何更改。

一些 sparc64 系统将控制台端口上的 BREAK 信号解释为与 STOP-A 命令相同。这将系统踢回 Forth 提示符，停止任何应用程序和操作系统。这在需要时很方便，但不幸的是，一些串口终端在断电时以及一些 RS-232 切换设备会发送计算机解释为中断信号的东西，从而停止机器。在投入生产之前进行测试。

如果你连接了键盘和显示器，你仍然可以通过在 `ok` 提示符下使用以下命令强制使用串口控制台：

```text
ok setenv input-device ttya
ok setenv output-device ttya
ok reset
```

如果 `ttyC0` 在 `/etc/ttys` 中处于活动状态（如<a href="#SerContty">上文所述</a>），你可以在 X 中使用键盘和显示器。

<h3>macppc</h3>

macppc 机器通过 OpenFirmware 配置串口控制台。使用命令：

```text
ok setenv output-device scca
ok setenv input-device scca
ok reset-all
```

将你的串口控制台设置为 57600bps，8N1。

不幸的是，大多数 MacPPC 无法直接使用串口控制台。虽然这些机器大多数确实有串口硬件，但在机器外部无法访问。幸运的是，一些公司为几种 Macintosh 型号提供附加设备，使该端口可用作串口控制台。

你必须在单用户模式下将 `/etc/ttys` 中的 `tty00` 更改为 `on` 并将速度设置为 57600 而不是默认的 9600（如<a href="#SerContty">上文</a>详述），然后才能引导多用户模式并使串口控制台正常工作。

<h3>尝试使用 tty 设备时出现输入/输出错误</h3>

对于从 OpenBSD 系统发起的连接，你需要使用 `/dev/cuaXX`。`/dev/ttyXX` 设备仅用于终端或拨入使用。有关更多详细信息，请参阅 <a href="https://man.openbsd.org/cua">cua(4)</a> 手册。
