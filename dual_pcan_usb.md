<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# 双接口型PCAN-USB驱动

## 1、背景

* 作为半年前一篇文章[《记一次更换PCAN固件的经历》](记一次更换PCAN固件的经历.md)的续集
（重点在此文结尾的`总结`和`后续计划`章节）。

* `PEAK-System`官方论坛早年的一篇[帖子](https://forum.peak-system.com/viewtopic.php?f=59&t=1963)。

* `双`接口指的是支持**同时**使用`网络`和`字符设备`两种接口，或最低限度**不需要重新编译和切换驱动**。

## 2、实现要点

* **核心逻辑**：基于`Linux`内核主线源码（位于`drivers/net/can/usb/peak_usb`目录）和`PEAK-System`官方驱动而改写，
只有这样才能确保正确性和开发进度。在实现基本功能的基础上，其余的特性才是锦上添花。

* 程序内部的**数据和状态的共享**：更多的考虑是**软件工程**方面的设计问题，
而非底层技术（包括硬件和操作系统）。当然，理念设计和代码实现并非泾渭分明，
所以在实际的编码中往往需要将两者放到一起考虑。可简单举例以作说明：
    * 例`1`：由于`网络`接口和`字符设备`的`接收`渠道均是`USB`接口，所以一份数据就需要分发到两个地方，
    因而需要一份`URB`、两份`内部缓冲区`；而对于发送，这两个接口的数据来源不同，
    因此在数据通道容量足够的前提下，`URB`和`内部缓冲区`都是两份。
    * 例`2`：需要控制下位机的通断，但又不能在`网络`和`字符设备`两个接口各自开关的时间节点切换一次下位机状态，
    所以需要维护内部的阶段值、引用计数，这就需要借用原子性接口、锁机制等等。
    此外，需要共享的还有设备基准时间值。
    * 例`3`：两种接口对数据所用的储存方式不同——`网络`接口可沿用`sk_buff`那一套规则，
    `字符设备`则需要自行设计特定数据结构的数组或队列，并维护内部的读写指针。
    这两种方式必须能很好地与`URB`对接。

* 改进**运行效率**：`网络`接口目前看起来能优化的空间并不大，`字符设备`倒是存在一些低效操作可以优化，
可以考虑的其中一种手段是使用`内存映射`减少内核空间与用户空间的数据复制次数。

* 提升**操作便利度**（懒人刚需）：例如下位机插入即激活网络接口、通过传递模块参数值进行更多的定制化、
内核更新可自动编译（尽管官方驱动也有此特性，但它是一个版本一个压缩包，而本项目是在同一个目录下进行升级）等等。

* **避旧趋新**：最典型的例子是`procfs`接口可以完全放弃（`Linux`官方在很早之前就已提倡任何新代码不应再使用此机制，
除非需要兼容极少数旧设备，而`PEAK-System`官方驱动虽然保留此接口，但新近的上位机程序`pcanview`并不使用），
`ioctl`能免则免，这两者可考虑使用`sysfs`代替，结构清晰，代码简洁。此外，官方驱动里`单项`读写接口也没必要移植过来，
仅实现`批量`读写接口即可，有利于提升吞吐量，改进运行效率。

* 未来还可能增加更多特性，无法在此一一阐述，感兴趣可自行查阅[源码](https://github.com/FooFooDamon/dual_pcan_usb)。

## 3、对官方驱动和上位软件的若干吐槽

* 代码臃肿：一个驱动项目里完全集成了`USB`、`PCI`、`ISA`、`Dongle`等接口的`基础版`和`专业版`/`升级版`的功能，
还有很多兼容旧版逻辑的代码，即便不至于说成屎山，也是相悖于`Unix哲学`的典型。最初的想法是直接改官方代码，
但看了几天代码之后，脑壳疼，打消了这个不切实际的想法，只整理了需要用到的逻辑，另开新项目重写。顺带一提，
文首那篇官方论坛的帖子里有用户提问未来是否支持同时使用两种接口，官方回复`is not a conceivable option`，
也已暗示连作者都改不动了……

* `CPU`占用过高：`pcanview`运行起来能占一个`CPU`核`100%`的计算资源！通过`strace`命令的追踪，
发现主要耗费在`ppoll`系统调用，极有可能是将该系统调用的超时参数值设为`0`，马不停蹄的`轮询`有无数据就绪……
究竟是犯了一个低级错误，还是有什么特殊的考虑，这就不得而知了……

* 界面简陋功能不全：`Linux`版`pcanview`采用的是`字符界面`（基于`ncurses`），而不像`Windows`版那样用图形界面，
要说是为了迎合一部分命令行爱好者的口味并且顺便实现拓宽软件适用范围的效果，倒也说得过去，
但功能特性没有`Windows`版的齐全又该作何解释？

* 更新进度缓慢：这个套件应该很多年前就开始做了，但`Linux`版的`pcanview`直到`Ubuntu 22.04`还是很简陋……
个人猜想主要不是技术原因，而更多是商业上的考虑。`pcanview`是基础版软件，不收费，无利可图，
何况`Linux`版的用户很少，所以投入的资源也少，这也顺便解释了上一点的原因。

* 上位软件闭源：能理解，但不舒服。对官方来说，也是好坏参半，不需多作解释。

## 4、后续计划

* `字符设备`接口支持`发送`操作：目前仅支持`接收`操作。

* 按需添加更多的定制化参数。

* 增强驱动的健壮性，包括但不限于提高容错性、对新内核的接口变动适配。

* 研究官方驱动里的时间校正算法，看有无改进之处。

* 跟随上位机软件（即`pcanview`）的变动而及时适配，或自行开发一个新的（可能性很小，
除非极需此类分析软件且`pcanview`完全不够用）。

* 使用平价的`STM32`或类似的芯片重做一个下位机（必做，但精力所限，产出之日遥遥无期）。

* 研究更底层的`CAN`主控逻辑、此类分析软件普遍的功能特性，或升级成`CAN-FD`（需另开项目），
这些均是视工作需求或个人需要才决定做不做，时间上也会不确定。

* 以上这些将会直接体现在项目代码中，本文不再更新。
