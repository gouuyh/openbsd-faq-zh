## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 入门</span>

---

- <a href="#activate">激活</a>
- <a href="#config">配置</a>
- <a href="#control">控制</a>

---

<h2 id="activate">激活</h2>

PF 默认启用。
可以在启动时使用 <a href="https://man.openbsd.org/rcctl">rcctl(8)</a> 工具禁用它：

```shell
# rcctl disable pf
```

重启系统以使其生效。

也可以使用 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 程序手动激活和停用 PF：

```shell
# pfctl -e
# pfctl -d
```

这将分别启用和禁用 PF。
然而，启用它实际上并不加载规则集。
规则集必须单独加载，可以在启用 PF 之前或之后。

<h2 id="config">配置</h2>

PF 在引导时从 <a href="https://man.openbsd.org/pf.conf">pf.conf(5)</a> 读取其配置规则，由 <a href="https://man.openbsd.org/rc">rc(8)</a> 脚本加载。
请注意，虽然 <a href="https://man.openbsd.org/pf.conf">pf.conf(5)</a> 是默认文件并由系统 rc 脚本加载，但它只是一个文本文件，由 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 加载并解释，然后插入到 <a href="https://man.openbsd.org/pf">pf(4)</a> 中。
对于某些应用程序，引导后可能会从其他文件加载其他规则集。

`pf.conf` 文件有多个部分：

- <a href="./02_macros.md">宏 (Macros)</a>：用户定义的变量，可以保存 IP 地址、接口名称等。
- <a href="./03_tables.md">表 (Tables)</a>：用于保存 IP 地址列表的结构。
- <a href="./08_options.md">选项 (Options)</a>：控制 PF 工作方式的各种选项。
- <a href="./04_filter.md">过滤规则 (Filter Rules)</a>：允许对通过任何接口的数据包进行选择性过滤或阻止。

空行被忽略，以 `#` 开头的行被视为注释。

<h2 id="control">控制</h2>

引导后，可以使用 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 程序管理 PF 的操作。
一些示例命令包括：

```shell
# pfctl -f  /etc/pf.conf	# 加载 pf.conf 文件
# pfctl -nf /etc/pf.conf	# 解析文件，但不加载
# pfctl -sr		# 显示当前规则集
# pfctl -ss		# 显示当前状态表
# pfctl -si		# 显示过滤统计信息和计数器
# pfctl -sa		# 显示它能显示的所有信息
```

有关命令的完整列表，请参阅<a href="https://man.openbsd.org/pfctl">手册页</a>。
