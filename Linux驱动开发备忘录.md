<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# Linux驱动开发备忘录

## 1、错误码及函数

* 头文件：`<linux/err.h>`

* 错误码：EXXX，直接查看上述头文件及其包含的头文件即可，略。

* 函数举例：
    * `void* ERR_PTR(long error)`：整型转指针。
    * `long PTR_ERR(const void *ptr)`：指针转整型。
    * `bool IS_ERR(const void *ptr)`：判断一个指针是否非空但不合法。

## 2、基本数据类型

* `dev_t`、`loff_t`、`ssize_t`等：`<linux/types.h>`

* `struct {inode, file, file_operations}`：`<linux/fs.h>`

## 3、模块入口与出口

* module_init(xx_init)
    * `<linux/init.h>`
    * __init int xx_init(void)

* module_exit(xx_exit)
    * `<linux/init.h>`
    * __exit void xx_exit(void)

* MODULE_LICENSE(str)
    * `<linux/module.h>`

* MODULE_AUTHOR(str)
    * `<linux/module.h>`

## 4、调试及打印

* `printk()`、`pr_{debug,info,notice,warning,err}()`：`<linux/printk.h>`

* `dev_{dbg,info,notice,warn,err}()`：`<linux/device.h>`

## 5、设备节点

* 头文件：`<linux/device.h>`

* 数据结构：
    * `struct class`
    * `struct device`

* 类
    * 创建：`class_create()`
    * 删除：`class_destroy()`

* 设备：
    * 创建：`device_create()`。需要上述一个类指针作为参数，
    成功后会出现`/dev/xxx`文件。
    * 删除：`device_destroy()`

## 6、字符设备

* 设备号：
    * 类型：`dev_t`，其实就是一个整数类型。
    * 查看已占用的设备号：`cat /proc/devices`
    * 注册：`<linux/fs.h>`
        * ~~`register_chrdev()`：旧版接口，会占据一个主设备号以及其下的所有次设备号，~~
        ~~已不推荐使用。~~
        * `register_chrdev_region()`：已知起始设备号的情况下使用。
        起始设备号可使用`MKDEV()`宏来对主设备号和次设备号进行组装。
        `MKDEV()`在`<linux/kdev_t.h>`定义。
        * `alloc_chrdev_region()`：完全自动分配设备号。
        得到的设备号可使用`MAJOR()`和`MINOR()`宏来拆解主次设备号。
        `MAJOR()`和`MINOR()`在`<linux/kdev_t.h>`定义。
    * 注销：`<linux/fs.h>`
        * ~~`unregister_chrdev()`：旧版接口，已不推荐使用。~~
        * `unregister_chrdev_region()`：初始化时不管是调用`alloc_chrdev_region()`，
        还是`register_chrdev_region()`，均通过此函数释放。

* 数据结构：`struct cdev` from `<linux/cdev.h>`

* 操作：`<linux/cdev.h>`
    * 动态分配：`cdev_alloc()`
    * 初始化：`cdev_init()`
    * 加载：`cdev_add()`
    * 卸载：`cdev_del()`

## 7、内存操作

* 复制：
    * `copy_{from,to}_user()`：
        * 头文件：`<linux/uaccess.h>`
        * 作用：从或向用户空间复制数据
        * 返回值：成功则返回`0`，失败则返回未复制的字节数

* 其余：待补充

## 8、设备树

* 头文件：`<linux/of.h>`。`of`即`Open Firmware`，`开放性固件`。

* 数据结构：`struct device_node`、`struct property`

* 函数接口：`of_xx()`

## 9、GPIO

* 头文件：
    * `<linux/gpio.h>`：GPIO自己的接口。
    * `<linux/of_gpio.h>`：一次包含`<linux/gpio.h>`和`<linux/of.h>`，提供GPIO专属的`OF`函数。

## 10、并发与竞争

* `原子变量`：
    * **最轻量**，但仅能操作`整型`及`二进制位`。
    * 数据结构：`atomic_t` from `<linux/types.h>`
    * 操作接口：`atomic_*()` from `<linux/atomic.h>`

* `自旋锁`：
    * 原地转圈圈**空等**，占用CPU时间。适用于**短时加锁**的场景。
    要注意**加锁期间不能休眠**，因为线程在持有和获取锁的时候均是禁止内核抢占的，
    但若休眠则会被切换出去。假设一个线程获得锁并休眠，此时又有另一个线程在请求锁，
    则会把后者切换进来且在请求成功之前一直会占着CPU，这就导致前者休眠结束也不会得到调度，
    于是`死锁`现象就发生了。此外，`中断`及其`下半部`机制也能争夺调度权，
    所以**普通函数**最好在**获取锁之前先关中断**、**释放锁前后再开中断**，
    中断服务函数则只需调用简单的加解锁函数即可，`下半部`函数则调用带`_bh`后缀的锁函数。
    * 数据结构：`spinlock_t` from `<linux/spinlock_types.h>`
    * 操作接口：`spin_*()` from `linux/spinlock.h`（已包含上述头文件）

