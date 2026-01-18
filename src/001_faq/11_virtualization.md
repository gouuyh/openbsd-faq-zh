## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">FAQ - 虚拟化</span>

---

- <a href="#Introduction">简介</a>
- <a href="#Prerequisites">先决条件</a>
- <a href="#StartVm">启动虚拟机</a>
- <a href="#VMMnet">网络</a>

---

<h2 id="Introduction">简介</h2>

OpenBSD 附带了 <a href="https://man.openbsd.org/vmm">vmm(4)</a> 管理程序 (hypervisor) 和 <a href="https://man.openbsd.org/vmd">vmd(8)</a> 守护进程。可以使用 <a href="https://man.openbsd.org/vmctl">vmctl(8)</a> 控制实用程序来编排虚拟机，使用存储在 <a href="https://man.openbsd.org/vm.conf">vm.conf(5)</a> 文件中的配置设置。

提供以下功能：

- 虚拟机的串口控制台访问
- <a href="https://man.openbsd.org/tap">tap(4)</a> 接口
- 每个 VM 的用户/组所有权
- 权限分离
- raw、qcow2 和 qcow2 派生镜像
- 转储和恢复客户机系统内存
- 虚拟交换机管理
- 暂停和恢复 VM

目前不提供以下功能：

- 图形界面
- 快照
- 客户机 SMP 支持（多核）
- 硬件直通 (Hardware passthrough)
- 跨主机实时迁移
- 实时硬件变更

<!-- XXXrelease - update when vmm supports vga or more oses -->
支持的客户操作系统目前仅限于 OpenBSD 和 Linux。由于尚不支持 VGA，客户操作系统必须支持串口控制台。

<h2 id="Prerequisites">先决条件</h2>

使用 <a href="https://man.openbsd.org/vmm">vmm(4)</a> 需要支持嵌套分页 (nested paging) 的 CPU。可以通过查看处理器功能标志来检查支持情况：AMD 为 SLAT，Intel 为 EPT。在某些情况下，必须在系统 BIOS 中手动启用虚拟化功能。这样做之后，请务必运行 <a href="https://man.openbsd.org/fw_update">fw_update(8)</a> 命令以获取所需的 `vmm-firmware` 软件包。

可以使用以下命令检查处理器兼容性：

```shell
$ dmesg | egrep '(VMX/EPT|SVM/RVI)'
```

在继续之前，启用并启动 <a href="https://man.openbsd.org/vmd">vmd(8)</a> 服务。

```shell
# rcctl enable vmd
# rcctl start vmd
```

<h2 id="StartVm">启动虚拟机</h2>

在以下示例中，将创建一个具有 50GB 磁盘空间和 1GB RAM 的 VM。它将从 `install78.iso` 镜像文件启动。

```shell
# vmctl create -s 50G disk.qcow2
vmctl: qcow2 imagefile created
# vmctl start -m 1G -L -i 1 -r install78.iso -d disk.qcow2 example
vmctl: started vm 1 successfully, tty /dev/ttyp8
# vmctl show
   ID   PID VCPUS  MAXMEM  CURMEM     TTY        OWNER NAME
    1 72118     1    1.0G   88.1M   ttyp8         root example
```

要查看新创建的 VM 的控制台，请连接到其串口控制台：

```shell
# vmctl console example
Connected to /dev/ttyp8 (speed 115200)
```

需要转义序列 `~.` 来离开串口控制台。有关更多信息，请参阅 <a href="https://man.openbsd.org/cu">cu(1)</a> 手册页。当通过 SSH 使用 `vmctl` 串口控制台时，必须转义 ~ (波浪号) 字符，以防止 <a href="https://man.openbsd.org/ssh">ssh(1)</a> 断开连接。要通过 SSH 退出串口控制台，请改用 `~~.`。

可以使用 <a href="https://man.openbsd.org/vmctl">vmctl(8)</a> 停止 VM。

```shell
# vmctl stop example
stopping vm: requested to shutdown vm 1
```

无论是否有 <a href="https://man.openbsd.org/vm.conf">vm.conf(5)</a> 文件，都可以启动虚拟机。以下 `/etc/vm.conf` 示例将复制上述配置：

```text
vm "example" {
    memory 1G
    enable
    disk /home/user/disk.qcow2
    local interface
}
```

<a href="https://man.openbsd.org/vm.conf">vm.conf(5)</a> 中的某些配置属性可以由 <a href="https://man.openbsd.org/vmd">vmd(8)</a> 动态重新加载。其他更改，如调整 RAM 大小或磁盘空间，则需要重启 VM。

<h2 id="VMMnet">网络</h2>

<a href="https://man.openbsd.org/vmm">vmm(4)</a> 客户机的网络访问可以通过多种不同的方式配置，本节详细介绍了其中的四种。

在下面的示例中，将针对不同的用例提及各种 IPv4 地址范围：

- **私有地址** (<a href="https://tools.ietf.org/html/rfc1918">RFC1918</a>) 是保留给私有网络使用的，例如 `10.0.0.0/8`、`172.16.0.0/12` 和 `192.168.0.0/16`，它们不可全局路由。
- **共享地址** (<a href="https://tools.ietf.org/html/rfc6598">RFC6598</a>) 与私有地址类似，不可全局路由，但旨在用于可以执行地址转换的设备上。地址空间为 `100.64.0.0/10`。

