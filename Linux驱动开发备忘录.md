<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<base target="_blank" />

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

* `struct list_head`：`<linux/list.h>`

* 其他：用到时再补充。

## 3、模块入口与出口

* `module_init`(xx_init)
    * `<linux/init.h>` or `<linux/module.h>` depending on the `MODULE` macro
    * __init int xx_init(void)

* `module_exit`(xx_exit)
    * `<linux/init.h>` or `<linux/module.h>` depending on the `MODULE` macro
    * __exit void xx_exit(void)

* `MODULE_LICENSE`(str)
    * `<linux/module.h>`

* `MODULE_AUTHOR`(str)
    * `<linux/module.h>`

* `*_initcall[_sync]`(fn)
    * 一种可在不同内核启动阶段调用相应函数的机制
    * 有多种级别，这些级别也反映了被调用函数的顺序和所存放的`ELF`段
    * 仅适用于保证内核正常启动的必要内置模块（built-in modules，即静态链接在内核镜像中的模块），
    对于可动态加载和卸载的模块则要用`module_init`()，不重要的内置模块也可用`module_init`()或顺序靠后的`initcall`
    * 头文件：`<linux/init.h>` or `<linux/module.h>` depending on the `MODULE` macro
    * 目前可用的`initcall`：
        ````
        //
        // A "pure" initcall has no dependencies on anything else, and purely
        // initializes variables that couldn't be statically initialized.
        //
        #define pure_initcall(fn)               __define_initcall(fn, 0)

        #define core_initcall(fn)               __define_initcall(fn, 1)
        #define core_initcall_sync(fn)          __define_initcall(fn, 1s)
        #define postcore_initcall(fn)           __define_initcall(fn, 2)
        #define postcore_initcall_sync(fn)      __define_initcall(fn, 2s)
        #define arch_initcall(fn)               __define_initcall(fn, 3)
        #define arch_initcall_sync(fn)          __define_initcall(fn, 3s)
        #define subsys_initcall(fn)             __define_initcall(fn, 4)
        #define subsys_initcall_sync(fn)        __define_initcall(fn, 4s)
        #define fs_initcall(fn)                 __define_initcall(fn, 5)
        #define fs_initcall_sync(fn)            __define_initcall(fn, 5s)
        #define rootfs_initcall(fn)             __define_initcall(fn, rootfs)
        #define device_initcall(fn)             __define_initcall(fn, 6)
        #define device_initcall_sync(fn)        __define_initcall(fn, 6s)
        #define late_initcall(fn)               __define_initcall(fn, 7)
        #define late_initcall_sync(fn)          __define_initcall(fn, 7s)
        ````

## 4、调试及打印

* `printk()`、`pr_{debug,info,notice,warning,err}()`：`<linux/printk.h>`

* `dev_{dbg,info,notice,warn,err}()`：`<linux/device.h>`

