## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">Frequently Asked Questions</span>

---

本 FAQ 是手册页（man pages）的补充文档，手册页可在安装好的系统中查看，也可以<a href="https://man.openbsd.org">在线查看</a>。
本文档涵盖了 OpenBSD 的最新发行版。
OpenBSD 的<a href="./001_faq/05_building-from-source.md#Flavors">开发版本</a>（-current）中可能包含本 FAQ 尚未涵盖的新功能和变更。

### 快速链接：

| | | |
| :--- | :--- | :--- |
| <a href="./001_faq/03_system.md#Patches">安全更新</a> | <a href="https://man.openbsd.org">手册页 (Manual Pages)</a> | <a href="./003_ports/00_index.md">Porter 手册</a> |
| <a href="./100_upgrade.md">升级到 7.8</a> | <a href="https://www.openbsd.org/mail.html">邮件列表</a> | <a href="./003_ports/04_testing.md">Port 测试指南</a> |
| <a href="./200_current.md">跟随 -current</a> | <a href="https://www.openbsd.org/report.html">报告 Bug</a> | <a href="./002_pf/00_index.md">PF 用户指南</a> |

---

### <a href="./001_faq/01_introduction.md">OpenBSD 简介</a>
*   <a href="./001_faq/01_introduction.md#WhatIs">关于 OpenBSD</a>
*   <a href="./001_faq/01_introduction.md#Platforms">硬件支持</a>
*   <a href="./001_faq/01_introduction.md#ManPages">手册页 (Manual Pages)</a>
*   <a href="./001_faq/01_introduction.md#MailLists">邮件列表</a>
*   <a href="./001_faq/01_introduction.md#OtherUnixes">迁移到 OpenBSD</a>
*   <a href="./001_faq/01_introduction.md#Bugs">报告 Bug</a>
*   <a href="./001_faq/01_introduction.md#Support">支持本项目</a>

### <a href="./001_faq/02_installation.md">安装指南</a>
*   <a href="./001_faq/02_installation.md#bsd.rd">安装过程概览</a>
*   <a href="./001_faq/02_installation.md#Checklist">安装前检查清单</a>
*   <a href="./001_faq/02_installation.md#Download">下载 OpenBSD</a>
*   <a href="./001_faq/02_installation.md#MkInsMedia">制作安装介质</a>
*   <a href="./001_faq/02_installation.md#Install">执行简单安装</a>
*   <a href="./001_faq/02_installation.md#FilesNeeded">文件集 (File Sets)</a>
*   <a href="./001_faq/02_installation.md#Partitioning">磁盘分区</a>
*   <a href="./001_faq/02_installation.md#WifiOnly">引导无线固件</a>
*   <a href="./001_faq/02_installation.md#SendDmesg">安装后发送 dmesg</a>
*   <a href="./001_faq/02_installation.md#site">自定义安装过程</a>
*   <a href="./001_faq/02_installation.md#Multibooting">多重引导</a>

### <a href="./001_faq/03_system.md">系统管理</a>
*   <a href="./001_faq/03_system.md#Patches">安全更新</a>
*   <a href="./001_faq/03_system.md#rc">系统守护进程 (System Daemons)</a>
*   <a href="./001_faq/03_system.md#doas">以其他用户身份执行命令 (doas)</a>
*   <a href="./001_faq/03_system.md#vipw">编辑密码文件</a>
*   <a href="./001_faq/03_system.md#LostPW">重置 Root 密码</a>
*   <a href="./001_faq/03_system.md#OpenNTPD">时钟同步</a>
*   <a href="./001_faq/03_system.md#TimeZone">时区</a>
*   <a href="./001_faq/03_system.md#locales">字符集和本地化</a>
*   <a href="./001_faq/03_system.md#SMT">SMT / 超线程</a>
*   <a href="./001_faq/03_system.md#SKey">使用 S/Key</a>
*   <a href="./001_faq/03_system.md#Dir">目录服务</a>

### <a href="./001_faq/04_package.md">包管理</a>
*   <a href="./001_faq/04_package.md#Intro">简介</a>
*   <a href="./001_faq/04_package.md#Mirror">选择镜像站</a>
*   <a href="./001_faq/04_package.md#PkgFind">查找软件包</a>
*   <a href="./001_faq/04_package.md#PkgInstall">安装软件包</a>
*   <a href="./001_faq/04_package.md#PkgUpdate">更新软件包</a>
*   <a href="./001_faq/04_package.md#PkgRemove">移除软件包</a>
*   <a href="./001_faq/04_package.md#PkgDup">在另一台机器上复制已安装的软件包</a>
*   <a href="./001_faq/04_package.md#PkgPartial">不完整的软件包安装或移除</a>

