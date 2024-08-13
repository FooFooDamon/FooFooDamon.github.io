<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# `U-Boot`移植进阶版——基于`香橙派5`（`RK3588S`）

## 1、背景

* 之前入手了一块`香橙派5`开发板用来做驱动测试，近日由于在调试某些功能需要修改一些内核代码，
担心中途改出问题而进不了系统，所以打算不动储存卡上的内核镜像，而是修改`U-Boot`启动脚本，
让`U-Boot`在启动时从电脑下载修改过的内核镜像来运行，如此一来即使出问题也能轻松重启，
且不必输入任何额外的`U-Boot`命令或修改其启动脚本。

* 改好`U-Boot`启动脚本（主要是增加优先通过`TFTP`获取内核镜像的逻辑）之后，
才发现出厂`U-Boot`程序的网络功能并未做好（平时没有留意其启动日志，所以未发现这个问题），
于是问题变成先移植一个有网络功能的`U-Boot`。

* 既然要移植，那么就要知道编译和烧录的方法，但若按照开发板的用户手册来操作，
则需要下载一个很笨重的`SDK`——其中绝大部分内容都来源于`GitHub`——这对国内多数人是个重大考验，
原因不需细说。为了绕开这个笨重的`SDK`，并且对该`U-Boot`移植项目做版本管理，
需要另辟蹊径——这就是这篇文章的诞生原因。

* 在正文开始之前，还要稍微解释一下本文为何是`进阶版`——皆因以前写过一篇`基础版`文章（
感兴趣者可到[此处](姗姗来迟的U-Boot移植.md)进行阅读），本文只是多了一些`基础版`文章未涉及的概念，
并且在讲述过程中也只是拣某些重点来说，比较适合有一定基础和项目经验的读者。

## 2、原理简述

* `香橙派5`开发板的处理器是瑞芯微公司的`RK3588S`，其与前述`基础版`文章里`NXP`公司的`i.MX6ULL`的区别
不仅表现在芯片功能和性能，也表现在引导程序——即`U-Boot`：其一，`U-Boot`程序的文件格式有变；
其二，启动流程变长变复杂。

