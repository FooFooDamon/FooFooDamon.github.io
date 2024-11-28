<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# 目的性明确的U-Boot启动流程简析

## 1、背景

* 需要解决`香橙派5`开发板的有线网络在`U-Boot`下用不了的问题，
具体原因详见[这篇文章](U-Boot移植进阶版——基于香橙派5（RK3588S）.md)。

* 此前试过仅凭部分日志去查找和分析若干个源码文件并不能解决问题，
感觉需要从全局视角去排查才能有更多线索。加上接触`U-Boot`挺长时间却未有一些总结，
索性专门研究一番，并写下一些笔记梳理一下脉络以及用于备忘。

* 由于初始动机是排查网络驱动未生效的原因而不是研究汇编语言和芯片原理，
而且本人的领域也不是芯片原厂的驱动开发，所以本文与网上很多文章大不相同，
不会过多地深究和解析汇编代码的含义，而是侧重于整个流程的大体走向，
如此便不至于偏离初衷主次不分，更不会陷入只见树木不见森林的困境。

* 温馨提示：本文不适合零基础新手，建议读者具备一定的编译链接常识（含链接脚本的知识）、
C语言基础、`Linux`内核驱动基础知识（包括设备树、设备模型等）。

## 2、链接脚本

即代码根目录下的`u-boot.lds`文件，基于`arch/arm/cpu/u-boot.lds`生成（须完整编译过一次之后），
摘要如下：

````
...
ENTRY(_start) /* 入口，详见后面的u-boot.map内容 */
SECTIONS
{
    . = 0x00000000;     /* 当前段的起始地址，后同 */
    . = ALIGN(8);       /* 当前段的对齐方式，后同 */
    .text :             /* 文本段（代码段）*/
    {
        *(.__image_copy_start)
        arch/arm/cpu/armv8/start.o (.text*)
        *(.text*)
    }
    ...
}
````

## 3、内存布局

代码根目录下的`u-boot.map`是`U-Boot`的映射文件，其中就包含内存布局信息。
本文只关注与程序入口相关的信息，摘要如下：

````
...
 1920 内存配置
 1921
 1922 名称           来源             长度             属性
 1923 *default*        0x0000000000000000 0xffffffffffffffff
 1924
 1925 链结器命令稿和内存映射
 1926
 1927 段 .text 的地址设置为 0x200000
 1928                 0x0000000000000000                . = 0x0
 1929                 0x0000000000000000                . = ALIGN (0x8)
 1930
 1931 .text           0x0000000000200000    0xc7054
 1932  *(.__image_copy_start)
 1933  .__image_copy_start
 1934                 0x0000000000200000        0x0 arch/arm/lib/built-in.o
 1935                 0x0000000000200000                __image_copy_start
 1936  arch/arm/cpu/armv8/start.o(.text*)
 1937  .text          0x0000000000200000      0x138 arch/arm/cpu/armv8/start.o
 1938                 0x0000000000200000                _start
 1939                 0x0000000000200008                _TEXT_BASE
 1940                 0x0000000000200010                _end_ofs
 1941                 0x0000000000200018                _bss_start_ofs
 1942                 0x0000000000200020                _bss_end_ofs
 1943                 0x000000000020002c                save_boot_params_ret
 1944                 0x0000000000200094                apply_core_errata
 1945                 0x00000000002000b8                lowlevel_init
 1946                 0x00000000002000d4                smp_kick_all_cpus
 1947                 0x00000000002000e0                c_runtime_cpu_setup
 1948                 0x0000000000200118                save_boot_params
 1949  *(.text*)
 1950  *fill*         0x0000000000200138      0x6c8
 1951  .text          0x0000000000200800      0x6d4 arch/arm/cpu/armv8/built-in.o
 1952                 0x0000000000200800                vectors
...
````

## 4、汇编代码简析

从入口`_start`开始分析，其位于`arch/arm/cpu/armv8/start.S`，核心逻辑如下：

````
_start:
#ifdef CONFIG_ENABLE_ARM_SOC_BOOT0_HOOK
#include <asm/arch/boot0.h>
#else
    b reset
#endif
````

由于编译选项`CONFIG_ENABLE_ARM_SOC_BOOT0_HOOK`=`y`，所以执行`#include <asm/arch/boot0.h>`，
实际上是`arch/arm/include/asm/arch-rockchip/boot0.h`，核心逻辑如下：

````
#if CONFIG_IS_ENABLED(TINY_FRAMEWORK)   /* 前置的 tpl/u-boot-tpl.bin 走此分支 */
    b save_boot_params
    b board_init_f