### <a href="./001_faq/05_building-from-source.md">从源代码构建系统</a>
*   <a href="./001_faq/05_building-from-source.md#Flavors">OpenBSD 的版本分支 (Flavors)</a>
*   <a href="./001_faq/05_building-from-source.md#Snapshots">开发快照 (Snapshots)</a>
*   <a href="./001_faq/05_building-from-source.md#Bld">从源代码构建 OpenBSD</a>
*   <a href="./001_faq/05_building-from-source.md#Release">制作发布版 (Release)</a>
*   <a href="./001_faq/05_building-from-source.md#Xbld">构建 X</a>
*   <a href="./001_faq/05_building-from-source.md#buildprobs">编译时的常见问题</a>
*   <a href="./001_faq/05_building-from-source.md#Miscellanea">杂项问题和提示</a>
*   <a href="./001_faq/05_building-from-source.md#Custom">自定义内核</a>
*   <a href="./001_faq/05_building-from-source.md#Diff">准备 Diff 文件</a>

### <a href="./001_faq/06_disk-setup.md">磁盘配置</a>
*   <a href="./001_faq/06_disk-setup.md#intro">磁盘和分区</a>
*   <a href="./001_faq/06_disk-setup.md#fdisk">使用 fdisk</a>
*   <a href="./001_faq/06_disk-setup.md#disklabel">磁盘标签 (Disk Labels)</a>
*   <a href="./001_faq/06_disk-setup.md#BootAmd64">amd64 引导过程</a>
*   <a href="./001_faq/06_disk-setup.md#altroot">Root 分区备份 (/altroot)</a>
*   <a href="./001_faq/06_disk-setup.md#DupFS">复制文件系统</a>
*   <a href="./001_faq/06_disk-setup.md#Quotas">磁盘配额</a>
*   <a href="./001_faq/06_disk-setup.md#foreignfs">访问其他文件系统</a>
*   <a href="./001_faq/06_disk-setup.md#MountImage">挂载磁盘镜像</a>
*   <a href="./001_faq/06_disk-setup.md#GrowPartition">扩容磁盘分区</a>
*   <a href="./001_faq/06_disk-setup.md#softraid">RAID 和磁盘加密</a>

### <a href="./001_faq/07_networking.md">网络</a>
*   <a href="./001_faq/07_networking.md#Setup">网络配置</a>
*   <a href="./001_faq/07_networking.md#DHCP">动态主机配置协议 (DHCP)</a>
*   <a href="./001_faq/07_networking.md#Wireless">无线网络</a>
*   <a href="./001_faq/07_networking.md#Bridge">设置网桥</a>
*   <a href="./001_faq/07_networking.md#Multipath">等价多路径路由 (ECMP)</a>
*   <a href="./001_faq/07_networking.md#NFS">使用 NFS</a>

### <a href="./001_faq/08_keyboard-and-display.md">键盘与显示控制</a>
*   <a href="./001_faq/08_keyboard-and-display.md#Consoles">在控制台工作</a>
*   <a href="./001_faq/08_keyboard-and-display.md#VGAConsoles">VGA 硬件上的控制台</a>
*   <a href="./001_faq/08_keyboard-and-display.md#Blanker">非活动控制台黑屏</a>
*   <a href="./001_faq/08_keyboard-and-display.md#SerCon">配置串口控制台</a>

### <a href="./001_faq/09_X.md">X 窗口系统</a>
*   <a href="./001_faq/09_X.md#Intro">X 简介</a>
*   <a href="./001_faq/09_X.md#ConfigX">配置 X</a>
*   <a href="./001_faq/09_X.md#StartingX">启动 X</a>
*   <a href="./001_faq/09_X.md#CustomizingX">自定义 X</a>

