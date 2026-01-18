## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 运行时选项</span>

选项用于控制 PF 的操作。它们在 `pf.conf` 中使用 `set` 指令指定。

<a id="block-policy"></a>
**`set block-policy option`**
<br>
设置指定了 `block` 操作的<a href="./04_filter.md">过滤</a>规则的默认行为。
- `drop` - 数据包被静默丢弃。
- `return` - 对被阻止的 TCP 数据包返回 TCP RST 数据包，对所有其他数据包返回 ICMP Unreachable 数据包。

注意，单个过滤规则可以覆盖默认响应。默认值为 `drop`。

<a id="debug"></a>
**`set debug option`**
<br>
设置 pf 的调试级别。
选项包括 `emerg`、`alert`、`crit`、`err`、`warning`、`notice`、`info` 和 `debug`。

<a id="fingerprints"></a>
**`set fingerprints file`**
<br>
设置加载操作系统指纹的文件。用于<a href="./04_filter.md#osfp">被动操作系统指纹识别</a>。默认值为 `/etc/pf.os`。

<a id="limit"></a>
**`set limit option value`**
<br>
设置 pf 操作的各种限制。可以使用 `pfctl -s memory` 查看这些值的当前设置。
- `frags` - 用于数据包重组（scrub 规则）的内存池中的最大条目数。默认值为 5000。
- `src-nodes` - 用于跟踪源 IP 地址（由 `sticky-address` 和 `source-track` 选项生成）的内存池中的最大条目数。默认值为 10000。
- `states` - 用于状态表条目（指定 `keep state` 的<a href="./04_filter.md">过滤</a>规则）的内存池中的最大条目数。默认值为 100000。
- `tables` - 可以创建的<a href="./03_tables.md">表</a>的最大数量。默认值为 1000。
- `table-entries` - 所有表中可以存储的地址总数的限制。默认值为 200000。如果系统的物理内存小于 100MB，则默认值设置为 100000。

<a id="loginterface"></a>
**`set loginterface interface`**
<br>
设置 PF 应为其收集统计信息（如输入/输出字节数和通过/阻止的数据包数）的接口。一次只能为 *一个* 接口收集统计信息。注意，无论是否设置了 `loginterface`，都会记录 `match`、`bad-offset` 等计数器和状态表计数器。要关闭此选项，请将其设置为 `none`。默认值为 `none`。

<a id="optimization"></a>
**`set optimization option`**
<br>
针对以下网络环境之一优化 PF：
- `normal` - 适用于几乎所有网络。
- `high-latency` - 高延迟网络，如卫星连接。
- `aggressive` - 积极地使状态表中的连接过期。这可以大大减少繁忙防火墙上的内存需求，但风险是可能会过早断开空闲连接。
- `conservative` - 极其保守的设置。这以更大的内存利用率和略微增加的处理器利用率为代价，避免断开空闲连接。

默认值为 `normal`。

<a id="ruleset-optimization"></a>
**`set ruleset-optimization option`**
<br>
控制 PF 规则集优化器的操作。
- `none` - 完全禁用优化器。
- `basic` - 启用以下规则集优化：
    1. 移除重复规则
    2. 移除作为另一条规则子集的规则
    3. 在有利时将多条规则合并到一个表中
    4. 重新排序规则以提高评估性能
- `profile` - 使用当前加载的规则集作为反馈配置文件，根据实际网络流量调整 quick 规则的顺序。

默认值为 `basic`。有关更完整的描述，请参阅 <a href="https://man.openbsd.org/pf.conf">pf.conf(5)</a>。

<a id="skip"></a>
**`set skip on interface`**
<br>
跳过 `interface` 上的 *所有* PF 处理。这在不需要过滤、标准化、排队等的环回接口上很有用。此选项可以使用多次。默认情况下，未设置此选项。

<a id="state-policy"></a>
**`set state-policy option`**
<br>
设置 PF 在保持状态时的行为。此行为可以在每个规则的基础上被覆盖。参见<a href="./04_filter.md#state">保持状态</a>。
- `if-bound` - 状态绑定到创建它们的接口。如果流量匹配状态表条目但没有穿过该状态条目中记录的接口，则匹配被拒绝。然后数据包必须匹配过滤规则，否则将被完全丢弃/拒绝。
- `floating` - 状态可以匹配任何接口上的数据包。只要数据包匹配状态条目并且传递方向与创建状态时在接口上的方向相同，它穿过哪个接口并不重要。它将被通过。

默认值为 `floating`。

<a id="timeout"></a>
**`set timeout option value`**
<br>
设置各种超时（以秒为单位）。
- `interval` - 清除过期状态和数据包分片之间的秒数。默认值为 `10`。
- `frag` - 未重组的分片过期前的秒数。默认值为 `30`。
- `src.track` - 在最后一个状态过期后，将<a href="./04_filter.md#stateopts">源跟踪</a>条目保留在内存中的秒数。默认值为 `0`。

示例：

```text
set timeout interval 10
set timeout frag 30
set limit { frags 5000, states 2500 }
set optimization high-latency
set block-policy return
set loginterface dc0
set fingerprints "/etc/pf.os.test"
set skip on lo0
set state-policy if-bound
```