#else                                   /* 主线uboot走此分支 */
    b reset
#endif
````

再次回到`arch/arm/cpu/armv8/start.S`找`reset`函数，其实现只有简单的一句`b save_boot_params`，
而`save_boot_params`又会调用`save_boot_params_ret`，后者的核心逻辑如下：

````
save_boot_params_ret:
    bl apply_core_errata /* 芯片勘误操作，此处不关心 */
    bl lowlevel_init
    bl _main
````

`lowlevel_init`在同一文件内有一个`弱符号`定义（注意：部分汇编宏定义位于`arch/arm/include/asm/macro.h`），
核心逻辑如下：

````
WEAK(lowlevel_init)
    bl gic_init_secure /* 或gic_init_secure_percpu，根据CONFIG_IRQ、CONFIG_GICV3等选项而定 */
                       /* 均定义在arch/arm/lib/gic_64.S */
                       /* 其作用摘自源码注释：Initialize secure copy of GIC at EL3. */
ENDPROC(lowlevel_init)
````

又在`arch/arm/cpu/armv8/lowlevel_init.S`有强定义，但只有激活`CONFIG_ARCH_SUNXI`才用到，
所以对于**非**`全志`系列的芯片没用。


至于`_main`函数，则在`arch/arm/lib/crt0_64.S`，需要重点关注，其核心逻辑如下：

````
ENTRY(_main)
    ... /* 设置指令缓存、栈指针、数据对齐、错误标志位等 */

    /* 以下两个初始化函数均位于common/init/board_init.c，仅执行少数非常简单的地址计算和赋值操作 */
    bl board_init_f_init_reserve
    bl board_init_f_boot_flags

    bl board_init_f /* 定义在common/board_f.c */

    ...

    b relocate_code /* 位于arch/arm/lib/relocate_64.S，作用是将U-Boot从闪存复制到内存 */
    bl c_runtime_cpu_setup /* 位于arch/arm/cpu/armv8/start.S，作用是根据安全模式来将异常向量表设置到相应的vBAR寄存器 */
                           /* 异常向量表则位于arch/arm/cpu/armv8/exceptions.S */

    /* 清BSS段 */

    b board_init_r /* 正式进入业务循环，该函数定义在common/board_r.c */
ENDPROC(_main)
````

稍微提一下，`指令缓存`（即`I-Cache`）可以开启，但`数据缓存`（`D-Cache`）通常不能开启，
否则会导致在启动初期就从`D-Cache`而非从内存取数据，但内存中的数据又未填充到`D-Cache`，
从而发生`预取异常`（`Prefetch Exception`）。

## 5、C函数调用链

`common/board_f.c`：

````
board_init_f()
    `-- initcall_run_list(init_sequence_f)
            `-- for (const init_fnc_t *init_fnc_ptr : init_sequence_f) { (*init_fnc_ptr)(); }

static const init_fnc_t init_sequence_f[] = {
    ...
    arch_cpu_init,  // arch/arm/mach-rockchip/rk3588/rk3588.c
    mach_cpu_init,  // RK3588无自定义内容
    ...
    env_init,       // env/env.c
    init_baud_rate,
    serial_init,    // drivers/serial/serial-uclass.c
    ...
    dram_init,      // arch/arm/mach-rockchip/sdram.c
    ...
    reserve_*,
    ...
    NULL
};
````

`common/board_r.c`：

````
board_init_r()
    `-- initcall_run_list(init_sequence_r)
            `-- for (const init_fnc_t *init_fnc_ptr : init_sequence_r) { (*init_fnc_ptr)(); }

static const init_fnc_t init_sequence_r[] = {
    ...
    initr_env_nowhere,
    ...
    board_init, // 先记住此函数，后文会用到
    ...
    initr_net,
    ...
    run_main_loop,
};