### <a href="./001_faq/10_multimedia.md">多媒体</a>
*   <a href="./001_faq/10_multimedia.md#enablerec">启用音频录制</a>
*   <a href="./001_faq/10_multimedia.md#confaudio">配置音频硬件</a>
*   <a href="./001_faq/10_multimedia.md#audiolevel">调整音量</a>
*   <a href="./001_faq/10_multimedia.md#usbaudio">使用 USB 音频接口</a>
*   <a href="./001_faq/10_multimedia.md#playaudio">播放音频文件</a>
<!-- XXX
<li><a href="faq13.html#playCD"     >Playing Audio CDs</a>
XXX  -->
*   <a href="./001_faq/10_multimedia.md#recordaudio">录制音频文件</a>
*   <a href="./001_faq/10_multimedia.md#recordmon">录制所有音频播放的监听混音</a>
*   <a href="./001_faq/10_multimedia.md#audiolat">降低音频延迟</a>
*   <a href="./001_faq/10_multimedia.md#audionet">使用远程音频硬件</a>
*   <a href="./001_faq/10_multimedia.md#default">选择默认音频设备</a>
*   <a href="./001_faq/10_multimedia.md#audioprob">调试音频问题</a>
*   <a href="./001_faq/10_multimedia.md#midi">使用 MIDI 乐器</a>
*   <a href="./001_faq/10_multimedia.md#webcam">使用网络摄像头</a>

### <a href="./001_faq/11_virtualization.md">虚拟化</a>
*   <a href="./001_faq/11_virtualization.md#Introduction">简介</a>
*   <a href="./001_faq/11_virtualization.md#Prerequisites">先决条件</a>
*   <a href="./001_faq/11_virtualization.md#StartVm">启动虚拟机</a>
*   <a href="./001_faq/11_virtualization.md#VMMnet">网络</a>

### <a href="./001_faq/12_vpn.md">虚拟专用网络 (VPN)</a>
*   <a href="./001_faq/12_vpn.md#introduction">简介</a>
*   <a href="./001_faq/12_vpn.md#authentication">认证</a>
*   <a href="./001_faq/12_vpn.md#server">配置 IKEv2 服务器</a>
*   <a href="./001_faq/12_vpn.md#site2site">构建站点到站点 VPN</a>
*   <a href="./001_faq/12_vpn.md#clientikev2">连接到 IKEv2 OpenBSD VPN</a>
*   <a href="./001_faq/12_vpn.md#clientikev1">连接到 IKEv1/L2TP OpenBSD VPN</a>

### <a href="./002_pf/00_index.md">PF 用户指南</a>
*   基本配置
    *   <a href="./002_pf/01_config.md">入门</a>
    *   <a href="./002_pf/02_macros.md">列表与宏</a>
    *   <a href="./002_pf/03_tables.md">表</a>
    *   <a href="./002_pf/04_filter.md">包过滤</a>
    *   <a href="./002_pf/05_nat.md">网络地址转换 (NAT)</a>
    *   <a href="./002_pf/06_rdr.md">流量重定向（端口转发）</a>
    *   <a href="./002_pf/07_shortcuts.md">创建规则集的捷径</a>
*   高级配置
    *   <a href="./002_pf/08_options.md">运行时选项</a>
    *   <a href="./002_pf/09_anchors.md">锚 (Anchors)</a>
    *   <a href="./002_pf/10_pools.md">地址池和负载平衡</a>
    *   <a href="./002_pf/11_tagging.md">数据包标记（策略过滤）</a>
*   其他内容
    *   <a href="./002_pf/12_logging.md">日志</a>
    *   <a href="./002_pf/13_perf.md">性能</a>
    *   <a href="./002_pf/14_ftp.md">FTP 相关问题</a>
    *   <a href="./002_pf/15_authpf.md">用于验证网关的用户 Shell (authpf)</a>
    *   <a href="./002_pf/16_carp.md">防火墙冗余 (CARP 和 pfsync)</a>
*   示例规则集
    *   <a href="./002_pf/17_example1.md">构建一个路由</a>

---

本 FAQ 过去的贡献者包括 Nick Holland, Steven Mestdagh, Joel Knight, Eric Jackson, Wim Vandeputte 和 Chris Cappuccio。

有关本 FAQ 的问题和评论可发送至 <a href="mailto:misc@openbsd.org">misc@openbsd.org</a>。
关于 OpenBSD 的一般性问题应发送至相应的<a href="https://www.openbsd.org/mail.html">邮件列表</a>。

<small>
$OpenBSD: index.html,v 1.561 2025/10/22 01:27:44 tj Exp $
</small>