* `读写锁`：
    * **宽松版**的`自旋锁`：允许多个同时读，只有发生一个或以上的写操作时才需要加锁,
    即读与读可以并行，而读与写、写与写需要独占。其余与自旋锁类似。
    * 数据结构：`rwlock_t` from `<linux/rwlock_types.h>`
    * 操作接口：`rwlock_init()`、`{read,write}_*()` from `<linux/rwlock.h>`（
    但该头文件不能直接包含，需要使用自旋锁的头文件）

* `顺序锁`：
    * **宽松版**的`读写锁`：允许多读、多读一写（会有脏读），但写与写还是要独占。
    其余与读写锁类似。
    * 数据结构：`seqlock_t` from `<linux/seqlock.h>`
    * 操作接口：`seqlock_init()`、`{read,write}_*()` from `<linux/seqlock.h>`

* `信号量`：
    * **计数型**的锁，在等待锁的时候宿主线程会进入休眠状态，所以不耗费CPU，
    但会有上下文切换的开销，所以适用于可能会长时间占用资源且允许休眠的场合（
    中断服务函数不能休眠）。
    * 数据结构：`struct semaphore` from `<linux/semaphore.h>`
    * 操作接口：`sema_init()`、`up()`、`down()`、`down_*()` from `<linux/semaphore.h>`

* `互斥体`：
    * **锁如其名**，理念上类似于**二值型**的`信号量`，但在实现上不一定基于`信号量`。
    * 数据结构：`struct mutex` from `<linux/mutex.h>`
    * 操作接口：`mutex_*()` from `<linux/mutex.h>`

## 11、内核定时器

* `节拍`：即一秒内产生的时钟中断次数，单位是`赫兹`（`Hz`），有`100`、`200`、`1000`等值可以选择，
通常默认为`100`，在内核中有一个`unsigned long volatile jiffies`的全局变量用于保存该值，
在`<linux/jiffies.h>`中声明，该头文件同样声明了`msecs/usecs/nsecs`与`jiffies`之间转换的函数，
这些函数提供很大的便利，因为定时器函数是以`jiffies`为计时单位。

* `内核定时器`：
    * 头文件：`<linux/timer.h>`
    * 数据结构：`<struct timer_list>`
    * 操作接口：`*_timer[_*]()`
    * 使用要点：
        * 定义一个定时器变量，并对其结构体成员进行初始化，包括超时时间、回调函数等。
        * 实现回调函数。注意内核定时器是一次性的，所以要想实现周期性定时器，
        必须在回调函数里重新激活定时器。
        * 在使用时要注册该定时器变量，不用时要注销。

* 延时函数：
    * 头文件：`<linux/delay.h>`
    * `操作接口`：`{m,u,n}delay()`、`{m,s}sleep[_*]}()`

## 12、Linux中断机制

* 中断原理详见[《极简嵌入式ARM知识点——基于NXP公司i.MX6ULL处理器》](极简嵌入式ARM知识点——基于NXP公司i.MX6ULL处理器.md)。

* 操作接口详见`<linux/interrupt.h>`。

* 上半部与下半部阶段：
    * 将一个中断的处理分成两部分，以缩短中断的响应时间。
    * 时间敏感型、不能打断或与硬件有关的操作放在`上半部`，这个阶段往往只设置一些标志。
    * 耗时型、批处理的任务放到`下半部`，这个阶段才是中断处理的实际内容。在`2.5`版本之前，
    后半部（`Bottom Half`）只有一个单一的`BH`机制，此版本之后则发展出以下`3`种机制取代原先的`BH`：
        * `软中断`：类似于`操作系统信号`（`Signal`）机制，采取**静态注册**回调函数的方式。
        * `tasklet`：即小任务，采取**动态调度**的方式，与软中断相比，建议优先使用tasklet。
        * `工作队列`：专门设置一个内核线程来执行相关的操作，允许睡眠（前两种不允许），
        调试方式与`tasklet`类似。
        前两种机制的操作接口仍定义在`<linux/interrupt.h>`，工作队列接口则定义在`<linux/workqueue.h>`。

## 13、阻塞、非阻塞IO、异步通知

* `阻塞`IO涉及休眠，所以需要`等待队列`与`唤醒`，相关的数据结构和操作接口定义在`<linux/wait.h>`，
要实现的操作往往在`struct file_operations`里的`read`和`write`指针指向的函数。

* `非阻塞`IO涉及轮询，相关的数据结构和操作接口定义在`<linux/poll.h>`，
要实现的操作往往在`struct file_operations`里的`poll`指针指向的函数，
而`xx_read()`和`xx_write()`则返回`-EAGAIN`。

* `异步通知`：用于解决要求应用层不断轮询的问题，在应用层而言其实就是`操作系统信号`（`Signal`）机制
和`fcntl()`（该函数可用于开启异步通知）的使用，在驱动层则是实现`struct file_operations`里的
`fasync`指针指向的函数，相关的数据结构和操作接口（均带`fasync`字样）定义在`<linux/fs.h>`。

## 14、平台（Platform）设备驱动