initr_net()
    `-- eth_initialize()                    // net/eth-uclass.c
            |-- eth_common_init()           // net/eth_common.c
            |       |-- miiphy_init()       // common/miiphyutil.c
            |       `-- phy_init()          // drivers/net/phy/phy.c
            |-- uclass_first_device()       // drivers/core/uclass.c
            |-- eth_get_dev_by_name()
            |       |-- uclass_get()        // drivers/core/uclass.c
            |       `-- device_probe()      // drivers/core/device.c
            |-- eth_set_dev()
            |       |-- device_probe()      // drivers/core/device.c
            |       `-- eth_get_uclass_priv()->current = dev;
            `-- eth_write_hwaddr()

run_main_loop()
    `-- main_loop()                                         // common/main.c
            |-- cli_init()                                  // common/cli.c
            |-- run_preboot_environment_command()
            |-- bootdelay_process()                         // common/autoboot.c
            |-- cli_process_fdt() & cli_secure_boot_cmd()   // common/cli.c
            |-- autoboot_command()                          // common/autoboot.c
            `-- cli_loop()                                  // common/cli.c
````

以上函数的具体实现可自行查阅相应源码，若需要更详细更基础的细节解说，可阅读[这篇文章](https://mp.weixin.qq.com/s/861UBZxLgVL_OsYNkRK82A)
（若链接失效可查看[备份文档](references/完全理解ARM启动流程：Uboot-Kernel.pdf)），
尽管处理器和开发板不一样，但原理和思路大同小异。抛开无关细节不谈，本文仅重点关注与有线网卡相关的逻辑，
详见下一节分析。

## 6、适配有线网卡

先考虑`PHY`。由于`IEEE`标准化了前`16`个`基础寄存器`的功能， 所以若只想简单使用一下基础功能，
直接采用通用驱动代码即可，换句话说就是先不用修改或增加任何`PHY`驱动代码，
等到真的测出问题时再对症下药。

接着该考虑`MAC`。可以很轻易地找出`瑞芯微`系列芯片的`MAC`驱动主要逻辑集中在`drivers/net/gmac_rockchip.c`。
打开该文件确认一下设备兼容列表里（即`struct udevice_id rockchip_gmac_ids[]`）包含了`RK3588`，
同时在同目录下`Makefile`里找出与该源文件相关的`CONFIG_GMAC_ROCKCHIP`编译选项，
并确保其处于启用状态（执行`make menuconfig`并搜索，或直接对`.config`文件`grep`亦可），
即可继续后面步骤。

由于`香橙派5`用到设备树，所以还要确保有线网络（具体对应到`RK3588`/`RK3588S`则是`GMAC`）的设备树节点配置正确。
在检查`arch/arm/dts/rk3588s-orangepi-5.dts`并参考了`内核设备树文件`之后，
从后者复制一段内容补充到前者：

````
&gmac1 {
    /* Use rgmii-rxid mode to disable rx delay inside Soc */
    phy-mode = "rgmii-rxid";
    clock_in_out = "output";

    snps,reset-gpio = <&gpio3 RK_PB2 GPIO_ACTIVE_LOW>;
    snps,reset-active-low;
    /* Reset time is 20ms, 100ms for rtl8211f */
    snps,reset-delays-us = <0 20000 100000>;

    pinctrl-names = "default";
    pinctrl-0 = <&gmac1_miim
                 &gmac1_tx_bus2
                 &gmac1_rx_bus2
                 &gmac1_rgmii_clk
                 &gmac1_rgmii_bus>;

    tx_delay = <0x42>;
    /* rx_delay = <0x3f>; */

    phy-handle = <&rgmii_phy1>;
    status = "okay";
};

&mdio1 {
    rgmii_phy1: phy@1 {
        compatible = "ethernet-phy-ieee802.3-c22";
        reg = <0x1>;
    };
};
````

改好之后就可以编译、烧录、重启测试了。以一贯的~~倒霉~~经历可知，事情是不会这么轻易成功的，
必须要出现一些波折。从打印日志来看，除了`Net: No ethernet found`之外，
就再无更多与有线网络相关的内容了，感觉是未触发相关的初始化和注册逻辑。
通过添加一些打印，证实了`GMAC`的`probe`函数确实未被触发，
于是进入其所在源文件查看其驱动总线匹配逻辑，摘要如下：

````
U_BOOT_DRIVER(eth_gmac_rockchip) = {
    ...
    .id	= UCLASS_ETH,
    ...
    .probe	= gmac_rockchip_probe,
    ...
};
````

追溯一下`UCLASS_ETH`的引用情况，其中有一处是在一个名为`dm_rm_u_boot_dev`的函数里，
此函数的作用是删除和解绑`ETH`类（即以太网）设备，并与上一节的`board_init`函数有如下调用关系：

````
int board_init(void) // arch/arm/mach-rockchip/board.c
{
    ...
#ifdef CONFIG_USING_KERNEL_DTB
    init_kernel_dtb(); // arch/arm/mach-rockchip/board.c
#endif
    ...
}
        |
        |
        V
int init_kernel_dtb(void) // arch/arm/mach-rockchip/kernel_dtb.c
{
    ...
#ifndef CONFIG_USING_KERNEL_DTB_V2
    phandles_fixup_cru((void *)gd->fdt_blob);
    phandles_fixup_gpio((void *)gd->fdt_blob, (void *)ufdt_blob);
#endif
    ...
#ifdef CONFIG_USING_KERNEL_DTB_V2
    dm_rm_kernel_dev();
    dm_rm_u_boot_dev();
#endif
    ...
}
        |
        |
        V
static int dm_rm_u_boot_dev(void)
{
    struct udevice *dev, *rec[10];
    u32 uclass[] = { UCLASS_ETH };
    int del = 0;
    int i, j, k;

    for (i = 0, j = 0; i < ARRAY_SIZE(uclass); i++) {
        for (uclass_find_first_device(uclass[i], &dev); dev;
            uclass_find_next_device(&dev)) {
            if (dev->flags & DM_FLAG_KNRL_DTB)
                del = 1;
            else
                rec[j++] = dev;
        }

        /* remove u-boot dev if there is someone from kernel */
        if (del) {
            for (k = 0; k < j; k++) {
                device_remove(rec[k], DM_REMOVE_NORMAL);
                device_unbind(rec[k]);
            }
        }
    }

    return 0;
}
````

以上内容的关键点是`CONFIG_USING_KERNEL_DTB`编译选项，
由其名称及`menuconfig`里的解释可知其作用是一个开关，用于决定是否使用内核的设备树，
而且还根据其版本的不同（即是否同时指定`CONFIG_USING_KERNEL_DTB_V2`）
来决定是否对设备树内容进行一些修补（即上述代码里的`phandles_fixup_*()`）。
再通过进一步阅读代码发现，这些逻辑是为`RK3588`及将来更新的芯片而准备的，
目前疑似仍处于半成品的状态，而且就`U-Boot`而言在大部分场景中并不需要很复杂的功能，
亦即不需要很复杂很完备的设备树配置，所以最简单的方法就是把这个编译选项禁掉，
其在`menuconfig`里的位置如下：

````
ARM architecture  --->
    [ ] Using dtb from Kernel/resource for U-Boot
````

取消该编译选项之后继续测试了，这次总算触发了相应的初始化和注册逻辑，但仍未执行成功。
继续查看报错内容和追溯代码，发现是`GMAC`的`probe`函数返回异常，出问题的地方如下：

````
static int gmac_rockchip_probe(struct udevice *dev) // drivers/net/gmac_rockchip.c
{
    ...
    struct clk clk;
    ulong rate;
    int ret;
    ...
    ret = clk_get_by_index(dev, 0, &clk); // 获取设备树节点clocks属性数组的首个属性值
    if (ret)
        return ret;
    ...
    switch (eth_pdata->phy_interface) {
    case PHY_INTERFACE_MODE_RGMII:
    case PHY_INTERFACE_MODE_RGMII_RXID:
        if (!pdata->clock_input) {
            rate = clk_set_rate(&clk, 125000000); // 在此处将前述的时钟设置成125MHz时失败
            if (rate != 125000000)
                return -EINVAL;
        }
        ...
    }
    ...
}
````

定位到出问题的设备树节点内容并修改，其位于`arch/arm/dts/rk3588s.dtsi`，修改前后对比如下：

````
        gmac1: ethernet@fe1c0000 {
                ...
-               clocks = <&cru CLK_GMAC1>, <&cru ACLK_GMAC1>,
-                        <&cru PCLK_GMAC1>, <&cru CLK_GMAC1_PTP_REF>;
-               clock-names = "stmmaceth", "aclk_mac",
-                             "pclk_mac", "ptp_ref";
+               clocks = <&cru CLK_GMAC_125M>, <&cru CLK_GMAC_50M>,
+                        <&cru PCLK_GMAC1>, <&cru ACLK_GMAC1>,
+                        <&cru CLK_GMAC1_PTP_REF>;
+               clock-names = "stmmaceth", "clk_mac_ref",
+                             "pclk_mac", "aclk_mac",
+                             "ptp_ref";
                ...
        };
````

继续走之前的测试流程，终于成功检测到有线网卡，并能顺利使用`TFTP`功能！打完收工！

## 7、项目链接

* 代码详见[U-Boot移植项目](https://github.com/FooFooDamon/uboot_porting)的`orange-pi-5/2017.09`子目录。

* 该项目并不是直接`复刻`（`fork`）某一个`U-Boot`项目进行修改，而是有自己一套组织形式，
原理详见[这篇文章](移植类项目的版本管理.md)。