* `U-Boot`程序的**文件格式**方面：
    * `i.MX6ULL`采用的是`传统`（`Legacy`）格式。此格式最简单的情形是只有一个`64`字节的文件头，
    但各芯片厂商往往会加料、魔改（例如`i.MX6ULL`就额外增加`IVT`(`Image Vector Table`)、
    `DCD`(`Device Configuration Data`)等信息，有兴趣可阅读[这篇文章](https://www.cnblogs.com/bathwind/p/18180871)），
    使得最终文件更专有化，需要使用厂商提供的烧录工具。不过，无论文件头的具体形式如何，
    这种格式的特点都是：能表达的信息通常很有限，但启动的路径也短，依赖的文件也少
    （就是其本身）。`mkimage`命令的使用手册（可通过执行`man mkimage`）有这样一段话：
        > The old legacy image format concatenates the individual parts
        (for example, kernel image, device tree blob and ramdisk image) and adds a 64 bytes header
        containing information about target architecture, operating system, image type, compression method,
        entry points, time stamp, checksums, etc.
    * `RK3588S`采用的是`扁平化镜像树`（`Flattened Image Tree`，`FIT`）格式。此格式**类似于设备树**，
    因而能很灵活地把数量不固定的多个文件组织到一起。`mkimage`命令的使用手册有这样一段话：
        > The  new  FIT (Flattened Image Tree) format allows for more flexibility in handling images of
        various types and also enhances integrity protection of images with stronger checksums.
        It also supports verified boot.

* **启动流程**方面：
    * `i.MX6ULL`：在大多数的应用场景下，可以简单认为只有一个阶段或步骤
    （仅针对`U-Boot`而言，不包括之前的芯片内部固化代码和之后的内核启动，下同）。
    * `RK3588S`：至少两个阶段或步骤。之所以分阶段，除了有扩展性方面的考虑，
    更因为芯片内部的`SRAM`极其有限，不能完整加载较大体积的程序文件。

* 根据以上两个方面，可引申出新的问题。首先是搞清楚需**生成哪些文件**：
    * `i.MX6ULL`：
        * `u-boot.imx` = `文件头` + `u-boot-nodtb.bin`
    * `RK3588S`：
        * 用于`储存卡`、`eMMC`等介质时：
            * `idbloader.img` = `u-boot-tpl.bin` + `u-boot-spl.bin`
            * `u-boot.itb` = `u-boot-nodtb.bin` + `u-boot.dtb` + `u-boot.its` + `bl31.elf` + `...`
        * 用于`SPI NOR Flash`时：
            * `rkspi_loader.img` = `idbloader.img` + `u-boot.itb`

* 还有就是**烧录方式**是怎样的：
    * `i.MX6ULL`：`NXP`提供的工具集，或`Buildroot`集成的`kobs-ng`命令。
    * `RK3588S`：瑞芯微提供的工具集，或`香橙派5`系统自带脚本`/usr/sbin/nand-sata-install`，
    或最基本的`dd`命令。

* 通过以上的对比说明，读者应该有一个大致的了解，至少站在做项目的角度来说已经足够。
当然，若想更深入探究，可参阅以下文章：
    * 瑞芯微官方说明：
        * `U-Boot`：[原网页](https://opensource.rock-chips.com/wiki_U-Boot)及<a href="references/uboot/U-Boot - Rockchip open source Document.pdf">备份文档</a>
        * `Boot option`：[原网页](https://opensource.rock-chips.com/wiki_Boot_option)及<a href="references/uboot/Boot option - Rockchip open source Document.pdf">备份文档</a>
        * `RK3399 boot sequence`：[原网页](https://wiki.pine64.org/wiki/RK3399_boot_sequence)及<a href="references/uboot/RK3399 boot sequence - PINE64.pdf">备份文档</a>
    * 网友经验总结：
        * `Mini2440之uboot移植流程之linux内核启动分析（六）`：[原网页](https://www.cnblogs.com/zyly/p/15814964.html)及<a href="references/uboot/Mini2440之uboot移植流程之linux内核启动分析（六） - 大奥特曼打小怪兽 - 博客园.pdf">备份文档（仅保存部分相关内容）</a>
        * `Rockchip RK3399 - 引导流程和准备工作`：[原网页](https://www.cnblogs.com/zyly/p/17380243.html)及<a href="references/uboot/Rockchip RK3399 - 引导流程和准备工作 - 大奥特曼打小怪兽 - 博客园.pdf">备份文档</a>
        * `Rockchip RK3399 - TPL/SPL方式加载uboot`：[原网页](https://www.cnblogs.com/zyly/p/17389525.html)及<a href="references/uboot/Rockchip RK3399 - TPL_SPL方式加载uboot - 大奥特曼打小怪兽 - 博客园.pdf">备份文档</a>
        * `瑞芯微 RK 系列芯片启动流程简析`：[原网页](https://www.w568w.eu.org/rockchip-boot-process.html)及<a href="references/uboot/瑞芯微 RK 系列芯片启动流程简析.pdf">备份文档</a>

## 3、实现细节

首先要强调的是，现实中的情形并非是先理论后实践、且一气呵成的过程，因为即使有前面的参考材料，
也要费一番心思去阅读、梳理，更别提在对很多新概念和知识点一空二白的情况下，还要先费心找资料，
并在一堆沙子里找黄金，这就很考验一个人的探索和筛选能力了。下面将尽量还原此次经历的来龙去脉，
供各位参考（参考就好，不要照搬）。

第一个且极其重要的步骤，就是**看用户手册**，即使有些用户手册很烂。**不看文档的人**，**踩坑完全不冤**！
得益于用户手册，才能获知这些信息：源码出处、构建工具是什么、产出的形式是`DEB`包、
开发板已自带烧录脚本等等。接下来要做的只是逐个击破即可。

回顾文章开头提到的绕开笨重的官方`SDK`而手动编译`U-Boot`，首先要做的当然是**下载源码包**。
这一步很简单，唯一需要注意的是，最好按`散列码`（`commit hash`）或`标签`（`tag`）下载，
而不是按`分支`（`branch`），例如：

````
$ wget -c -O u-boot-orangepi-752ac3f2fdcfe9427ca8868d95025aacd48fc00b.tar.gz \
    'https://github.com/orangepi-xunlong/u-boot-orangepi/archive/752ac3f2fdcfe9427ca8868d95025aacd48fc00b.tar.gz'
````

而不是：

````
$ wget -c -O u-boot-orangepi-v2017.09-rk3588.tar.gz \
    'https://github.com/orangepi-xunlong/u-boot-orangepi/archive/refs/heads/v2017.09-rk3588.tar.gz'
````

拿到源码之后，先不急着编译，而是先**搞清楚需要编译出什么产物**。由于用户手册已说明产物是`DEB`包，
且提供了解包命令，因而可获知结果文件的安装路径，然后直接在开发板上查看：

````
$ ls /usr/lib/u-boot
LICENSE  orangepi_5_defconfig  platform_install.sh
$
$ ls /usr/lib/linux-u-boot-*
idbloader.img  rkspi_loader.img  u-boot.itb
````

其中，`/usr/lib/u-boot`目录下的`orangepi_5_defconfig`可直接拿来覆盖源码包里的同名配置文件，
并用于`make menuconfig`；`platform_install.sh`是烧录脚本的其中一个组件，此时可先不理会。
至于`/usr/lib/linux-u-boot-*`（末尾的星号对应的是官方`SDK`定义的分支和发行版本号，
在此不关心）目录下的文件，则与前一章节列举的待烧录清单对应上了，接下来只需搞清楚如何把它们编译出来就行了。

可以肯定的是，直接在`U-Boot`源码根目录执行`make`大概率是得不到这些文件的，
因为仅从文件名就能看出它们多少有些特殊。要解决这个问题，还是要**研读官方`SDK`的脚本逻辑**，
不能仅靠前面的参考材料——原因也很简单，一来不同芯片有差异，二来代码和文章也有时效性，
所以即使很麻烦，官方`SDK`也是最接近正确答案的方案。不过，前面的参考材料并不算白读，
因为基础理论也同等重要，不可或缺。

现在稍微分析一下官方`SDK`的脚本逻辑。香橙派官方`SDK`叫`orangepi-build`，
在[GitHub](https://github.com/orangepi-xunlong/orangepi-build)有开源。
为了避免脚本内容更新而导致与本文解说对应不上的情况，特意复刻了一份以冻结住相关状态，
读者若感兴趣，查阅[复刻的这个仓库](https://github.com/FooFooDamon/orangepi-build)即可。
由于篇幅的限制以及不是本文重点，详细的分析就不在这里展开，仅列举部分核心逻辑。
首先，找出编译`U-Boot`的顶层函数，在`scripts/compilation.sh`文件，
剔除不重要以及与`RK3588(S)`无关的内容，其内部核心逻辑大致如下：

````
compile_uboot()
    |
    |-- ……
    |
    |-- while read -r target do ... done <<< $UBOOT_TARGET_MAP
    |       |
    |       |-- UBOOT_TARGET_MAP：定义在external/config/sources/families/include/rockchip64_common.inc，
    |       |       且取值由若干变量影响，这些变量定义在external/config/boards/orangepi5.conf，
    |       |       此处仅列举重要变量取值如下：
    |       |           BOARDFAMILY：rockchip-rk3588
    |       |           BOOTCONFIG：orangepi_5_defconfig
    |       |           BOOT_FDT_FILE：rockchip/rk3588s-orangepi-5.dtb
    |       |           BOOT_SCENARIO：spl-blobs
    |       |       因此，UBOOT_TARGET_MAP的值为：BL31=$RKBIN_DIR/$BL31_BLOB spl/u-boot-spl.bin u-boot.dtb u-boot.itb;;idbloader.img u-boot.itb
    |       |       很容易在同一文件内找到BL31_BLOB的取值应为：rk35/rk3588_bl31_v1.42.elf
    |       |       这对于后面下载瑞芯微闭源固件很重要。
    |       |
    |       |-- target_make=$(cut -d';' -f1 <<< "${target}")，即UBOOT_TARGET_MAP的值以分号分隔的第一部分内容：
    |       |       BL31=$RKBIN_DIR/$BL31_BLOB spl/u-boot-spl.bin u-boot.dtb u-boot.itb
    |       |
    |       |-- target_files=$(cut -d';' -f3 <<< "${target}")，逻辑与上相似，是第三部分：
    |       |       idbloader.img u-boot.itb
    |       |
    |       |-- make $BOOTCONFIG：根据orangepi_5_defconfig生成“.config”
    |       |
    |       |-- 微调“.config”里面部分编译选项
    |       |
    |       |-- make $target_make：即编译出spl/u-boot-spl.bin、u-boot.dtb、u-boot.itb
    |       |
    |       |-- uboot_custom_postprocess()：定义在external/config/sources/families/include/rockchip64_common.inc，
    |       |       作用是针对部分型号的板进行特定的后处理，例如若想从香橙派5的SPI NOR Flash启动，
    |       |       还需生成rkspi_loader.img
    |       |
    |       `-- 复制$target_files到待打包目录
    |
    |-- 生成platform_install.sh：主要作用是提供write_uboot_platform()、write_uboot_platform_mtd()
    |       等函数供烧录脚本/usr/sbin/nand-sata-install调用，
    |       它们均是定义在external/config/sources/families/include/rockchip64_common.inc
    |
    `-- 打DEB包：略
````

从上面可以看出，除了有一个`rk3588_bl31_v*.elf`固件是闭源、需要手动获取之外，其余内容都是开源的，
而且也不复杂。还应注意到有若干个关键函数定义在`external/config/sources/families/include/rockchip64_common.inc`，
可以简单看看其实现：

````
uboot_custom_postprocess()
{
    if [[ $BOOT_SUPPORT_SPI == yes ]]; then
        if [[ $BOARDFAMILY == "rockchip-rk3588" ]]; then
            tools/mkimage -n rk3588 -T rksd -d $RKBIN_DIR/$DDR_BLOB:spl/u-boot-spl.bin idbloader.img
            dd if=/dev/zero of=rkspi_loader.img bs=1M count=0 seek=4
            /sbin/parted -s rkspi_loader.img mklabel gpt
            /sbin/parted -s rkspi_loader.img unit s mkpart idbloader 64 1023
            /sbin/parted -s rkspi_loader.img unit s mkpart uboot 1024 7167
            dd if=idbloader.img of=rkspi_loader.img seek=64 conv=notrunc
            dd if=u-boot.itb of=rkspi_loader.img seek=1024 conv=notrunc
        elif [[ $BOARDFAMILY == "rockchip-rk356x" ]]; then
            ...
        fi
    fi
}

#
# 用法：write_uboot_platform <镜像所在目录> <储存卡对应的设备节点>
# 示例：write_uboot_platform /usr/lib/linux-u-boot-current-orangepi5_1.1.8_arm64 /dev/mmcblk1
#
write_uboot_platform()
{
    if [[ -f $1/rksd_loader.img ]]; then # legacy rk3399 loader
        ...
    elif [[ -f $1/u-boot.itb ]]; then # $BOOT_USE_MAINLINE_ATF == yes || $BOOT_USE_TPL_SPL_BLOB == yes
        dd if=$1/idbloader.img of=$2 seek=64 conv=notrunc status=none >/dev/null 2>&1
        dd if=$1/u-boot.itb of=$2 seek=16384 conv=notrunc status=none >/dev/null 2>&1
    elif [[ -f $1/uboot.img ]]; then # $BOOT_USE_BLOBS == yes
        ...
    else
        echo "[write_uboot_platform]: Unsupported u-boot processing configuration!"
        exit 1
    fi
}

#
# 用法同write_uboot_platform()
#
write_uboot_platform_mtd()
{
    if [[ -f $1/rkspi_loader.img ]]; then
        dd if=$1/rkspi_loader.img of=$2 conv=notrunc status=none >/dev/null 2>&1
    else
        echo "SPI u-boot image not found!"
        exit 1
    fi
}
````

当然，还会有一些边边角角的小逻辑，读者感兴趣的话可自行查看其余脚本，但最重要最核心的逻辑已在上面列出，
看明白之后就可以**开始编译**了。编译环境方面，使用电脑或`香橙派5`开发板均可，
但最好是`Ubuntu 22.04`，参考命令如下（先解压前述`U-Boot`源码压缩包并进入其根目录）：

````
$ sudo apt install gcc-11-aarch64-linux-gnu # 若使用电脑，则需要先安装交叉编译器
$
$ wget -c -O bl31.elf 'https://github.com/armbian/rkbin/raw/master/rk35/rk3588_bl31_v1.42.elf'
$ sed -i '1s/\(python\)2/\1/' arch/arm/mach-rockchip/decode_bl31.py
$ make all u-boot.dtb u-boot.itb ARCH=arm CROSS_COMPILE=aarch64-linux-gnu-
$
$ make tpl/u-boot-tpl.bin spl/u-boot-spl.bin ARCH=arm CROSS_COMPILE=aarch64-linux-gnu-
$ tools/mkimage -n rk3588s -T rksd -d tpl/u-boot-tpl.bin idbloader.img
$ cat spl/u-boot-spl.bin >> idbloader.img
````

当然，若要作为一个项目，肯定要为其增加版本管理，具体可通过本文开头的`基础版`文章链接去进一步了解，
在这里就不继续重复述说，至于代码仓库，详见[U-Boot移植项目](https://github.com/FooFooDamon/uboot_porting)的`orange-pi-5`子目录。

成功编译后，便可以**烧录**了，只需**在开发板**执行以下命令：

````
$ source /usr/lib/u-boot/platform_install.sh # 为了导入一个镜像目录变量DIR
$ sudo cp u-boot.itb idbloader.img ${DIR}/ # 要先拷贝镜像到特定目录才能烧录；若之前在电脑编译，此处就用scp命令
$ sudo nand-sata-install && sudo reboot # 烧录并重启
````

至此，本文到了收尾阶段。至于开头提到的移植网络功能的需求，与本文的`U-Boot`核心逻辑关系不大，
而且还未实现，所以就不作为本文内容了，感兴趣的可继续关注前述的`U-Boot移植项目`，
但由于日常繁忙，完工的日期无法预料。

