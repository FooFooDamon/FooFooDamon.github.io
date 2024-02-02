<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# 姗姗来迟的U-Boot移植

## 1、迟到原因

* **嵌入式**`ARM`-`Linux`系统的构成可归结成`4`大件：`引导程序`、`设备树`、`内核`、`根文件系统`。

* `引导程序`即常说的`Boot Loader`，有多种选择，目前在嵌入式`ARM`领域用得最多的是`U-Boot`，
为方便叙述，后文将其简写成`uboot`。

* 若从**启动流程**的角度来说，`uboot`排第一位毫无疑问；但若从**学习和做项目**的角度，则未必。

* `uboot`的**核心任务**是**初始化必要的外设（时钟、内存等）**、**加载设备树和内核**以及**启动内核**，
然后它的使命就完成而退出舞台了——这隐含了`2`个意思：
    * `uboot`承担的功能没必要太复杂，在满足基本的项目需求的前提下，要做的移植工作越少越好。
    * `uboot`能加载其他部件，也意味着能**烧录**它们。引申出更具体的意思就是：
    其他部件可以用`uboot`来烧录，反复烧也不会太麻烦（仅需要一条网线连接、不需要切换模式跳线、
    不需要来回切换`Windows`/`Linux`）；但若将未改好的`uboot`烧录进去而导致内核引导失败进不去系统，
    则下次再烧`uboot`就需要借助外部的`储存卡`或`USB`线，模式跳线和主机操作系统也需要切换。

* 对于普通客户和项目来说，经过了`半导体厂商`和`开发板厂商`的适配，大量的工作已经完成，
而项目所用的电路板往往又是参考了前两者的设计，所以`uboot`并不需要多少（复杂的）改动，
就能得到一个包含基本、常用功能的成品。对于那些非核心非紧急的功能（例如某些调试、
关联变量自动更新及保存等），可以在后期慢慢打磨。

## 2、基础版移植

### 2.1 获取源码包

* `uboot`源码的来源有`3`个：`U-Boot`官方、`半导体厂商`（例如`NXP`等）、`开发板厂商`（例如某立功、某点原子等）。

