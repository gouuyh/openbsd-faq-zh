## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">PF - 用户指南</span>

- 基本配置
    - [入门](./01_config.md)
    - [列表与宏](./02_macros.md)
    - [表](./03_tables.md)
    - [包过滤](./04_filter.md)
    - [网络地址转换 (NAT)](./05_nat.md)
    - [流量重定向（端口转发）](./06_rdr.md)
    - [创建规则集的捷径](./07_shortcuts.md)
- 高级配置
    - [运行时选项](./08_options.md)
    - [锚 (Anchors)](./09_anchors.md)
    - [地址池和负载平衡](./10_pools.md)
    - [数据包标记（策略过滤）](./11_tagging.md)
- 其他内容
    - [日志](./12_logging.md)
    - [性能](./13_perf.md)
    - [FTP 相关问题](./14_ftp.md)
    - [用于验证网关的用户 Shell (authpf)](./15_authpf.md)
    - [防火墙冗余 (CARP 和 pfsync)](./16_carp.md)
- 示例规则集
    - [构建一个路由](./17_example1.md)

---

包过滤（Packet Filter，以下简称 PF）是 OpenBSD 用于过滤 TCP/IP 流量和执行网络地址转换的系统。PF 还可以对 TCP/IP 流量进行标准化和调节，以及提供带宽控制和数据包优先级排序。自 OpenBSD 3.0 以来，PF 一直是 GENERIC 内核的一部分。

PF 最初由 Daniel Hartmeier 开发，现在由整个 OpenBSD 团队维护和开发。

这套文档旨在作为 OpenBSD 中使用的 PF 系统的总体介绍。即使它涵盖了 PF 的所有主要功能，它也仅旨在作为<a href="https://man.openbsd.org">手册页</a>的补充，而不是替代它们。

要全面深入地了解 PF 的功能，请从阅读 <a href="https://man.openbsd.org/pf">pf(4)</a> 手册页开始。

与 FAQ 的其余部分一样，这套文档主要面向 OpenBSD 最新版本的用户。
