# [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">FAQ - 多媒体</span>

---

- <a href="#enablerec">启用音频录制</a>
- <a href="#confaudio">配置音频硬件</a>
- <a href="#audiolevel">调整音量</a>
- <a href="#usbaudio">使用 USB 音频接口</a>
- <a href="#playaudio">播放音频文件</a>
- <a href="#recordaudio">录制音频文件</a>
- <a href="#recordmon">录制所有音频播放的监听混音</a>
- <a href="#audiolat">降低音频延迟</a>
- <a href="#audionet">使用远程音频硬件</a>
- <a href="#default">选择默认音频设备</a>
- <a href="#audioprob">调试音频问题</a>
- <a href="#midi">使用 MIDI 乐器</a>
- <a href="#webcam">使用网络摄像头</a>

---

<h2 id="enablerec">启用音频录制</h2>

出于隐私原因，OpenBSD 默认禁用音频录制。可以使用 `kern.audio.record` sysctl 来启用它。

```shell
# sysctl kern.audio.record=1
# echo kern.audio.record=1 >> /etc/sysctl.conf
```

<h2 id="confaudio">配置音频硬件</h2>

每个声卡型号都有自己的一组控件。有些根本没有控件；其他的有一百个或更多。以 root 身份运行 <a href="https://man.openbsd.org/mixerctl">mixerctl(8)</a> 将列出所有控件和当前设置。

并非每个音频芯片的每个选项都一定能到达外部世界。例如，列出的输出可能比物理连接的更多。音频设备的控件可能有不同的标签。通常控件有一个有意义的标签，但有时必须简单地尝试不同的设置，看看每个控件有什么效果。

以下是一些需要考虑的控件：

**静音和电平控制** - 即使主控件似乎设置正确，信号路径中也可能有多个静音或电平控制。在下面的示例中，设备有一个主录音电平和一个麦克风增益控制：

```text
record.adc-0:1=248,248
record.adc-0:1_source=mic
inputs.mic=85,85
```

**外部放大器断电 (EAPD)** - 此开关通常用于笔记本电脑的省电，可能需要设置才能获得输出信号：

```text
outputs.spkr_eapd=on
```

**录音源** - 某些设备有多个麦克风输入。此类控件的示例：

```text
record.source=mic
record.adc-0:1_source=mic
```

要使更改在每次重启时生效，请编辑 `/etc/mixerctl.conf` 文件。例如：

```text
record.adc-0:1_source=mic
inputs.mic=85,85
```

<h2 id="audiolevel">调整音量</h2>

<a href="https://man.openbsd.org/sndioctl">sndioctl(1)</a> 实用程序用于作为普通用户操作音频控件。不带参数运行它将列出所有控件和当前设置。

`output.level` 控件始终存在。它要么对应于硬件控件，要么在软件中模拟。

<h2 id="usbaudio">使用 USB 音频接口</h2>

dmesg 输出中的 USB 音频接口示例如下所示：

```text
uaudio0 at uhub2 port 1 configuration 1 interface 1 "ABC C-Media USB Audio Device" rev 1.10/1.00 addr 2
uaudio0: class v1, full-speed, sync, channels: 2 play, 1 rec, 8 ctls
audio1 at uaudio0
```

在大多数系统上，第一个音频设备是内部声卡。连接后，USB 音频设备将成为第二个。在默认的 <a href="https://man.openbsd.org/sndiod">sndiod(8)</a> 配置中，这两个设备对程序来说分别为 `snd/0` 和 `snd/1`，并且可以独立使用。

程序将默认使用 `snd/default`，它指向 `snd/0`。可以使用 `server.device` 快速更改此绑定，如下所示：

```shell
$ sndioctl server.device=1
```

或者，可以配置 <a href="https://man.openbsd.org/sndiod">sndiod(8)</a>，使 `snd/default` 在连接时对应于 USB 设备，未连接时对应于内部设备：

```shell
# rcctl set sndiod flags -f rsnd/0 -F rsnd/1
# rcctl restart sndiod
```

当服务器打开设备时，它将首先尝试 USB 设备。如果不存在，则使用内部设备。如果 USB 设备断开连接，sndiod 尝试使用内部声卡继续运行。如果 USB 设备再次连接，sndiod 将在下次尝试打开设备时看到它。要强制 sndiod 在设备之间切换，请重新加载服务器：

```shell
# rcctl reload sndiod
```

<h2 id="playaudio">播放音频</h2>

OpenBSD 附带 <a href="https://man.openbsd.org/aucat">aucat(1)</a>，这是一个能够播放未压缩的 WAV、AIFF 和 AU 文件的程序。它可以用于非常简单的情况或测试播放：