* 动机：为了提高代码的复用性，在移植时少改代码，于是将驱动进行分层，见下面。

* 抽象（从上至下）：`驱动`-`总线`-`设备`：
    * `驱动`：代码逻辑因**板子**而异。
    * `设备`：代码逻辑因**器件**而异。但在引入`设备树`之后，就不必写代码以及加载`.ko`了。
    * `总线`：充当**中介**角色，`驱动`和`设备`均需向总线`注册`，`总线`会进行`匹配`，
    若匹配成功则会将它们`关联`在一起。支持哪些总线，可在`/sys/bus`目录查看。
        * 数据结构：`struct bus_type` from `<linux/device.h>`
        * 操作接口：`bus_*` from `<linux/device.h>`

* Platform`驱动`：并非一种新的驱动，而是一个通用抽象，最终还是要依赖字符驱动、块驱动等。
数据结构`struct platform_driver`和操作接口`platform_driver_{register,unregister}()`
详见`<linux/platform_device.h>`和`drivers/base/platform.c`。
基于`Platform驱动`的驱动程序的重点是实现`probe()`。

* Platform`总线`：总线不一定是实际的总线，还可以是一个虚拟层。 USB、SPI、I2C等拥有具体的总线，
但RTC、定时器等SOC内部设备不好归类，这就是`Platform`总线的由来。
实现逻辑详见`<drivers/base/platform.c>`。

* Platform`设备`：现在已由设备树代替，但此机制仍可使用，而且就算使用设备树，
其信息在解析后也会保存到`struct platform_device`的`dev`成员。
数据结构`struct platform_device`和操作接口`platform_device_{register,unregister}()`
详见`<linux/platform_device.h>`和`drivers/base/platform.c`。

## 15、杂项（Misc）设备驱动

* 动机：为了应对设备号短缺的局面。

* 特点：
    * 主设备号固定为`10`，次设备号则自定义或由系统分配。
    * 实质仍是`字符设备驱动`，并且可以自动创建设备节点，
    即`misc_register()`已封装`alloc_chrdev_region()`、
    `cdev_add()`等操作，`misc_deregister()`亦同理。

* 代码逻辑：见`<linux/miscdevice.h>`和`drivers/char/misc.c`。

## 16、输入（Input）子系统

* 动机：为了更好地统一按键、键盘、鼠标、触摸屏等输入设备的接口标准，
灵活支持连击、多点触控等常见业务场景。

* 特点：增加了一个`事件层`。

* 代码逻辑：见`<linux/input.h>`和`drivers/input/input.c`。

## 17、液晶显示屏（LCD）驱动

* 引入`帧缓冲`（`FrameBuffer`）机制。绕开虚拟内存的障碍，解决显存的分配和映射的问题，
使驱动和应用都能访问。

* 虚拟出一个`fb`字符型设备。

* Linux内核已集成LCD驱动，详见`drivers/video/fbdev/core/fbmem.c`和`drivers/video/fbdev/mxsfb.c`。

## 18、实时时钟（RTC）驱动

* 总线类型是`I2C`，设备类型则是`字符型`，应用层可通过`ioctl()`进行访问，
代码逻辑详见`<linux/rtc.h>`和`drivers/rtc/rtc-dev.c`

## 19、块设备

待补充。

## 20、网络设备（重点是有线以太网）

* 由于支持网络功能的主流芯片通常会集成`MAC`，而在实际做产品时可能会选用不同的`PHY`芯片，
所以移植的重点是`PHY`的参数调整。由于`PHY`芯片寄存器地址线宽是`5`位，
亦即支持最多`32`个寄存器，并且`IEEE`规定了前`16`个基础寄存器的功能，
所以无论采用哪个厂家的`PHY`芯片，单靠前`16`个寄存器就足以驱动起来，
亦即可以采用通用驱动代码，但要使用芯片的专用特性，就需要原厂驱动了。
关于`PHY`基础寄存器的说明，详见`IEEE Std 802.3™-2018`文档`SECTION2`第`22.2.4 Management functions`小节。

* `NAPI`机制：即`New API`，为了解决轮询延迟大、中断过多又消耗CPU（
尤其是频繁小数据包）的问题而提出，使用中断来唤醒数据接收服务程序，
在接收服务程序中则采用轮询的方法来处理数据。

* 数据结构：
    * `struct net_device`、`struct net_device_ops`等：`<linux/netdevice.h>`
    * `struct sk_buff`：`<linux/skbuff.h>`
    * `struct phy_device`：`<linux/phy.h>`

* 操作接口：
    * 带`netdev`、`netif`、`ether`的函数：：`<linux/netdevice.h>`
    * `skb`系列函数：`<linux/skbuff.h>`
    * NAPI系列函数：实现逻辑见`net/core/dev.c`，头文件则为`<linux/netdevice.h>`
    * 网络控制器代码逻辑：详见`drivers/net`目录，按所用芯片到相应的子目录下查找即可。
    * `PHY`实现逻辑在`drivers/net/phy`目录，通用驱动则是`phy_device.c`。

## 21、常见通信协议

待补充。