* 更多的接口详见“[懒编程秘笈](https://github.com/FooFooDamon/lazy_coding_skills)”项目的`c_and_cpp/native/klogging.h`文件。

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

* 数据结构：`struct cdev`，位于`<linux/cdev.h>`

* 操作：`<linux/cdev.h>`
    * 动态分配：`cdev_alloc()`
    * 初始化：`cdev_init()`
    * 加载：`cdev_add()`
    * 卸载：`cdev_del()`

## 7、内存操作

* 分配及释放：
    * 函数：`kmalloc()`、`kfree()`，位于`<linux/slab.h>`
    * 标志：`GFP_ATOMIC`、`GFP_KERNEL`、……，位于`<linux/gfp.h>`

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

* 管脚复用——以i.MX6ULL为例：
    * 头文件：`arch/arm/boot/dts/imx6ul*-pinfunc*.h`
    * 复用配置项格式：`MX6UL_PAD_*`或`MX6ULL_PAD_*`的格式为：
    `<mux_reg conf_reg input_reg mux_mode input_val>`：
        * <font color="red">mux_reg</font>：`复用寄存器`的地址**偏移量**，至于**基址**，
        则可通过`iomuxc`或`iomuxc_snvs`节点的`reg`属性指定，即`iomuxc`基址为`0x020E0000`，
        `iomuxc_snvs`则为`0x02290000`。对应到官方文档的寄存器定义，则名为`IOMUXC[_SNVS]_SW_MUX_CTL_PAD_*`。
        * `conf_reg`：`电气特性配置寄存器`的地址**偏移量**，**基址**同上。
        对应到官方文档的寄存器定义，则名为`IOMUXC[_SNVS]_SW_PAD_CTL_PAD_*`。
        值要根据功能及应用场景手动指定，寄存器内各二进制位的定义（`DRAM`和`GRP`除外）如下：
            * `bit[31:17]`：预留未用。
            * `bit[16]`：`HYS`（`Hysteresis`），是否开启`磁滞`特性，
            在管脚设置成`输入`时开启该特性可以**过滤掉部分电磁干扰**：
                * `0`：关闭。
                * `1`：开启。
            * `bit[15:14]`：`PUS`（`Pull Select?`），`上下拉电阻`的选择：
                * `00`：`100K`欧`下`拉。
                * `01`：`47K`欧`上`拉。
                * `10`：`100K`欧`上`拉。
                * `11`：`22K`欧`上`拉。
            * `bit[13]`：`PUE`（`Pull E???`），`上下拉或保持`的选择，
            其中`保持`特性是指当断电之后此管脚能保持住断电前的状态，对`输入`和`输出`管脚均有意义：
                * `0`：保持。
                * `1`：上下拉。
            * `bit[12]`：`PKE`（`Pull/Keep Enable`），是否开启上下拉/保持特性：
                * `0`：关闭。
                * `1`：开启。
            * `bit[11]`：`ODE`（`Open Drain Enable`），是否开启`开漏输出`特性。
            在管脚设置成`输出`时才有意义：
                * `0`：关闭。
                * `1`：开启。
            * `bit[10:8]`：预留未用。
            * `bit[7:6]`：`SPEED`，速度：
                * `00`：低速（`50MHz`）。
                * `01`：中速（`100MHz`）。
                * `10`：中速（`100MHz`）。
                * `11`：最高速（`200MHz`）。
            * `bit[5:3]`：`DSE`（`Drive Strength E???`），输出驱动能力，
            在管脚设置成`输出`时才有意义，且电阻越小，驱动能力越强：
                * `000`：关闭输出驱动能力。
                * `001`：`R0`（3.3V配260欧；1.8V配150欧；DDR内存则使用240欧）。
                * `010`：`R0/2`（电阻为`R0`的`1/2`，以下同理）。
                * `011`：`R0/3`。
                * `100`：`R0/4`。
                * `101`：`R0/5`。
                * `110`：`R0/6`。
                * `111`：`R0/7`。
            * `bit[2:1]`：预留未用。
            * `bit[0]`：`SRE`（`Slew Rate E???`），电平转换（翻转）速率，
            快翻转则波形更陡，慢则更平缓：
                * `0`：慢速。
                * `1`：快速。
        * <font color="green">input_reg</font>：`输入寄存器`的地址**偏移量**，为`0`则表示无输入寄存器，
        **基址**同`iomuxc`。对应到官方文档的寄存器定义，则名为`IOMUXC_*_SELECT_INPUT`，
        相当于`mux_reg`下的一个**次级开关**。
        * <font color="red">mux_mode</font>：`复用模式`，即`mux_reg`的值。二进制位定义如下：
            * `bit[31:5]`：预留未用。
            * `bit[4]`：`SION`（`Software Input ON`）标志位，值为`1`表示锁定为某一种功能输入路径，
            为`0`则表示输入路径由以下`模式选项`决定。
            * `bit[3:0]`：`模式选项`，详见具体寄存器的定义。
        * <font color="green">input_val</font>：`输入值`，即`input_reg`的值，
        若无`input_reg`则忽略本值。二进制位定义如下：
            * `bit[31:m+1]`：预留未用。
            * `bit[m:0]`：`DAISY`（菊花链？）。次级开关值，详见具体寄存器的定义。
    * 通常结合`引脚控制子系统`（`pinctrl`）一起使用，在以上项的基础上再加一个用于设置`conf_reg`的值，
    即构成一个完整的pinctrl引脚定义。并且，每个具体的`conf_reg`在官方文档中的定义都包含默认值，
    多数场合可参考默认值并进行少量改动即可，但少数情形除外，
    例如`MX6UL_PAD_ENETn_TX_CLK__ENETn_REF_CLKn`（n = 1, 2）的取值应为`0x4001b009`（**原理暂未清楚**），
    采用文档默认值或示例设备树中的`0x4001b031`均不能正常工作。换而言之，**官方文档有部分错误**，
    **不可盲信，在实际开发中应同时参考其他资料，以实际情况为准！**
    * 更多原理性的内容及具体寄存器定义详见[参考材料1](#ref_1)以下章节和框图：
        * `Figure 28-1. Chip IOMUX Scheme`
        * `Figure 28-3. GPIO pad functional diagram`
        * `Figure 28-4. Schmitt trigger transfer characteristic`
        * `Figure 28-5. Receiver output in CMOS and hysteresis`
        * `Figure 28-6. Output Driver Functional Diagram`
        * `Figure 28-7. Keeper functional diagram`
        * `Figure 28-8. Output buffer in open drain mode`
        * `Figure 32-3. Daisy chain illustration`
        * `32.5 IOMUXC SNVS Memory Map/Register Definition`
        * `32.6 IOMUXC Memory Map/Register Definition`

## 9、GPIO

* 头文件：
    * `<linux/gpio.h>`：GPIO自己的接口。
    * `<linux/of_gpio.h>`：一次包含`<linux/gpio.h>`和`<linux/of.h>`，提供GPIO专属的`OF`函数。

## 10、并发与竞争

* `原子变量`：
    * **最轻量**，但仅能操作`整型`及`二进制位`。
    * 数据结构：`atomic_t`，位于`<linux/types.h>`
    * 操作接口：`atomic_*()`，位于`<linux/atomic.h>`

* `自旋锁`：
    * 原地转圈圈**空等**，占用CPU时间。适用于**短时加锁**的场景。
    要注意**加锁期间不能休眠**，因为线程在持有和获取锁的时候均是禁止内核抢占的，
    但若休眠则会被切换出去。假设一个线程获得锁并休眠，此时又有另一个线程在请求锁，
    则会把后者切换进来且在请求成功之前一直会占着CPU，这就导致前者休眠结束也不会得到调度，
    于是`死锁`现象就发生了。此外，`中断`及其`下半部`机制也能争夺调度权，
    所以**普通函数**最好在**获取锁之前先关中断**、**释放锁前后再开中断**，
    中断服务函数则只需调用简单的加解锁函数即可，`下半部`函数则调用带`_bh`后缀的锁函数。
    * 数据结构：`spinlock_t`，位于`<linux/spinlock_types.h>`
    * 操作接口：`spin_*()`，位于`<linux/spinlock.h>`（已包含上述头文件）

* `读写锁`：
    * **宽松版**的`自旋锁`：允许多个同时读，只有发生一个或以上的写操作时才需要加锁,
    即读与读可以并行，而读与写、写与写需要独占。其余与自旋锁类似。
    * 数据结构：`rwlock_t`，位于`<linux/rwlock_types.h>`
    * 操作接口：`rwlock_init()`、`{read,write}_*()`，位于`<linux/rwlock.h>`（
    但该头文件不能直接包含，需要使用自旋锁的头文件）

* `顺序锁`：
    * **宽松版**的`读写锁`：允许多读、多读一写（会有脏读），但写与写还是要独占。
    其余与读写锁类似。
    * 数据结构：`seqlock_t`，位于`<linux/seqlock.h>`
    * 操作接口：`seqlock_init()`、`{read,write}_*()`，位于`<linux/seqlock.h>`

* `信号量`：
    * **计数型**的锁，在等待锁的时候宿主线程会进入休眠状态，所以不耗费CPU，
    但会有上下文切换的开销，所以适用于可能会长时间占用资源且允许休眠的场合（
    中断服务函数不能休眠）。
    * 数据结构：`struct semaphore`，位于`<linux/semaphore.h>`
    * 操作接口：`sema_init()`、`up()`、`down()`、`down_*()`，位于`<linux/semaphore.h>`

* `互斥体`：
    * **锁如其名**，理念上类似于**二值型**的`信号量`，但在实现上不一定基于`信号量`。
    * 数据结构：`struct mutex`，位于`<linux/mutex.h>`
    * 操作接口：`mutex_*()`，位于`<linux/mutex.h>`

* 更详细的解释可参考一位网友的[读书笔记](references/《Linux内核设计与实现》读书笔记（十）-内核同步方法-闫宝平-博客园.pdf)。

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
        * 数据结构：`struct bus_type`，位于`<linux/device.h>`
        * 操作接口：`bus_*`，位于`<linux/device.h>`

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

### 21.1 串口（UART）

待补充。

### 21.2 I2C

待补充。

### 21.3 SPI

待补充。

### 21.4 控制器局域网（CAN）

待补充。

### 21.5 USB

* 头文件：`<linux/usb.h>`

* 抽象层级：
    * 内核角度：其他子系统（字符设备、网络设备等）-> USB设备驱动 -> USB核心层 -> USB主机控制器驱动
    * USB设备角度：设备（Device）-> 配置（Configuration）-> 接口（Interface）-> 端点（Endpoint）/ 管道（Pipe）

* 注册与注销：
    * 接口驱动：
        * 数据结构：`struct usb_driver`、`struct usb_device_id`、`struct usbdrv_wrap`
        * 操作接口：`usb_register_driver()`、`usb_register()`、`usb_deregister()`
    * 设备驱动：
        * 数据结构：`struct usb_device_driver`、`struct usbdrv_wrap`
        * 操作接口：`usb_register_device_driver()`、`usb_deregister_device_driver()`
    * 使用主设备号与应用程序通信的驱动：
        * 数据结构：`struct usb_interface`、`struct usb_class_driver`
        * 操作接口：`usb_register_dev()`、`usb_deregister_dev()`

* 主要数据结构的访问与互转：
    * 数据结构：`struct usb_device`、`struct usb_interface`
    * 操作接口：
        * `interface_to_usbdev()`
        * `usb_get_intfdata()`、`usb_set_intfdata()`

* 管道操作接口：`usb_{snd,rcv}{ctrl,isoc,bulk,int}pipe()`

* 缓冲区操作接口：
    * ~~旧：`usb_buffer_{alloc,free}()`~~
    * 新：`usb_{alloc,free}_coherent()`

* 异步处理逻辑——`URB`（`USB Request Block`）：
    * 数据结构：`struct urb`
    * 操作接口：
        * 分配与释放：`usb_alloc_urb()`、`usb_free_urb()`
        * 初始化：`usb_fill_{control,bulk,int}_urb()`（等时型即isoc URB需手动初始化）、`usb_init_urb()`
        * 提交任务：`usb_submit_urb()`（初始化URB或从中断函数退出时，都要提交一次）
        * 终止任务：`usb_kill_urb()`（等待完成）、`usb_unlink_urb()`（当指定`URB_ASYNC_UNLINK`标志时不等待）。
        注意要避免与回调函数里的释放URB逻辑的冲突，可能需要加锁。此外，在回调函数里也不能休眠。

* 同步处理逻辑：
    * 收发数据：`usb_{control,bulk,interrupt}_msg()`（不能用于中断上下文或持有自旋锁，不能中途取消执行）
    * 其他：`usb_get_descriptor()`、`usb_get_status()`、`usb_string()`、……

### 21.X 待补充

待补充。

## 附录：参考材料

1. <a id="ref_1" href="references/i.MX 6ULL Applications Processor Reference Manual.pdf" target="_blank">i.MX 6ULL Applications Processor Reference Manual</a>