* 除了`半导体厂商`之外，基本没有人会在`U-Boot`官方的源码包基础来做移植，除非嫌工作量不饱和。
不过，感兴趣者可自行访问其[官网](https://docs.u-boot.org/)和[GitHub](https://github.com/u-boot/u-boot)了解更多情况。

* 可以使用`开发板厂商`提供的源码包，或者去`半导体厂商`的网站或`GitHub`搜索。以`NXP`公司的`i.MX6ULL`为例，
其`i.MX`系列的`GitHub`账号是[nxp-imx](https://github.com/nxp-imx/)。搜索过程略去不提，
在此直接给出下载命令示例（读者也可以在前述`GitHub`账号主页查找`uboot-imx`项目，然后在其`Tags`页找到对应版本再手动下载）：
    ````
    $ wget -c 'https://github.com/nxp-imx/uboot-imx/archive/refs/tags/rel_imx_4.1.15_2.1.0_ga.tar.gz' -O uboot-imx-rel_imx_4.1.15_2.1.0_ga.tar.gz
    # 顺便给出配套的Linux内核源码包的下载命令
    # wget -c 'https://github.com/nxp-imx/linux-imx/archive/refs/tags/rel_imx_4.1.15_2.1.0_ga.tar.gz' -O linux-imx-rel_imx_4.1.15_2.1.0_ga.tar.gz
    ````

* 需要指出的，并非使用`开发板厂商`的源码就一定比`半导体厂商`的源码省心。譬如，
开发板厂商若选择某点原子，虽然它的教程浅显易懂，适合初学者入门，但代码风格欠佳，
很多方法也不够通用化，属于头痛医头脚痛医脚的做法；若选择某立功，虽然其硬件做得不错，
但软件却未达到与之匹配的高度，而且个别项目还包含加密文件，藏着掖着的让人很不舒服，
尽管有办法破解（破解之后还会发觉并无大幅改进性能的逻辑或高深的算法，
只有少量适配自己硬件的校验操作，若用于其他硬件可能会校验失败而卡死），总归是费事。
所以，对于有一定经验、又想对源码有更大掌控的工程师来说，还不如直接使用半导体厂商的代码，
一来因为它们往往在`GitHub`上有长期维护、持续迭代的项目，对二次开发的项目的引用和部署都很便利；
二来因为就算开发板厂商的硬件有改动，绝大部分的内容还是基于半导体厂商的，改动并不大，
反而可能引入一些不必要的复杂度，使代码没有原来的易懂和“纯洁”。不过，建议归建议，
各位根据自己的实际情况来选择就行了，没有明显的好坏优劣之分，只有适合与否的问题。

### 2.2 定制编译配置

* 若是从`开发板厂商`购买`整板`或`核心板`并**在此基础上二次定制**，则使用具体开发板或核心板的编译配置。

* 若是购买`主控芯片`并**自行设计电路板**，通常会参考`半导体厂商`的`EVK`（`Evaluation Kit`，`评估套件`）版本。
本文以这种情况为例，所用的配置是`configs/mx6ull_14x14_evk_nand_defconfig`（`NAND`版的`EVK`）。

* 在源码根目录下先执行`make mx6ull_14x14_evk_nand_defconfig ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-`，
再执行`make menuconfig ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-`，并至少修改以下配置项：
    ````
    Command line interface  --->
        (i.MX6ULL > ) Shell prompt
            Environment commands  --->
                [*] editenv
                [*] saveenv
                [*] env exists
            Device access commands  --->
                [*] nand
                [*] usb
            Network commands  --->
                [*] bootp, tftpboot
                [*] nfs
                [*] ping
    Networking support  --->
        [*]   Control TFTP timeout and count through environment
    Device Drivers  --->
        [*] Ethernet PHY (physical media interface) support
        [*] USB support  --->
            [*]   EHCI HCD (USB 2.0) support
            [*]   Support for i.MX6 on-chip EHCI USB controller
            [*]   USB Mass Storage support
    Library routines  --->
        [*] Enable function for getting errno-related string message
    ````
    突出`2`个重点：
    * 营造一个**最小功能集**：例如对于`NAND`版本，`nand`系列命令肯定要有。
    * 必须具备**必要的补救措施**：方便在出现意外（例如写过界破坏文件系统分区）时，
    能够很容易地进行恢复，例如`USB`和网络相关的功能。

### 2.3 修改板级头文件

* 即`include/configs/mx6ullevk.h`。首先修改**内存大小**：
    ````
    31 #ifdef CONFIG_TARGET_MX6ULL_9X9_EVK
    32 #define PHYS_SDRAM_SIZE              SZ_256M
    33 #define CONFIG_BOOTARGS_CMA_SIZE     "cma=96M "
    34 #else
    35 #define PHYS_SDRAM_SIZE              SZ_512M     /* 此处改总内存大小 */
    36 #define CONFIG_BOOTARGS_CMA_SIZE     ""          /* 此处改连续内存区大小，若无需要也可置空 */
    37 /* DCDC used on 14x14 EVK, no PMIC */
    38 #undef CONFIG_LDO_BYPASS_CHECK
    39 #endif
    ````
    注：这里稍微提一下`CMA`（`Contiguous Memory Allocator`，`连续内存分配器`），
    这是一个致力于提高内存的运用效率和调配灵活性的机制，即：预先划出一片地址连续的内存，
    优先提供给需要大量连续内存进行`DMA`（`Direct Memory Access`，`直接内存访问`）操作的设备驱动使用，
    但当这些设备不工作时，这部分内存又可以作为通用内存供系统里的其他部件使用，
    等到设备再需要时又通过回收或迁移的方式划归回来，而不像早期的`Linux`内核那样把这部分内存锁死
    而得不到充分利用。并且，这一系列策略对驱动开发者是透明、无感知的。在`配置方式`方面则有`3`种，
    按优先级从高到低列举则为：`设备树`（`linux,cma { ... }`）、`参数传递`（`cma=XXX`，
    `uboot`在启动内核时传递，既可使用上面的编译期硬编码方式，也可由用户在运行期进入`uboot`动态修改）、
    `内核编译选项`（`CONFIG_CMA_SIZE_*`）。

* 开启**（调试）串口**并指定其`基址`：
    ````
    59 #define CONFIG_MXC_UART
    60 #define CONFIG_MXC_UART_BASE        UART1_BASE /* 若调试串口不是串口1，则需修改 */
    ````

* 修改**NAND闪存分区**（根据实际需求）：
    ````
    92 #ifdef CONFIG_SYS_BOOT_NAND
    93 #define CONFIG_MFG_NAND_PARTITION "mtdparts=gpmi-nand:4m(boot),1m(dtb),8m(kernel),8m(kernel-bk),4m(logo),73m(rootfs),-(custom) "
    94 #else
    95 #define CONFIG_MFG_NAND_PARTITION ""
    96 #endif
    ````

* 修改**启动参数**（根据实际需求）：
    ````
    113 #if defined(CONFIG_SYS_BOOT_NAND)
    114 #define CONFIG_EXTRA_ENV_SETTINGS \
    115     CONFIG_MFG_ENV_SETTINGS \
    116     "fdt_addr=0x83000000\0" \
    117     "fdt_high=0xffffffff\0"   \
    118     "console=ttymxc0\0" \
    119     "bootargs=console=ttymxc0,115200 ubi.mtd=5 "  \
    120         "root=ubi0:rootfs rootfstype=ubifs "    \
    121         CONFIG_BOOTARGS_CMA_SIZE \
    122         "\0"\
    123     "bootcmd=nand read ${loadaddr} 0x500000 0x800000;"\
    124         "nand read ${fdt_addr} 0x400000 0x100000;"\
    125         "bootz ${loadaddr} - ${fdt_addr}\0"
    ````
    注意`bootargs`变量的`ubi.mtd`参数值、`bootcmd`的两个`NAND`起始地址值均与前面的闪存分区有关。

* **最重要**别忘记修改**环境变量的分区偏移量和大小**：
    ````
    317	#elif defined(CONFIG_ENV_IS_IN_NAND)
    318	#undef CONFIG_ENV_SIZE
    319	#define CONFIG_ENV_SIZE         SZ_1M /* 新增一个宏用于表示环境变量储存区大小 */
    320	#define CONFIG_ENV_OFFSET       (SZ_4M - CONFIG_ENV_SIZE) /* 该储存区目前位于uboot分区内部，所以偏移量也要在分区范围内 */
    321	#define CONFIG_ENV_SECT_SIZE    (128 << 10)
    322	#endif
    ````
    注意：由于原代码将`uboot`分区划分到`64M`那么大，`CONFIG_ENV_OFFSET`的值也达到`60M`的大小，
    而新代码的`uboot`分区只有`4M`，若`CONFIG_ENV_OFFSET`不同步改小，则在执行`saveenv`命令保存环境变量时，
    毫无疑问会写到别的分区去，造成破坏，详见后文叙述。

## 3、进阶版移植

前面的`基础版移植`只是实现了**最简单的**引导逻辑。在实际的项目中，通常还要做更多的移植，
使调试和日常使用更加便利。

### 3.1 修改以太网络（方便以网络形式调试Linux系统和根文件系统）

* 由于`i.MX6ULL`已内置`MAC`部件，所以只需再外接一个`PHY`芯片，
本文以`TI`公司的`DP83848`芯片为例进行说明。

* 首先在`include/configs/mx6ullevk.h`修改`PHY`芯片地址：
    ````
    345 #define CONFIG_FEC_ENET_DEV             0 /* 0表示使用ENET1接口 */
    346
    347 #if (CONFIG_FEC_ENET_DEV == 0)
    348 #define IMX_FEC_BASE                    ENET_BASE_ADDR /* ENET1寄存器基址 */
    349 #define CONFIG_FEC_MXC_PHYADDR          0x1  /* 根据芯片的地址引脚的实际连线来确定，若悬空则使用默认地址0x1 */
    350 #define CONFIG_FEC_XCV_TYPE             RMII /* 使用RMII接口 */
    351 #elif (CONFIG_FEC_ENET_DEV == 1)        /* ENET2在本项目没用到，所以该条件分支里的内容不用改 */
    352 #define IMX_FEC_BASE                    ENET2_BASE_ADDR
    353 #define CONFIG_FEC_MXC_PHYADDR          0x1
    354 #define CONFIG_FEC_XCV_TYPE             RMII
    355 #endif
    356 #define CONFIG_ETHPRIME                 "FEC" /* 网口名称，可改可不改 */
    357
    358 #define CONFIG_PHYLIB
    359 #define CONFIG_PHY_TI                   /* 从*_MICREL改成*_TI */
    360 #endif
    ````

* 还要修改板级源文件`board/freescale/mx6ullevk/mx6ullevk.c`。分两点：
    * 修改`时钟`引脚的**电气属性**（原因在后面章节有说）：
        ````
        72 #define ENET_CLK_PAD_CTRL  (PAD_CTL_HYS | PAD_CTL_PUS_100K_UP | \
        73     PAD_CTL_PUE | PAD_CTL_PKE | PAD_CTL_SPEED_LOW | PAD_CTL_DSE_240ohm | \
        74     PAD_CTL_SRE_FAST)
        ````
    * 修改`复位`引脚，也分为两点：
        * 将`board_init()`初始化`74LV595`芯片（用于扩展引脚数量）的两行代码删掉：
            ````
            imx_iomux_v3_setup_multiple_pads(iox_pads, ARRAY_SIZE(iox_pads));
            iox74lv_init();
            ````
            同时删除的还有与之相关的`iox74lv_*()`函数定义、`IOX_*`宏定义、`qn*`枚举值及数组等。
        * 在`fec1_pads`数组添加（直连的）`复位`引脚配置项：
            ````
            MX6_PAD_LCD_RESET__GPIO3_IO04 | MUX_PAD_CTRL(NO_PAD_CTRL), /* 注意引脚名称根据实际情况而定 */
            ````

* 对于`DP83848`来说，这些修改已经足够，不像某点原子用的`LAN8720A`，
还要在另外两个地方分别进行硬复位和软复位。

### 3.2 支持可读性更好的储存分区管理（即`mtdparts`）

* 首先解释一下`mtdparts`的含义及其必要性：
    * `mtdparts`可拆分为`MTD`和`parts`：
        * `MTD`即`Memory Technology Device`（`内存技术设备`）——不必纠结这么拗口的命名，
        只需知道这种技术理念能屏蔽`闪存`类设备的数据格式差异、
        简化驱动设计（尤其是与擦写寿命相关的逻辑，以及坏块管理）即可。
        * `parts`可理解为`分区`（`partition`）管理，并且使用了可读性更好的`描述性文字`，
        而非原先的一堆数字。
    * 由以上得知，开启了这项功能之后，就能**免除用户在使用闪存类命令时记忆各种地址数字的痛苦**，
    这就是它的必要性以及最大意义所在。

* 其次需要解释它的修改思路。由于在`make menuconfig`找不到与之相关的配置项，
所以只能**通过阅读源码来解决**，大致如下：
    * 所有命令的实现均在`cmd`目录，而该功能又有一个相关的命令叫`mtdparts`，所以首先查看`cmd/Makefile`，
    找到引用`mtdparts.o`的行，确定其相关的宏定义，然后在板级头文件定义该宏。
    * 前一步只是添加了`mtdparts`命令，但不一定能正常工作，需要先烧写一次并运行该命令，
    观察其报错，再根据报错内容去查找源码，一步一步补齐缺失的配置项（大多是宏定义）。
    虽然涉及多个地方，但主要逻辑集中在`cmd/mtdparts.c`和`cmd/nand.c`。

* 最后是实现详情。前面的思路，说起来很简单，但要阅读源码还是挺繁杂，
不过最后的修改结果又很简单，仅需在`include/configs/mx6ullevk.h`增加和修改若干内容：
    ````
     92	#define CONFIG_CMD_MTDPARTS
     93	#define CONFIG_MTD_DEVICE
     94	#define CONFIG_MTD_PARTITIONS
     95	#define MTDIDS_DEFAULT              "nand0=gpmi-nand"
     96	#define MTDPARTS_DEFAULT            "mtdparts=gpmi-nand:4m(boot),1m(dtb),8m(kernel),8m(kernel-bk),4m(logo),73m(rootfs),-(custom)"
     97	#define UBOOT_PARTITION_SIZE        SZ_4M
     98	#define DTB_PARTITION_SIZE          SZ_1M
     99	#define KERNEL_PARTITION_SIZE       SZ_8M
    100
    101	#ifdef CONFIG_SYS_BOOT_NAND
    102	#define CONFIG_MFG_NAND_PARTITION   MTDPARTS_DEFAULT " "
    103	#else
    104	#define CONFIG_MFG_NAND_PARTITION   ""
    105	#endif
    ...
    122	#if defined(CONFIG_SYS_BOOT_NAND)
    123	#define CONFIG_EXTRA_ENV_SETTINGS \
    124		CONFIG_MFG_ENV_SETTINGS \
    125		"mtdparts=" MTDPARTS_DEFAULT "\0" \
    126		"kernel_size=" __stringify(KERNEL_PARTITION_SIZE) "\0" \ /* 手动定义一个内核文件大小的变量，以便后续版本更新时由相关命令自动更新，并用于bootcmd */
    127		"dtb_size=" __stringify(DTB_PARTITION_SIZE) "\0" \       /* 与上相似 */
    128		"fdt_addr=0x83000000\0" \
    129		"fdt_high=0xffffffff\0"	  \
    130		"console=ttymxc0\0" \
    131		"ipaddr=192.168.111.111\0" \
    132		"netmask=255.255.255.0\0" \
    133		"serverip=192.168.111.3\0" \
    134		"bootargs=console=ttymxc0,115200 ubi.mtd=5 ro quiet "  \
    135			"root=ubi0:rootfs rootfstype=ubifs "		     \
    136			CONFIG_BOOTARGS_CMA_SIZE \
    137			"\0"\
    138		"bootcmd=nand read ${loadaddr} kernel ${kernel_size};"\ /* 注意原镜像分区地址已替换成kernel */
    139			"nand read ${fdt_addr} dtb ${dtb_size};"\       /* 注意原设备树分区地址已替换成dtb */
    140			"bootz ${loadaddr} - ${fdt_addr}\0"
    141	
    142	#else
    ...
    ````

### 3.3 其他特定项目所需的修改

例如：储存卡、显示屏、蓝牙、无线网络等，都是可选配置，视实际项目的需要而增减，
若有配备则需要移植，否则不用理会，在此不能一一列举讲述，有需要者可自行搜索其移植方法。

## 4、独树一帜的项目管理

### 4.1 一般做法带来的问题

* **代码仓库臃肿**：由于是将整个项目的源码**全量**提交到版本管理软件，
因此一个仓库的内容极多，如果是像`Linux`内核这种体量的项目，更是臃肿到无以复加，
可以想象不论是`GitHub`上数以千万计的项目`复刻`（`fork`），还是下载下来提交到私有仓库，
都是一遍又一遍地浪费磁盘空间。

* **不能第一时间发现修改点**：其实是上一点带来的副作用。对于一个项目来说，
`uboot`、`Linux`内核里的`99.99%`内容都是业务无关的，就像空气和水，虽然不可或缺，
却不如米饭肉菜那样对人产生那样大的`实际效用`，而适配项目的改动就如同米饭肉菜，
但却淹没在大量的内容之中，不能一下子发现，这对于项目的理解和维护是不利的。

* 生成的二进制文件**没带有详细版本号**：一个`uboot`源码包是有大的版本号的，
例如`2016.03`，但由于是拿来移植到自己的项目，必然要经过多次的改动，
以及在生产环境上部署，万一出现问题（应该说毫无意外都会出现问题，区别只是问题的大小），
仅靠大版本号是无法追溯的，所以最好也把版本管理软件每次提交生成的散列码（即`git hash`）
也编译到二进制文件中。不过，即使有意识这么做，散列码也往往需要手工获取以及赋值，
极其不便。

### 4.2 改进做法

* **调整项目内容储存形式**：即将原来的源码包作为一个压缩包保存在仓库中，
甚至可以不必保存，而是在`Makefile`保存一个下载链接，在首次编译时直接下载，
保存到本地但设置成被版本管理软件忽略，同时将需要改动的少数几个文件提取出来，
按原有的目录层次保存并提交到版本库，如此一来整个项目就会非常轻便。项目示例如下：
    ````
    新uboot根目录
        |-- arch
        |       `-- arm
        |               `-- cpu
        |                       `-- armv7
        |                               `-- start.S
        |-- board
        |       `-- freescale
        |               `-- mx6ullevk
        |                       `-- mx6ullevk.c
        |-- configs
        |       `-- mx6ull_14x14_evk_nand_defconfig
        |-- include
        |       `-- configs
        |               `-- mx6ullevk.h
        |-- Makefile
        `-- uboot-imx-rel_imx_4.1.15_2.1.0_ga.tar.gz
    ````

* 在`Makefile`里定义**改动文件的覆盖操作**以及**小版本号的生成逻辑**：
即上面的任何文件改动之后，都要复制到压缩包解压之后的目录里进行编译，
以及自动获取当前的`git`散列码并告诉最终生成二进制文件的`Makefile`。
说白了，上面的`Makefile`只是基于`uboot`原有的编译脚本的封装，
不会太复杂但也不会太简单，但却能减少无意义的重复劳动。

* `Makefile`的详细逻辑不会在本文展开，但后续会有专门的文章进行解释，
至于改进过的项目链接则放在文末。

## 5、踩雷与排雷实录

### 5.1 不当地启用或禁用`Driver Model`选项引起的编译错误

`Driver Model`即`驱动模型`，目的在于为驱动的定义和访问接口提供统一的方法，
以提高驱动间的兼容性以及访问的标准性，其与`Linux`内核的`设备驱动模型`类似，
但也有所区别，由于不是本文重点就不展开叙述了。以下是部分踩雷与排雷示例
（`menuconfig`一级配置项均是`Device Drivers`）：

若开启了在启动期间显示`CPU`信息的功能，则同时需要开启以下选项：
````
[*] Enable CPU drivers using Driver Model
````
否则会报以下错误：
````
cmd/cpu.c:37：对‘cpu_get_info’未定义的引用
````

对于以太网口，则不能开启以下选项：
````
[ ] Enable Driver Model for Ethernet drivers
````
否则会报以下错误：
````
drivers/net/fec_mxc.c:538:47: 错误： dereferencing pointer to incomplete type ‘struct eth_device’
````
不过，这个问题只在较旧版本的`uboot`才出现。在`v2020.07`之后的版本，
`struct eth_device`及相关接口已移除，全面拥抱`Driver Model`机制。

对于`USB`，则不能开启以下选项：
````
[*] USB support  --->
    [ ]   Enable driver model for USB
````
否则会报与设备树相关接口的错误：
````
drivers/usb/host/usb-uclass.c:696: undefined reference to `fdtdec_get_int'
````

若有其他类似报错，可先定位报错信息所在的文件和行数，再查找其相关的`CONFIG_*`宏，
并复制此宏名称到`menuconfig`界面去查找，最后再开启或禁用相关的配置项，一般就能解决报错了。

### 5.2 `saveenv`配置未改全而导致写过界事故

在前面`基础版移植`一章中提到，若只修改`uboot`分区大小而忘记同步修改环境变量储存区偏移量的话，
会导致`saveenv`写过界，例如：

````
i.MX6ULL > saveenv 
Saving Environment to NAND...
Erasing NAND...
Erasing at 0x3c00000 -- 100% complete.
Writing to NAND... OK
````

注意上面的`0x3c00000`对应`60M`的位置，在本项目中则落在`根文件系统`分区内，
就是说会破坏根文件系统，后续重启会卡住。除了要修正偏移量重新编译`uboot`，
根文件系统也要单独修复。由于前面已经移植好`USB`，所以可以将根文件系统镜像放到U盘里，
然后直接在`uboot`命令行进行烧写，过程样例如下（注意镜像后缀是`.ubi`而非`.ubifs`，
镜像生成方法可参考《[Buildroot及BusyBox深度排雷](Buildroot及BusyBox深度排雷.md)》
一文“`2.8 bad VID header offset`”小节）：

````
i.MX6ULL > usb start
starting USB...
USB0:   USB EHCI 1.00
scanning bus 0 for devices... 2 USB Device(s) found
USB1:   USB EHCI 1.00
scanning bus 1 for devices... 1 USB Device(s) found
       scanning usb for storage devices... 1 Storage Device(s) found
       scanning usb for ethernet devices... 0 Ethernet Device(s) found
i.MX6ULL > usb dev

USB device 0: Vendor: Mass     Rev: 1.00 Prod: Storage Device
            Type: Removable Hard Disk
            Capacity: 32000.0 MB = 31.2 GB (65536000 x 512)
i.MX6ULL > usb dev 0

USB device 0:
    Device 0: Vendor: Mass     Rev: 1.00 Prod: Storage Device
            Type: Removable Hard Disk
            Capacity: 32000.0 MB = 31.2 GB (65536000 x 512)
... is now current device
i.MX6ULL > fatls usb 0
 29966336   rootfs.ubifs
 30801920   rootfs.ubi

2 file(s), 0 dir(s)

i.MX6ULL > fatload usb 0 ${loadaddr} rootfs.ubi
reading rootfs.ubi
31195136 bytes read in 3081 ms (9.7 MiB/s)
i.MX6ULL > printenv filesize
filesize=1dc0000
i.MX6ULL > nand device

Device 0: nand0, sector size 128 KiB
  Page size       2048 b
  OOB size         128 b
  Erase size    131072 b
  subpagesize     2048 b
  options     0x40000200
  bbt options 0x    8000
i.MX6ULL > nand device 0
i.MX6ULL > nand erase 0x1900000 0x4900000

NAND erase: device 0 offset 0x1900000, size 0x4900000
Erasing at 0x61e0000 -- 100% complete.
OK
i.MX6ULL > nand write ${loadaddr} 0x1900000 ${filesize}

NAND write: device 0 offset 0x1900000, size 0x1dc0000
 31195136 bytes written: OK
i.MX6ULL > reset
resetting ...
````

### 5.3 用`kobs-ng`命令烧写`uboot`的注意事项

除了可用厂商提供的工具软件并借助储存卡来进行整个系统的烧录，还可以在操作系统里单独烧`uboot`，
不过需要在制作根文件系统时安装好`kobs-ng`命令，并且要遵循一定的流程，如下：

````
$ mount -t debugfs debugfs /sys/kernel/debug # 获取NAND的BCH布局
$
# 注：uboot在mtd的第几个分区与前面的mtdparts划分情况有关，一般分在最开头，即第0个分区。
$ flash_erase /dev/mtd0 0 0
$
$ kobs-ng init -x -v --chip_0_device_path=/dev/mtd0 /path/to/u-boot.imx # 烧写
$
$ sync # 确保数据落盘
````

最后要注意的是，重启时**不能直接以`reboot`命令重启**，**只能按硬件复位键进行硬复位**！
否则，会卡住，并且无法重启，只能使用储存卡重烧整个系统。

### 5.4 开启`tftpput`命令时误将`tftp`掩盖

当勾选以下配置项时，`tftp`命令将不能使用（会影响下载调试），提示命令缺失：

````
Command line interface  --->
    Network commands  --->
        [*] tftp put
````

所以必须取消`tftpput`命令的勾选，反正一般情况下用不上，最好确保相邻的`tftpsrv`也取消掉。

### 5.5 以太网口能ping通但tftp下载失败

在`uboot`命令行下通过`tftp`命令下载文件时曾出现以下现象：

````
Loading: #error frame: 0x9ee55180 0x00000804
T error frame: 0x9ee551c0 0x00000804
T error frame: 0x9ee551c0 0x00000804
T error frame: 0x9ee551c0 0x00000804
T error frame: 0x9ee55200 0x00000804
T T error frame: 0x9ee55200 0x00000804
T #error frame: 0x9ee55240 0x00000804
T error frame: 0x9ee55240 0x00000804
T error frame: 0x9ee55240 0x00000804
T #error frame: 0x9ee55280 0x00000804
````

需要注意下载进度显示中，用`T`表示等待超时或网络不通，用`#`表示正常接收数据，
而以上内容两种符号都出现过，况且又能ping通，说明网络是通的，只是数据接收极不稳定。
本来这种情况比较难排查，但由于之前做`Linux`内核移植时，曾遇过网络通但`SSH`操作很卡的情况，
最终在搜索各种方案以及对比多个项目的设备树配置时，发现了时钟引脚的电气配置有点跷蹊，
最终也证实了官方（即`NXP`）的配置有点问题，而某点原子的配置竟然可用，从而解决了问题。
所以，这次我也往这个方向排查，没想到又是同一个问题！

时钟引脚的电气配置详见`3.1`小节。这里需要提一下的是，
最终配置与官方配置的差异仅表现在引脚`输出能力`上，即`电阻`大小：
可用的值是`PAD_CTL_DSE_240ohm`，官方的则是`PAD_CTL_DSE_40ohm`。
由于这完全是硬件层面的问题，所以我也只能作一个不成熟的猜想，就是：官方配置的电阻值过小，
从而导致驱动电流过大，信号边沿升降时间很短，`电磁干扰`（`Electromagnetic Interference`，`EMI`）也强，
所以对数据包的收发造成干扰。若说错则请对这方面有了解的朋友指正。

### 5.6 内存不对齐引发的数据中断异常

报错现象类似下面这样：

````
Load address: 0x83000000
Loading: data abort
pc : [<9ff889e8>]          lr : [<9ff88a90>]
reloc pc : [<878429e8>]    lr : [<87842a90>]
sp : 9ee45b70  ip : 000000eb     fp : 00000045
r10: 00000b0f  r9 : 9ee45e80     r8 : 9ffef1f0
r7 : 00000f0b  r6 : 00004500     r5 : 0000002f  r4 : 9ffed38e
r3 : 14000045  r2 : 6f6fa8c0     r1 : 9ee45b78  r0 : 9ffed38e
Flags: nzCv  IRQs off  FIQs off  Mode SVC_32
Resetting CPU ...

resetting ...
````

原因是启动初始化代码对CPU设置了`内存对齐`的检查，并且在运行过程中（网络收包时）检测到使用非对齐的内存地址的操作，
于是引发了异常而重置。本来应该修改相关的网络驱动代码，但若时间比较紧迫，或者要修改的范围太大，
则可以临时禁用这项检查，等以后有时间再慢慢改，或者等待修复这类缺陷（若是`uboot`自身缺陷的话）的新版本`uboot`发布。
禁用的方法是，在`arch/arm/cpu/armv7/start.S`将对应的设置语句改成清零，目标语句及相邻代码如下：

````
113	ENTRY(cpu_init_cp15)
114		/*
115		 * Invalidate L1 I/D
116		 */
117		mov	r0, #0			@ set up for MCR
118		mcr	p15, 0, r0, c8, c7, 0	@ invalidate TLBs
119		mcr	p15, 0, r0, c7, c5, 0	@ invalidate icache
120		mcr	p15, 0, r0, c7, c5, 6	@ invalidate BP array
121		mcr     p15, 0, r0, c7, c10, 4	@ DSB
122		mcr     p15, 0, r0, c7, c5, 4	@ ISB
123	
124		/*
125		 * disable MMU stuff and caches
126		 */
127		mrc	p15, 0, r0, c1, c0, 0
128		bic	r0, r0, #0x00002000	@ clear bits 13 (--V-)
129		bic	r0, r0, #0x00000007	@ clear bits 2:0 (-CAM)
130	#if 0
131		orr	r0, r0, #0x00000002	@ set bit 1 (--A-) Align    /* 原先是开启对齐检查 */
132	#else
133		bic	r0, r0, #0x00000002	@ clear bit 1 (--A-) Align  /* 现在要禁用 */
134	#endif
````

## 6、总结

* 由浅入深地介绍了`uboot`移植的大致流程和若干重点。

* 个中穿插了作者的一些个人经验之谈以及项目管理理念。

* 要明确区分`修改点`和`不变点`，并尽量将移植逻辑通用化，致力于解决一类问题而非单个问题，
这样在第二次之后的移植，工作量将会大大降低。

* 通用化的结果就是可将非业务敏感性的内容进行开源，可反复重用、经受锤炼，
利己利他。本文所涉及的源码及脚本详见[U-Boot移植](https://github.com/FooFooDamon/uboot_porting)项目的
`imx6ull`子目录，且后续的更新不会再同步到本文。

