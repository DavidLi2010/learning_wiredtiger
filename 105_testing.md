WiredTiger测试
==============
WiredTiger使用一组不同的工具和测试程序来测试。

WiredTiger测试使用[Jenkins](http://jenkins-ci.org/)持续集成测试框架自动完成。

**单元测试套件**

WiredTiger的主要功能和回归测试使用Python单元测试套件（在源代码的test/suite下）。

WiredTiger的Python测试套件包含了大约10000个独立的测试，可以在所有WiredTiger支持的平台上运行。每个测试都以可重现的方式测试一个单独的操作，从而易于诊断错误。测试套件并行执行多个测试用例，能够在相当短的时间内执行完。

WiredTiger单元测试套件包括：
- WiredTiger的功能（例如，游标、事务和恢复）；
- WiredTiger的配置设置和API的组合；
- Bug的回归测试。

WiredTiger的Python测试套件使用WiredTiger的Python API和Python单元测试功能（需要至少Python 2.6）。

WiredTiger测试套件在每次提交到WiredTiger的GitHub源码树时自动执行。

**性能测试**

性能测试主要使用bench/wtperf工具完成。基于bench/wtperf/runners下的脚本，各种各样的数据库配置被执行。

WiredTiger性能测试在每次提交到WiredTiger的GitHub源码树的develop分支时自动执行，并且与前面的执行结果做比较以检测性能回归。

**压力测试**

压力测试主要使用test/format工具完成。这个测试程序随机配置数据库并运行一些随机选择的操作，使用随机选择的数量的线程。WiredTiger压力测试在WiredTiger的GitHub develop分支上持续运行。

**并发测试**

并发测试主要使用test/format工具完成。另外test/thread和test/fops测试工具测试指定的重量级的线程操作。WiredTiger并发测试在WiredTiger的GitHub develop分支上持续运行。

**静态分析**

WiredTiger静态分析使用下面3个工具完成：
- Coverity, Inc. 软件分析工具；当前结果和历史缺陷报告可以在[Coverity's WiredTiger page](https://scan.coverity.com/projects/1018)看到；
- Gimpel Software实现的UNIX lint工具[FlexeLint](http://www.gimpel.com/html/flex.htm)；
- 伊利诺斯大学LLVM项目的[Clang Static Analyzer](http://clang-analyzer.llvm.org/)。