```shell
$ aucat -i filename.wav
```

还有许多其他播放器可作为<a href="./04_package.md">软件包</a>提供，支持其他音频格式。

<h2 id="recordaudio">录制音频</h2>

一旦使用 `kern.audio.record` sysctl 启用了录音，就可以使用 <a href="https://man.openbsd.org/aucat">aucat(1)</a> 录制未压缩的 WAV、AIFF 和 AU 文件。

```shell
$ aucat -o file.wav
```

上述命令将开始录制 WAV 格式的文件。按 CTRL+C 结束录制。

要回放文件，请运行：

```shell
$ aucat -i file.wav
```

如果录音似乎有效，但回放录音是静音的或不是预期的，混音器可能需要一些配置。确保你选择了正确的录音源并且它未静音。

如果需要，生成的 WAV 文件可以使用 ports 树中的适当程序进行压缩。或者，可以使用像 sox、ffmpeg 或 audacity 这样的 ports 来录制、处理和压缩音频文件。

<h2 id="recordmon">录制所有音频播放的监听混音</h2>

监听流记录来自所有播放设备的组合音频输出，允许你复制或保存通过音频子系统的任何内容。此功能对于截屏或任何类型的实时音频混合非常有用。

使用以下命令为 <a href="https://man.openbsd.org/sndiod">sndiod(8)</a> 创建监听子设备 `mon`：

```shell
# rcctl set sndiod flags -s default -m play,mon -s mon
# rcctl restart sndiod
```

配置你的程序从 `snd/mon` 设备录制音频，例如：

```shell
$ aucat -f snd/mon -o file.wav
```

此时，你的系统播放的任何内容都会记录在 `file.wav` 中。

<h2 id="audiolat">降低音频延迟</h2>

延迟是程序决定播放样本与用户听到样本之间的时间。由于音频数据总是被缓冲，此延迟与音频缓冲区大小成正比。建议使用以下值：

- 实时合成器：5ms。这是你按下 MIDI 键盘上的键与实际听到音符之间的时间。粗略地说，5ms 对应于声音传播 1.75m 所需的时间。
- 游戏：50ms。这是你看到事件与听到相应声音之间的时间。
- 电影播放器等：500ms 或更多。此类应用程序“预先知道”要播放的声音，并以与相应画面同时播放的方式发送音频数据。

音频缓冲区越小（以实现低延迟），发生超限/欠载的可能性就越大。缓冲区超限/欠载会导致声音卡顿。

sndiod(8) 对所有音频应用程序施加最小延迟，默认延迟为 160ms。

如果你打算使用需要较低延迟的应用程序，请使用 `-b` 选项选择所需的延迟（以帧数表示）。例如，在 48000 样本/秒时，50ms 延迟对应于：

> 48000 样本/秒 × 0.050 秒 = 2400 样本

然后执行：

```shell
# rcctl set sndiod flags -b2400
```

<h3>低延迟是否可以改善音视频同步？</h3>

不，音视频同步不需要低延迟。同步问题通常是由软件本身引起的。强制应用程序使用较小的缓冲区（通过在低延迟模式下启动 sndiod(8)）可能会在某些情况下掩盖实际问题，并给人一种软件运行得更好的感觉，但显然正确的做法是开始寻找相应的 Bug。

<h2 id="audionet">使用远程音频硬件</h2>

sndiod(8) 可以配置为接受来自网络的连接，允许其他机器也使用声卡。在带有声卡的远程系统上，运行：

```shell
# rcctl set sndiod flags -L-
```

在本地系统上，配置你的程序使用 `snd@hostname/0`，其中 "hostname" 是远程系统的地址。可以将 `AUDIODEVICE` 环境变量设置为上述值，以使远程声卡成为默认音频设备。

任何能够连接到远程主机 TCP 端口 11025 的系统都可以使用音频设备。出于隐私原因，一次只能有一个系统的用户连接到它。如果多个系统必须同时使用音频设备，则 sndio(7) 授权 cookie 必须相同。例如，将你的 `~/.sndio/cookie` 复制到可能使用音频设备的所有帐户。

为了避免故障，可以使用包过滤器对端口 11025 上的 TCP 流量进行优先级排序。在默认配置下，sndiod 将消耗大约 200kB/s 的网络带宽。

<h2 id="default">选择默认音频设备</h2>

默认音频设备通过 `AUDIODEVICE` 环境变量选择。如果未设置，则使用 `snd/default`，默认情况下指向 <a href="https://man.openbsd.org/sndiod">sndiod(8)</a> 管理的第一个音频设备。选择默认设备的最灵活方法是导出 `AUDIODEVICE`，可能是在用户的登录配置文件中。

