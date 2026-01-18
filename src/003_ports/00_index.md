## [*Open***BSD**](https://www.openbsd.org/index.html) <span style="color: red;">Porter's Handbook</span>

### [使用 Ports](./01_ports.md)
- [简介](./01_ports.md#PortsIntro)
- [获取 Ports 树](./01_ports.md#PortsFetch)
- [Ports 系统的配置](./01_ports.md#PortsConfig)
- [搜索 Ports 树](./01_ports.md#PortsSearch)
- [直接安装：一个简单的例子](./01_ports.md#PortsInstall)
- [构建后的清理](./01_ports.md#PortsClean)
- [卸载 Port 的软件包](./01_ports.md#PortsDelete)
- [使用 Flavors 和子包](./01_ports.md#PortsFlavors)
- [使用 dpb 构建多个 Ports](./01_ports.md#dpb)
- [安全更新 (-stable)](./01_ports.md#PortsSecurity)
- [软件包签名](./01_ports.md#PkgSig)
- [报告问题](./01_ports.md#Problems)
- [调试包、调试器和回溯](./01_ports.md#Backtrace)
- [帮助我们](./01_ports.md#Helping)

### [OpenBSD Ports 指南](./02_guide.md)
- [概览](./02_guide.md#Overview)
- [移植检查清单](./02_guide.md#PortsChecklist)
- [处理复杂情况](./02_guide.md#PortsComplex)
    - [了解软件](./02_guide.md#Know)
    - [找出重要选项](./02_guide.md#Figure)
    - [理想情况：MULTI_PACKAGES 和 PSEUDO_FLAVORS](./02_guide.md#Ideal)
    - [子包之间的相互依赖](./02_guide.md#Inter)
    - [真正的 FLAVORS 和 PKGNAMES](./02_guide.md#True)
- [更新 Ports](./02_guide.md#PortsUpdate)
    - [更新检查清单](./02_guide.md#UpdateChecklist)
    - [提交 Port 更新](./02_guide.md#CommitUpdates)
- [OpenBSD 移植策略](./02_guide.md#PortsPolicy)
- [安全建议](./02_guide.md#PortsSecurity)
- [通用移植提示](./02_guide.md#PortsGeneric)
- [其他有用的提示](./02_guide.md#PortsOther)
- [附加信息](./02_guide.md#PortsAvail)

### [Ports 特殊内容](./03_specialtopics.md)
- [共享库](./03_specialtopics.md#SharedLibs)
- [GNU autoconf](./03_specialtopics.md#Autoconf)
- [配置文件](./03_specialtopics.md#Config)
- [音频应用程序](./03_specialtopics.md#Audio)
- [手册页](./03_specialtopics.md#Mandoc)
- [rc.d(8) 脚本](./03_specialtopics.md#RcScripts)

### [Ports 测试指南](./04_testing.md)
- [简介](./04_testing.md#Introduction)
- [测试](./04_testing.md#Testing)
- [注释](./04_testing.md#Commenting)
- [更多测试](./04_testing.md#More)

### [与其他 BSD 项目的不同](./05_differences.md)
- [额外支持](./05_differences.md#Extra)
- [通用基础设施问题](./05_differences.md#Generic)
- [正确使用 make](./05_differences.md#Make)
- [获取源代码](./05_differences.md#Fetch)
- [`WRKDIR` 基础设施](./05_differences.md#wrkdir)
- [伪 Port 安装](./05_differences.md#Fake)
    - [简介](./05_differences.md#Introduction)
    - [优势](./05_differences.md#Advantages)
    - [如何做](./05_differences.md#How)
    - [陷阱](./05_differences.md#Pitfalls)
- [打包工具](./05_differences.md#Tools)
- [Flavors](./05_differences.md#Flavors)

---

OpenBSD Porter 手册是手册页的补充文档，最著名的是 <a href="https://man.openbsd.org/ports">ports(7)</a>。

OpenBSD 基础操作系统本身相当完整。然而，除了基础系统之外，人们可能还想使用大量的第三方软件。OpenBSD 开发者已经做了大量艰苦的工作，使成千上万的第三方应用程序可以作为<a href="../001_faq/04_package.md">预编译的二进制软件包</a>使用。

为了创建二进制软件包，开发者将从一个 `Makefile` 开始，其中包含获取、解压、修补、配置、编译和打包原始源代码的指令列表。所有这些 `Makefile` 的集合统称为 ports 系统。

在本手册中，我们将解释 ports 系统是如何工作的，并向你展示如何创建或更新你自己的 port，包括将你的工作提交回 OpenBSD 项目的准则。

为了充分利用本手册，你需要非常熟悉 pkg_* 工具和基础系统。要了解有关软件包的更多信息，请参阅 <a href="https://man.openbsd.org/packages">packages(7)</a>。

**注意：ports 和软件包集合没有经过 OpenBSD 基础系统那样的彻底安全审计。** 虽然我们努力保持软件包集合的高质量，但我们只是没有足够的人力资源来确保与基础操作系统相同水平的健壮性和安全性。
