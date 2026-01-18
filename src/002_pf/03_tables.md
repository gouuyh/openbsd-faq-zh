## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 表</span>

---

- <a href="#intro">简介</a>
- <a href="#config">配置</a>
- <a href="#manip">使用 <code>pfctl</code> 操作</a>
- <a href="#addr">指定地址</a>
- <a href="#match">地址匹配</a>

---

<h2 id="intro">简介</h2>

表 (Table) 用于保存一组 IPv4 和/或 IPv6 地址。对表的查找非常快，并且比<a href="./02_macros.md#lists">列表</a>消耗更少的内存和处理器时间。出于这个原因，表非常适合保存大量地址，因为在包含 50,000 个地址的表中查找的时间仅比在包含 50 个地址的表中查找的时间略多。表可以通过以下方式使用：

- 规则中的源和/或目标地址
- `nat-to` 和 `rdr-to` 规则选项中的转换和重定向地址
- `route-to`、`reply-to` 和 `dup-to` 规则选项中的目标地址

表可以在 <a href="https://man.openbsd.org/pf.conf">pf.conf(5)</a> 中创建，也可以使用 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 创建。

<h2 id="config">配置</h2>

表是使用 `pf.conf` 中的 `table` 指令创建的。可以为每个表指定以下属性：

- `const` - 表一旦创建，其内容就不能更改。如果未指定此属性，即使在 <a href="https://man.openbsd.org/securelevel">securelevel(7)</a> 为 2 或更高的情况下运行，也可以随时使用 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 向表中添加或删除地址。
- `persist` - 即使没有规则引用该表，也让内核将该表保留在内存中。如果没有此属性，当引用该表的最后一条规则被刷新时，内核将自动删除该表。

示例：

```text
table <goodguys> { 192.0.2.0/24 }
table <rfc1918>  const { 192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 }
table <spammers> persist
block in on fxp0 from { <rfc1918>, <spammers> } to any
pass  in on fxp0 from <goodguys> to any
```

也可以使用否定（或“非”）修饰符指定地址，例如：

```text
table <goodguys> { 192.0.2.0/24, !192.0.2.5 }
```

`goodguys` 表现在将匹配 192.0.2.0/24 网络中的所有地址，但 192.0.2.5 除外。

请注意，表名始终包含在 `< >` 尖括号中。

表也可以从包含 IP 地址和网络列表的文本文件中填充：

```text
table <spammers> persist file "/etc/spammers"
block in on fxp0 from <spammers> to any
```

文件 `/etc/spammers` 将包含 IP 地址和/或 <a href="https://web.archive.org/web/20150213012421/http://public.swbell.net/dedicated/cidr.html">CIDR</a> 网络块的列表，每行一个。

<h2 id="manip">使用 <code>pfctl</code> 操作</h2>

可以使用 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 动态操作表。例如，要向上面创建的 `<spammers>` 表添加条目：

```shell
# pfctl -t spammers -T add 203.0.113.0/24
```

如果 `<spammers>` 表尚不存在，这也将创建它。要列出表中的地址，请运行：

```shell
# pfctl -t spammers -T show
```

`-v` 参数也可以与 `-T show` 一起使用，以显示每个表项的统计信息。要从表中删除地址，请运行：

```shell
# pfctl -t spammers -T delete 203.0.113.0/24
```

有关使用 `pfctl` 操作表的更多信息，请参阅 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 手册页。

<h2 id="addr">指定地址</h2>

除了通过 IP 地址指定外，主机还可以通过其主机名指定。当主机名解析为 IP 地址时，所有生成的 IPv4 和 IPv6 地址都将放入表中。也可以通过指定有效的接口名称、接口组或 `self` 关键字将 IP 地址输入到表中。表将分别包含分配给该接口或组的所有 IP 地址，或分配给机器的所有 IP 地址（包括环回地址）。

指定地址时的一个限制是 `0.0.0.0/0` 和 `0/0` 在表中不起作用。替代方法是硬编码该地址或使用<a href="./02_macros.md#macros">宏</a>。

<h2 id="match">地址匹配</h2>

对表的地址查找将返回最窄匹配的条目。这允许创建如下表：

```text
table <goodguys> { 172.16.0.0/16, !172.16.1.0/24, 172.16.1.100 }

block in on dc0
pass  in on dc0 from <goodguys>
```

任何通过 `dc0` 进入的数据包都将其源地址与表 `<goodguys>` 进行匹配：

- 172.16.50.5 - 最窄匹配是 172.16.0.0/16；数据包匹配表并将被通过
- 172.16.1.25 - 最窄匹配是 !172.16.1.0/24；数据包匹配表中的一个条目，但该条目被否定（使用 "!" 修饰符）；数据包不匹配表并将被阻止
- 172.16.1.100 - 精确匹配 172.16.1.100；数据包匹配表并将被通过
- 10.1.4.55 - 不匹配表并将被阻止
