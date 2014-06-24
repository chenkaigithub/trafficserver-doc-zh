

## Cache 架构

### 简介
Apache Traffic Server不仅是HTTP代理服务器, 还是HTTP缓存服务器。Traffic Server可以缓存任何字节流数据, 不过目前仅支持通过HTTP协议来传输这些字节流数据。当这样的一个字节流被缓存住时(还有相关的HTTP协议头)我们称之为缓存中的一个对象。每个对象可以通过全局唯一的缓存key的来定位。

这篇文档旨在对Traffic Server缓存系统的基本框架和实现细节进行一下描述。为了帮助理解cache系统的内部机制, 我们也会对cache系统的配置做一些简单的讨论。这篇文当对从事TrafficServer核心开发以及插件开发的人员都会很有帮助。这里假定读者已经熟悉了管理员指南这部分内容，特别是Http反向代理缓存、缓存配置以及相关的配置文件和具体的取值。

遗憾的是内部的一些术语并不是特别一致, 因此为了试图创造一些一致性，这篇文档会经常以不同的形式来使用这些术语。

### Cache的布局
接下来的章节会介绍持久化下来的cache数据是如何组织的。Traffic Server将其持久化存储设备看成常规字节的集合, 假定存储设备上没有其他结构。而且Traffic Server不会使用操作系统上面的文件系统功能, 一个文件仅仅是用来标识出一个字节集合。

#### Cache存储
Traffic Server使用的裸存储设备定义在配置文件storage.config中。文件中的每一行定义了一个具有一致性存储特征的`cache设备`。

![Two cache spans](https://docs.trafficserver.apache.org/en/latest/_images/cache-spans.png)

Traffic Server的管理员可以根据实际情况将存储空间组织成一系列分卷，这些分卷定义在volume.config配置文件中, `cache分卷`是管理存储配置的基本单位。

`Cache分卷`的容量可以通过存储百分比来定义, 也可以定义为一个绝对的数值。默认情况下，每一个`cache分卷`定义的存储空间都会分散到所有的`cache设备`中，这是出于健壮性的考虑(一个盘有问题不会影响这个`cache分卷`在其他盘上的存储空间)。`cache分卷`和`cache设备`的交集是`cache带`，每个`cache设备`会被切分成若干个`cache带`, 而每个`cache分卷`是由一系列来自不同的`cache设备`中的`cache带`组成。

如果`Cache分卷`按下面这样定义:

![volumes](https://docs.trafficserver.apache.org/en/latest/_images/ats-cache-volume-definition.png)

那么对于前面定义的`Cache设备`的实际布局将会如下所示:

![layout](https://docs.trafficserver.apache.org/en/latest/_images/cache-span-layout.png)

`Cache带`是cache设计实现过程中的最基本的单位。