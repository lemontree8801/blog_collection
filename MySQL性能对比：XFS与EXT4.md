- [原文链接](http://dimitrik.free.fr/blog/posts/mysql-80-perf-xfs-vs-ext4.html)


# MySQL性能对比：XFS与EXT4
这篇博文被搁置了很长一段时间，主要是因为我期望观察到的问题可以很快被修复。但是随着时间流逝，问题仍然存在。我经常被问到“Dimitri，与过去您总是推荐EXT4相比，为什么您现在推荐使用XFS？”——希望以下的文章可以阐明原因，可能激发您自己做评估，以了解在您自己的负载下，这些事情在您的系统上运行情况如何。
>注意：这也将阐明为什么新的Double Write不按计划在2018年MySQL8.0出现，而是在最近才出现。

## 首先，一些背景故事
- 过去，我们一直在EXT4上观察到更好的性能和更稳定的处理
- 有很多尝试将XFS放在首位，但是，从过去来看，一旦负载受IO限制，总会有一些问题。
- 但是，自最近几年来，我们认真解决了InnoDB Double-Write（DBLWR）功能中存在的问题。
- 为什么在InnoDB中有DBLWR？
- 实际上，过去（现在仍然如此）Linux内核/文件系统层不能保证原子IO写入
- 因此，DBLWR可以保护InnoDB页写入免受部分写入的数据影响
- 例如，我们首先将页写到某个“保留位置”，一旦写入确认，我们将页写到数据文件中它自己的位置，如果中途发生崩溃，我们可以从DBLWR中重新应用页写入（如果在DBLWR写入时发生崩溃，则丢弃DBLWR）
- 然而，如果说DBLWR功能为用户数据提供了必要的安全性，但是它将页面写入延迟增加到2倍（最佳情况）甚至更多，同样，对于相同的数据发送了两倍的IO写入，闪存存储寿命缩短了两倍。
- 一段时间以来，很多闪存存储供应商希望在Linux上原子写提供解决方案（可以允许InnoDB安全禁用DBLWR）
- 但是直到现在，只有Fusion-io提供了真正安全的解决方案（通过自身的存储+自身的内核驱动+专有的文件系统“NVMFS”）
- 不幸的是，Fusion-io已经不在了，他们为Linux内核提出的补丁已经准备了5年多，仍然没有希望看到它们能被合并。

## 现在，发生了什么
- 我们为MySQL8.0的DBLWR代码已经在2018年中旬就已经就绪
- 我们正在准备最终的验证测试，以评估运行最新Linux内核的系统的预期性能收益
- 但我们根本没想到会有什么样的惊喜等着我们

## EXT4回归测试
首先，我们观察了EXT4上的回归，同时在类似的硬件系统上使用相同的MySQL二进制运行相同的负载，但是使用的是更新的内核。——这是在经过长时间的评估之后得出的结论，该结论指出，在最近的内核中，EXT4对混合的io限制负载(无论使用哪种硬件)进行了全面回归，可以用下图进行总结:
![enter image description here](http://dimitrik.free.fr/blog/perf/80_xfs_vs_ext4/img1.png)

不确定4.1.x系列内核是最后一个“运行良好”的版本，但是肯定是较新版本性能更差。
>在里斯本举行的Linux Plumbers Conference期间，Jan KARA (SUSE) 提出了宝贵意见，他怀疑问题可能来自于EXT4内部重新设计，涉及共享锁，可能无法有效处理混合读写负载（当I/O 读写同时进行）。后来被其他开发人员确认，由Alexey复现，请参阅：https://www.percona.com/blog/2019/11/12/watch-out-for-disk-i-o-performance-issues-when-running-ext4/  ——并且，此EXT4问题的“真正”修复程序预计将随内核5.6一起提供。

XFS带来了另一个惊喜。

## XFS之“谜”
下图是我们从过去观察到的EXT4和XFS在IO限制下的MySQL负载上的差异:
![enter image description here](http://dimitrik.free.fr/blog/perf/80_xfs_vs_ext4/img2.png)

请注意，这是在DBLWR关闭的情况下运行的负载，您可以想象，按任何“正常”逻辑，启用DBLWR会让事情更糟，因为：
- 我们将多写两次，尤其是页写入延迟将增加一倍（最好的情况下）
- 这会潜在降低我们IO读的速率
- (和以前一样，为了能够从存储到缓冲池(BP)中读取任何给定的页，我们必须在BP中找到一个空间(一个可用页)，但是如果大多数页都被修改(脏的)，我们必须首先刷新脏页，然后才能使它“可用” )
- 我们必须先页IO写入，才可以页IO读取。
- (由于写页的时间会增加一倍(或更多)，我们的页读取将会等待一倍(或更多)

但这是对该发生什么的“逻辑”解释，对吗？——现在，“实际情况”是什么样的？——“实践中”，我们观察到以下情况：
![enter image description here](http://dimitrik.free.fr/blog/perf/80_xfs_vs_ext4/img3.png)

**注释**：
- EXT4:在DBLWR关闭的情况下（蓝色）性能更好，开启情况下（绿色）性能较差
- XFS：在DBLWR关闭的情况下（红色）性能较差，开启情况下（黄色）性能较好
- XFS为啥会这样？
- （特别是与EXT4相比，在XFS上启用DBLWR会带来更好的性能）

长话短说，进一步的调查至少可以找到一个“变通方法”，在DBLWR关闭的情况下提高XFS的性能:
![enter image description here](http://dimitrik.free.fr/blog/perf/80_xfs_vs_ext4/img4.png)

**注释**：
- 上图表示相同的负载，但是有4种不同的参数设定
- （2个配置参数组成的4个组合）
- LRU scan depth (lru) = 1K 或者 10K （直接影响我们在一次可以写多少脏页）
- IO write threads (iow) = 16 or 4 （用于完成AIO请求的IO写线程数）
- 奇怪的是，将IO写入线程设定为4可以大大提高性能！
- 现在，为什么这确实有帮助？这是今天留下的“谜题”

**仍然有很多问题**：
- 我们是否在XFS上达到了饱和?
- 某处发生饥饿吗?
- 有办法找到这个问题的答案吗?
- 下一篇文章中描述的XFS统计数据能指出什么吗?= > http://xfs.org/index.php/Runtime_Stats

大概在一段时间内，我们将回答所有问题。但是直到今天，到此为止。

如果您使用的是比4.1更新的Linux内核，请考虑使用XFS！

正在进行中，敬请期待；-)）谢谢您使用MySQL！