<h3>选项 1 - VM 只需要与主机和彼此通信</h3>

对于此设置，vmm 使用 *本地接口 (local interfaces)*：使用上述定义的共享地址空间的接口。

使用 <a href="https://man.openbsd.org/vmctl">vmctl(8)</a> 的 `-L` 标志会在客户机中创建一个本地接口，该接口将通过 DHCP 从 vmd 接收地址。这本质上创建了两个接口：一个用于主机，另一个用于 VM。

<h3>选项 2 - VM 的 NAT</h3>

此设置建立在前一个设置的基础上，允许 VM 连接到主机外部。需要 <a href="https://man.openbsd.org/sysctl.2#ip.forwarding">IP 转发</a>才能使其工作。

`/etc/pf.conf` 中的以下行将启用<a href="../002_pf/05_nat.md">网络地址转换 (NAT)</a> 并将 DNS 请求重定向到指定的服务器：

```text
match out on egress from 100.64.0.0/10 to any nat-to (egress)
pass in proto { udp tcp } from 100.64.0.0/10 to any port domain \
	rdr-to $dns_server port domain
```

重新加载 pf 规则集，VM 现在可以连接到互联网了。

<h3>选项 3 - 对 VM 网络配置的额外控制</h3>

有时你可能希望对 VM 的虚拟网络进行额外的控制，例如能够将某些 VM 放在它们自己的虚拟交换机上。这可以使用 <a href="https://man.openbsd.org/veb">veb(4)</a> 和 <a href="https://man.openbsd.org/vport">vport(4)</a> 接口来完成。

创建一个 `vport0` 接口，该接口将具有如上定义的私有 IPv4 地址。在此示例中，我们将使用 `10.0.0.0/8` 子网。

```shell
# cat <<END > /etc/hostname.vport0
inet 10.0.0.1 255.255.255.0
up
END
# sh /etc/netstart vport0
```

创建 `veb0` 接口，并将 `vport0` 接口作为子接口：

```shell
# cat <<END > /etc/hostname.veb0
add vport0
up
END
# sh /etc/netstart veb0
```

如果虚拟网络上的客户机需要访问物理机以外的网络，请确保正确设置了 NAT。`/etc/pf.conf` 中调整后的 NAT 行可能如下所示：

```text
match out on egress from vport0:network to any nat-to (egress)
```

<a href="https://man.openbsd.org/vm.conf">vm.conf(5)</a> 中的以下行可用于确保定义了虚拟交换机：

```text
switch "my_switch" {
    interface veb0
}

vm "my_vm" {
    ...
    interface { switch "my_switch" }
}
```

在 `my_vm` 客户机内部，现在可以将 `vio0` 分配为 `10.0.0.0/24` 网络上的地址，并将默认路由设置为 `10.0.0.1`。

为了方便起见，你可能希望在 `vport0` 上设置 <a href="./07_networking.md#DHCP">DHCP 服务器</a>。

<h3>选项 4 - 真实网络上的 VM</h3>

在这种情况下，VM 接口将与主机所在的网络桥接。然后可以像物理连接到主机网络一样配置 VM。此选项仅适用于具有以太网连接的主机，因为 IEEE 802.11 标准阻止无线接口参与网桥。

以太网网络将使用 <a href="https://man.openbsd.org/veb">veb(4)</a> 在真实网络、主机和 VM 之间进行交换。因为 veb(4) 会将作为端口添加的接口从 IP 堆栈断开，所以真实接口上的任何 IP 配置都必须移动到 <a href="https://man.openbsd.org/vport">vport(4)</a> 接口，以便主机能够参与网络。在此示例中，`em0` 是连接到真实网络的接口。

将 IP 配置从 `em0` 移动到 `vport0`：

```shell
# mv /etc/hostname.em0 /etc/hostname.vport0
# echo up >> /etc/hostname.vport0
# echo up >> /etc/hostname.em0
# sh /etc/netstart em0 vport0
```

创建 `veb0` 接口并添加 `em0` 和 `vport0` 接口：

```shell
# cat <<END > /etc/hostname.veb0
add em0
add vport0
up
END
# sh /etc/netstart veb0
```

如前一个示例所示，创建或修改 <a href="https://man.openbsd.org/vm.conf">vm.conf(5)</a> 文件以确保定义了虚拟交换机：

```text
switch "my_switch" {
    interface veb0
}

vm "my_vm" {
    ...
    interface { switch "my_switch" }
}
```

`my_vm` 客户机现在可以像物理连接一样参与真实网络。

**注意：** 如果主机接口（上例中的 `em0`）使用自动地址配置（例如 DHCP），它可能依赖接口的 MAC 地址来获取分配的特定 IP 地址。在这种情况下，可以将 `em0` 的 MAC 地址分配给 `vport0`，以便它可以在真实网络上使用它。

通过在上述配置中省略 vport 接口，可以将虚拟机连接到真实网络但与主机隔离。