或者，可以使用 <a href="https://man.openbsd.org/sndioctl">sndioctl(1)</a> 暴露的 `server.device` 控件在运行时更改 <a href="https://man.openbsd.org/sndiod">sndiod(8)</a> 的默认设备：

```shell
$ sndioctl server.device=1
```

更改默认音频输出设备的另一种方法是将所需设备设为 <a href="https://man.openbsd.org/sndiod">sndiod(8)</a> 管理的第一个设备。例如，要使用外部 DAC 而不是主板的板载音频，只需更改 <a href="https://man.openbsd.org/sndiod">sndiod(8)</a> 的启动标志以使用该设备：

```shell
# rcctl set sndiod flags -f rsnd/1
# rcctl restart sndiod
```

这将使第二个音频设备 (`rsnd/1`) 成为默认设备。

<h2 id="audioprob">调试音频问题</h2>

如果在播放音频时听不到任何声音，可能是混音器控件调得太低或只是静音了。请参阅<a href="#confaudio">此部分</a>配置混音器。在报告问题之前，请取消静音**所有**输入和输出。

如果你认为你的设备应该可以工作，但由于某种原因无法工作，那么是时候进行一些调试了。以下步骤可以确定数据是否正在由 DAC 处理。

```shell
# cat > /dev/audio0 < /dev/zero &
[1] 9926
# audioctl play.{bytes,errors}
play.bytes=3312000
play.errors=0
# audioctl play.{bytes,errors}
play.bytes=7065600
play.errors=0
# audioctl play.{bytes,errors}
play.bytes=9379200
play.errors=0
# kill %1
# fg %1
cat > /dev/audio0 < /dev/zero
Terminated
```

在这里我们看到处理的数据计数 `play.bytes` 每次检查时都在增加，所以数据正在流动。我们也看到设备没有任何样本欠载 (`play.errors`)。这也很好。

请注意，即使你在运行上述测试时插入了扬声器，也不应该听到任何声音。该测试向设备发送零，这对于当前支持的所有默认编码都是静音。

既然我们知道设备可以处理数据，最好再次检查混音器设置。确保所有输出和所有输入都已取消静音并且处于合理的水平。

如果此时仍然遇到问题，可能需要<a href="https://www.openbsd.org/report.html">提交 Bug 报告</a>了。除了正常的 Bug 报告信息（如完整的 dmesg 和问题描述）外，还请包括 `mixerctl -v` 的默认输出和上述 DAC 处理测试的输出。

<h2 id="midi">使用 MIDI 乐器</h2>

乐器数字接口 (MIDI) 协议提供了一种标准化且有效的方法，将音乐表演信息表示为电子数据。MIDI 数据仅包含合成器播放声音所需的指令，而不是实际的声音。

要播放 MIDI 数据，需要连接到机器 MIDI 端口的合成器。同样，要录制 MIDI 数据，需要 MIDI 乐器（如 MIDI 键盘）。高级 MIDI 乐器可能包含多个子部分（合成器、键盘、控制界面等）。它们在 OpenBSD 上显示为多个 MIDI 端口。

当你已经运行 OpenBSD 时，在 dmesg(8) 命令的输出中查找 MIDI 端口。dmesg 输出中的 MIDI 端口示例如下：

```text
umidi0 at uhub2 port 2 configuration 1 interface 0 "Roland Roland XV-2020" rev 1.10/1.00 addr 2
midi0 at umidi0: <USB MIDI I/F>
umidi1 at uhub1 port 2 configuration 1 interface 1 "Evolution Electronics Ltd. USB Keystation 61es" rev 1.00/1.25 addr 3
midi1 at umidi1: <USB MIDI I/F>
```

它显示连接了两个 <a href="https://man.openbsd.org/midi">midi(4)</a> 驱动程序，程序将其识别为：

- `midi/0` - 通过 USB 连接的合成器
- `midi/1` - MIDI 主键盘

键盘的输出可以连接到合成器的输入，如下所示：

```shell
$ midicat -q midi/0 -q midi/1
```

现在你可以在合成器上听到你在 MIDI 键盘上弹奏的内容。

<a href="https://man.openbsd.org/sndiod">sndiod(8)</a> 服务器暴露 MIDI thru 端口，允许程序相互发送 MIDI 数据。例如，如果你没有连接硬件合成器，你可以启动一个软件合成器（如 audio/fluidsynth port），然后将其用作 MIDI 输出：

```shell
$ midicat -q midi/0 -q midithru/0
```

<h2 id="webcam">使用网络摄像头</h2>

<h3 id="enablevideo">启用视频录制</h3>

出于隐私原因，OpenBSD 默认禁用视频录制。可以使用 `kern.video.record` sysctl 来启用它。

