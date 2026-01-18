## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 列表与宏</span>

---

- <a href="#lists">列表</a>
- <a href="#macros">宏</a>

---

<h2 id="lists">列表</h2>

列表允许在一条规则中指定多个类似的标准。例如，多个协议、端口号、地址等。因此，无需为每个需要阻止的 IP 地址编写一条过滤规则，而是可以通过在列表中指定 IP 地址来编写一条规则。列表是通过在 `{ }` 括号内指定项目来定义的。

当 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 在加载规则集期间遇到列表时，它会创建多条规则，列表中的每个项目对应一条。例如：

```text
block out on fxp0 from { 192.168.0.1, 10.5.32.6 } to any
```

这会扩展为：

```text
block out on fxp0 from 192.168.0.1 to any
block out on fxp0 from 10.5.32.6 to any
```

可以在一条规则中指定多个列表：

```text
match in on fxp0 proto tcp to port { 22 80 } rdr-to 192.168.0.6
block out on fxp0 proto { tcp udp } from { 192.168.0.1, 10.5.32.6 } \
   to any port { ssh https }
```

列表项之间的逗号是可选的。

列表也可以包含嵌套列表：

```text
trusted = "{ 192.168.1.2 192.168.5.36 }"
pass in inet proto tcp from { 10.10.0.0/24 $trusted } to port 22
```

请注意像下面这样的结构，被称为“否定列表”，这是一个常见的错误：

```text
pass in on fxp0 from { 10.0.0.0/8, !10.1.2.3 }
```

虽然预期的含义通常是匹配“10.0.0.0/8 内的任何地址，但 10.1.2.3 除外”，但该规则会扩展为……

```text
pass in on fxp0 from 10.0.0.0/8
pass in on fxp0 from !10.1.2.3
```

……这会匹配任何可能的地址（因为第二条规则匹配除 10.1.2.3 之外的所有地址，包括 10.0.0.0/8 之外的地址，或者如果 10.1.2.3 在 10.0.0.0/8 内，第一条规则也会匹配它）。应该使用<a href="./03_tables.md">表 (table)</a> 来代替。

<h2 id="macros">宏</h2>

宏是用户定义的变量，可以保存 IP 地址、端口号、接口名称等。宏可以降低 PF 规则集的复杂性，并且使维护变得更加容易。

宏名称必须以字母开头，并且可以包含字母、数字和下划线。宏名称不能是保留字，如 `pass`、`out` 或 `queue`。

```text
ext_if = "fxp0"

block in on $ext_if from any to any
```

这将创建一个名为 `ext_if` 的宏。在创建宏后引用它时，其名称前要加上 `$` 字符。

宏也可以扩展为列表，例如：

```text
friends = "{ 192.168.1.1, 10.0.2.5, 192.168.43.53 }"
```

宏可以递归定义。由于宏在引号内不会展开，因此必须使用以下语法：

```text
host1      = "192.168.1.1"
host2      = "192.168.1.2"
all_hosts  = "{" $host1 $host2 "}"
```

宏 `$all_hosts` 现在扩展为 192.168.1.1, 192.168.1.2。
