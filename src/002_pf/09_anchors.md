## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 锚 (Anchors)</span>

---

- <a href="#intro">简介</a>
- <a href="#anchors">锚 (Anchors)</a>
- <a href="#options">锚选项</a>
- <a href="#manip">操作锚</a>

---

<h2 id="intro">简介</h2>

除了主规则集之外，PF 还可以评估子规则集。由于可以使用 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 动态操作子规则集，因此它们提供了一种动态更改活动规则集的便捷方法。正如<a href="./03_tables.md">表</a>用于保存动态地址列表一样，子规则集用于保存动态规则集。子规则集通过使用 `anchor`（锚）附加到主规则集。

锚可以嵌套，这允许将子规则集链接在一起。锚规则将相对于加载它们的锚进行评估。例如，主规则集中的锚规则将创建以主规则集为父级的锚挂载点，而使用 `load anchor` 指令从文件加载的锚规则将创建以该锚为父级的锚点。

<h2 id="anchors">锚 (Anchors)</h2>

锚是已分配名称的规则、表和其他锚的集合。当 PF 在主规则集中遇到 `anchor` 规则时，它将评估锚点内包含的规则，就像评估主规则集中的规则一样。然后，处理将在主规则集中继续，除非数据包匹配使用 `quick` 选项的过滤规则，在这种情况下，匹配将被视为最终匹配，并将中止锚和主规则集中的规则评估。

例如：

```text
block on     egress
pass  out on egress

anchor goodguys
```

此规则集在 egress 接口上为传入和传出流量设置默认拒绝策略，然后有状态地通过传出流量，并创建一个名为 `goodguys` 的锚规则。可以通过三种方法向锚填充规则：

- 使用 `load` 规则
- 使用 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a>
- 在主规则集中内联指定规则

`load` 规则使 `pfctl` 通过从文本文件读取规则来填充指定的锚。`load` 规则必须放置在 `anchor` 规则之后。

例如：

```text
anchor goodguys
load anchor goodguys from "/etc/anchor-goodguys-ssh"
```

要使用 `pfctl` 向锚添加规则，可以使用以下类型的命令：

```shell
# echo "pass in proto tcp from 192.0.2.3 to any port 22" | pfctl -a goodguys -f -
```

规则也可以保存到文本文件（并从中加载）。例如，可以将以下两行追加到 `/etc/anchor-goodguys-www` 文件中：

```text
pass in proto tcp from 192.0.2.3 to any port 80
pass in proto tcp from 192.0.2.4 to any port { 80 443 }
```

然后使用以下命令应用：

```shell
# pfctl -a goodguys -f /etc/anchor-goodguys-www
```

要直接从主规则集加载规则，请将锚规则括在大括号分隔的块中：

```text
anchor "goodguys" {
        pass in proto tcp from 192.168.2.3 to port 22
}
```

内联锚也可以包含更多锚。

```text
allow = "{ 192.0.2.3 192.0.2.4 }"

anchor "goodguys" {
  anchor {
       pass in proto tcp from 192.0.2.3 to port 80
  }
  pass in proto tcp from $allow to port 22
}
```

对于内联锚，锚的名称变为可选。请注意上例中的嵌套锚没有名称。宏 `$allow` 是在锚外部（在主规则集中）创建的，然后在锚内部使用。

可以使用与加载到主规则集中的规则相同的语法和选项将规则加载到锚中。一个警告是，除非使用内联锚，否则使用的任何<a href="./02_macros.md">宏</a>也必须在锚本身内定义。在父规则集中定义的宏在锚中是 *不可见* 的。

由于锚可以嵌套，因此可以指定评估指定锚内的所有子锚：

```text
anchor "spam/*"
```

此语法会导致评估附加到 `spam` 锚的每个锚内的每条规则。子锚将按字母顺序评估，但不会递归下降。锚总是相对于定义它们的锚进行评估。

每个锚以及主规则集都与其他规则集分开存在。对一个规则集进行的操作（例如刷新规则）不会影响其他规则集。此外，从主规则集中删除锚点不会破坏该锚或附加到该锚的任何子锚。直到使用 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 刷新其中的所有规则并且锚内没有子锚时，锚才会被销毁。

<h2 id="options">锚选项</h2>

可选地，`anchor` 规则可以使用与其他规则相同的语法指定接口、协议、源和目标地址、标记等。当给出此类信息时，仅当数据包匹配 `anchor` 规则的定义时才处理 `anchor` 规则。

例如：

```text
block          on egress
pass       out on egress
anchor ssh in  on egress proto tcp to port 22
```

锚 `ssh` 中的规则仅针对在 egress 接口上传入的、发往端口 22 的 TCP 数据包进行评估。然后像这样向 `anchor` 添加规则：

```shell
# echo "pass in from 192.0.2.10 to any" | pfctl -a ssh -f -
```

因此，即使过滤规则没有指定接口、协议或端口，主机 192.0.2.10 也只会被允许使用 SSH 连接，因为 `anchor` 规则的定义限制了它。

相同的语法可以应用于内联锚。

```text
allow = "{ 192.0.2.3 192.0.2.4 }"
anchor "goodguys" in proto tcp {
   anchor proto tcp to port 80 {
      pass from 192.0.2.3
   }
   anchor proto tcp to port 22 {
      pass from $allow
   }
}
```

<h2 id="manip">操作锚</h2>

锚的操作通过 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a> 执行。它可用于在不重新加载主规则集的情况下从锚中添加和删除规则。

要列出名为 `ssh` 的锚中的所有规则：

```shell
# pfctl -a ssh -s rules
```

要刷新同一锚中的所有规则：

```shell
# pfctl -a ssh -F rules
```

有关命令的完整列表，请参阅 <a href="https://man.openbsd.org/pfctl">pfctl(8)</a>。