```shell
# sysctl kern.video.record=1
# echo kern.video.record=1 >> /etc/sysctl.conf
```

<h3>支持的硬件</h3>

许多遵循 USB 视频类 (UVC) 规范的网络摄像头受 <a href="https://man.openbsd.org/uvideo">uvideo(4)</a> 设备驱动程序支持，并通过 <a href="https://man.openbsd.org/video.4">video(4)</a> 层暴露给用户。

受支持的网络摄像头（或其他视频设备）在 `dmesg` 中显示如下：

```text
uvideo0 at uhub0 port 8 configuration 1 interface 0 "Azurewave Integrated Camera" rev 2.01/69.05 addr 10
video0 at uvideo0
uvideo1 at uhub0 port 8 configuration 1 interface 2 "Azurewave Integrated Camera" rev 2.01/69.05 addr 10
video1 at uvideo1
```

此设备可以通过 `/dev/video0` 访问。

一些笔记本电脑还会为红外摄像头连接第二个（不可用的）视频设备。可以使用 <a href="https://man.openbsd.org/video.1">video(1)</a> 命令找到可用的摄像头设备：

```shell
$ video -q -f /dev/video0
video device /dev/video0:
  encodings: yuy2
  frame sizes (width x height, in pixels) and rates (in frames per second):
        320x180: 30
        320x240: 30
        352x288: 30
        424x240: 30
        640x360: 30
        640x480: 30
        848x480: 20
        960x540: 15
        1280x720: 10
  controls: brightness, contrast, saturation, hue, gamma, sharpness, white_balance_temperature
$ video -q -f /dev/video1
video: /dev/video1 has no usable YUV encodings
```

默认情况下，只有 root 允许访问视频设备。必须更改设备的权限才能作为普通用户使用它：

```shell
# chown $USER /dev/video0
```

<h3 id="recvideo">录制网络摄像头流</h3>

本节使用来自 `graphics/ffmpeg` 软件包的 `ffplay` 和 `ffmpeg`。要查看给定摄像头的功能，请运行以下命令：

```shell
$ ffplay -f v4l2 -list_formats all -i /dev/video0
[...]
[video4linux2,v4l2 @ 0x921f8420800] Raw       : yuyv422 : YUYV : 640x480 320x180 320x240 352x288 424x240 640x360 848x480 960x540 1280x720
[video4linux2,v4l2 @ 0x921f8420800] Compressed:   mjpeg : MJPEG : 1280x720 320x180 320x240 352x288 424x240 640x360 640x480 848x480 960x540
```

第一行显示未压缩 YUYV 格式支持的分辨率。此格式的帧率可能非常低。第二行显示支持的 MJPEG 压缩视频分辨率，它可以提供更高的帧率。

选择一个 MJPEG 分辨率并运行以下命令进行测试：

```shell
$ ffplay -f v4l2 -input_format mjpeg -video_size 1280x720 -i /dev/video0
[...]
Input #0, video4linux2,v4l2, from '/dev/video0':B sq=    0B f=0/0
  Duration: N/A, start: 1599377893.546533, bitrate: N/A
    Stream #0:0: Video: mjpeg (Baseline), yuvj422p(pc, bt470bg/unknown/unknown), 1280x720, 30 fps, 30 tbr, 1000k tbn, 1000k tbc
```

网络摄像头流应与分辨率和帧率一起显示。

如果这有效，可以使用 ffmpeg 录制视频，如下所示：

```shell
$ ffmpeg -f v4l2 -input_format mjpeg -video_size 1280x720 -i /dev/video0 ~/video.mkv
```

按 "q" 结束录制。

<h3>控制网络摄像头设置</h3>

网络摄像头通常有亮度、对比度和其他可以使用 <a href="https://man.openbsd.org/video.1">video(1)</a> 命令更改的控件。

```shell
$ video -c
brightness=128
contrast=32
saturation=64
hue=0
gamma=120
sharpness=3
white_balance_temperature=auto
```

例如，可以将亮度设置更改为 200：

```shell
$ video brightness=200
brightness: 128 -> 200
```

可以使用 `video -d` 将所有设置恢复为默认值。

某些设置如果设置为 "auto" 值，则支持自动调整。

<h3>Web 浏览器中的网络摄像头访问</h3>

Chromium 默认可以访问 `/dev/video`。要允许 Chromium 访问其他视频设备，必须将设备路径添加到 `/etc/chromium/unveil.main` 和 `/etc/chromium/unveil.utility_video`。

Firefox 默认可以访问 `/dev/video` 和 `/dev/video0`。要允许 Firefox 访问其他视频设备，必须将设备路径添加到 `/etc/firefox/unveil.main`。

