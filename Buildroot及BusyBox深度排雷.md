<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<base target="_blank" />

# Buildroot及BusyBox深度排雷

## 1、前言

作为《[嵌入式根文件系统构建实录](嵌入式根文件系统构建实录.md)》的续集，
专排`Buildroot`及`BusyBox`埋的雷。

## 2、详情

* **注1**：以下各个解决方案通常要求重新编译`Buildroot`，
根据实际情况而定，不再赘述。

* **注2**：除非特别说明，所有命令均在`Buildroot`根目录下执行。

* **注3**：本文所有的定制化目录及文件均放在`Buildroot`源码根目录下的`custom`文件夹内，
定制化原理详见“前言”里的文章“添加自定义目录及文件”章节的内容，后文不再赘述。

### 2.1 OpenSSH：Privilege separation user sshd does not exist

* `现象`：系统在执行开机启动脚本`/etc/init.d/S50sshd`（即启动SSH服务进程）时，
报`Privilege separation user sshd does not exist`的错误，启动失败。

* `原因`：`OpenSSH`出于安全的考虑，默认会以非特权用户的角色去处理来自客户端的身份认证请求，
即所谓的**特权分离**机制，原理可阅读这篇[英文博客](https://jfrog.com/blog/examining-openssh-sandboxing-and-privilege-separation-attack-surface-analysis/)
（链接若失效可点击[此处](references/openssh_privilege_separation.pdf)查看备份文档）。
此机制要求有一个非特权用户（即非root用户）供其使用，否则便会报上述错误。
而`OpenSSH`会通过`/etc/passwd`等文件验证`sshd`用户，
但`Buildroot`的**用户表**（Users Tables）机制却没向上述文件写入`sshd`用户信息，
因而导致问题的出现。

* `方案`：手工添加`sshd`用户信息，或通过脚本操作，本文使用后者。
创建`custom/etc/init.d/S49sshdpatch`，写入以下内容：

    ````
    #!/bin/sh

    start()
    {
        local gid=$(grep "^sshd:" /etc/group | awk -F : '{ print $3 }')
        local uid=$(grep "^sshd:" /etc/passwd | awk -F : '{ print $3 }')

        if [ -z "${gid}" ]; then
            gid=$(awk -F : '{ print $3 }' /etc/group | sort -n | tail -n 1)
            gid=$((${gid} + 1))
            echo "sshd:x:${gid}:" >> /etc/group
        fi

        if [ -z "${uid}" ]; then
            uid=$(awk -F : '{ print $3 }' /etc/passwd | sort -n | tail -n 1)
            uid=$((${uid} + 1))
            echo "sshd:x:${uid}:${gid}:SSH drop priv user:/:/bin/false" >> /etc/passwd
        fi

        [ $(grep -c "^sshd:" /etc/shadow) -gt 0 ] || echo "sshd:*:::::::" >> /etc/shadow
    }

    stop()
    {
        :
    }

    restart()
    {
        stop && start
    }

    case "$1" in
      start)
        start
        ;;
      stop)
        stop
        ;;
      restart|reload)
        restart
        ;;
      *)
        echo "Usage: $0 {start|stop|restart}"
        exit 1
    esac

    exit $?
    ````

### 2.2 modprobe: can't change directory to '/lib/modules': No such file or directory

* `现象`：系统在执行`mdev`的开机启动脚本时，报此错误。

* `原因`：`Buildroot`未创建系统模块所需的目录和文件。

* `方案`：需手动创建：

    ````
    $ mkdir -p custom/lib/modules/4.1.15 # 内核版本需自行确定
    $ touch custom/lib/modules/4.1.15/modules.dep # 文件可空，若有.ko文件也可复制到此目录，然后编辑该文件
    ````

### 2.3 ps: invalid option -- 'e'

* `现象`：执行`ps -ef`时，报此错误。

* `原因`：编译了阉割版的`BusyBox`。

* `方案`：执行`make busybox-menuconfig`开启`BusyBox`更多特性支持，然后再重新编译`Buildroot`：

    ````
    Settings  --->
        [*] Enable compatibility for full-blown desktop systems
    ````

### 2.4 令BusyBox接受中文输入

* 将待修改的源码文件复制一份，用于后面的补丁制作：

    ````
    $ cp output/build/busybox-*/libbb/printable_string.c output/build/busybox-*/libbb/unicode.c ~/
    ````

* 执行`vim output/build/busybox-*/libbb/printable_string.c`，
将函数`printable_string`（`1.29.3`版本）或`printable_string2`（`1.36.0`版本）以下内容：

    ````
    /* 省略很多行 */
            if (c < ' ')
                break;
            if (c >= 0x7f)
                break;
            s++;
    /* 省略若干行 */
                if (c == '\0')
                    break;
                if (c < ' ' || c >= 0x7f)
                    *d = '?';
                d++;
    /* 省略很多行 */
    ````

    修改为：

    ````
    /* 省略很多行 */
            if (c < ' ')
                break;
            /*if (c >= 0x7f)
                break;*/
            s++;
    /* 省略若干行 */
                if (c == '\0')
                    break;
                if (c < ' '/* || c >= 0x7f*/)
                    *d = '?';
                d++;
    /* 省略很多行 */
    ````

* 执行`vim output/build/busybox-*/libbb/unicode.c`，
将函数`unicode_conv_to_printable2`以下内容：

    ````
    /* 省略很多行 */
                    *d++ = (c >= ' ' && c < 0x7f) ? c : '?';
                    src++;
    /* 省略若干行 */
                    if (c < ' ' || c >= 0x7f)
                        *d = '?';
                    d++;
    /* 省略很多行 */
    ````

    修改为：

    ````
    /* 省略很多行 */
                    *d++ = (c >= ' '/* && c < 0x7f*/) ? c : '?';
                    src++;
    /* 省略若干行 */
                    if (c < ' '/* || c >= 0x7f*/)
                        *d = '?';
                    d++;
    /* 省略很多行 */
    ````

* 制作补丁（补丁文件名随便取，可以参考其他补丁文件名）：

    ````
    $ diff -urN ~/printable_string.c output/build/busybox-1.36.0/libbb/printable_string.c > package/busybox/0005-libbb-printable_string.patch
    $
    $ sed -i -e 's/\(---[ \t]\+\)[^ \t]\+\(\/printable_string\.c[ \t]*\)/\1a\/libbb\2/' \
        -e 's/\(+++[ \t]\+\)[^ \t]\+\(\/printable_string\.c[ \t]*\)/\1b\/libbb\2/' package/busybox/0005-libbb-printable_string.patch
    $
    $ diff -urN ~/unicode.c output/build/busybox-1.36.0/libbb/unicode.c > package/busybox/0006-libbb-unicode_conv_to_printable2.patch
    $
    $ sed -i -e 's/\(---[ \t]\+\)[^ \t]\+\(\/unicode\.c[ \t]*\)/\1a\/libbb\2/' \
        -e 's/\(+++[ \t]\+\)[^ \t]\+\(\/unicode\.c[ \t]*\)/\1b\/libbb\2/' package/busybox/0006-libbb-unicode_conv_to_printable2.patch
    ````

    这样，即使以后对整个工程`make clean`再重新`make`，也不需要执行前面几步的手动操作，编译脚本会自动打补丁。

### 2.5 解决`kobs-ng`编译错误

`kobs-ng`是一个可在目标板内Linux系统将U-Boot烧写到NAND Flash的工具。

若使用`2019.02.6`版本的`Buildroot`搭配`4.9`版本的`交叉编译器`，则会发生编译错误，
将此两者的版本分别更新为`2023.02`和`7.5.0`即可解决。顺便附上`kobs-ng`的配置路径：

````
Target packages  --->
    -*- BusyBox
        Hardware handling  --->
            [*] Freescale i.MX libraries  --->
                [*]   imx-kobs
````

### 2.6 解决`ip`命令设置`CAN`接口时产生的报错

* `现象`：

    执行以下命令时：
    ````
    $ ip link set can0 type can bitrate 1000000 restart-ms 1000
    $ ip link set can0 up
    ````

    产生以下报错：
    ````
    ip: either "dev" is duplicate, or "type" is garbage
    flexcan 2090000.can can0: bit-timing not yet defined
    ip: SIOCSIFFLAGS: Invalid argument
    ````

* `原因`：`BusyBox`的`ip`命令不支持`CAN`，需要使用`Buildroot`的。

* `方案`：执行`make menuconfig`，按以下路径选中`iproute2`选项，再重新`make`即可：
    ````
    Target packages  --->
        Networking applications  --->
            [*] iproute2
    ````

### 2.7 LEB size mismatch

* `现象`：

    系统启动时挂载根文件系统出现以下报错：
    ````
    UBIFS error (ubi0:0 pid 1): validate_sb: LEB size mismatch: 129024 in superblock, 126976 real
    ````

* `原因`：构建根文件系统时，指定的`逻辑块`大小与实际的大小不一致。

* `方案`：执行`make menuconfig`，找到`logical eraseblock size`参数项并修改（注：原默认值为`0x1f800`），
再重新`make`以及烧录即可：
    ````
    Filesystem images  --->
        [*] ubifs root filesystem
        (0x1f000) logical eraseblock size
    ````

### 2.8 bad VID header offset

* `现象`：

    系统启动时挂载根文件系统出现以下报错：
    ````
    UBI error: validate_ec_hdr: bad VID header offset 512, expected 2048
    ````

* `原因`：构建根文件系统时，指定的`子页面`大小与实际的大小不一致。

* `方案`：执行`make menuconfig`，找到`sub-page size`参数项并修改（注：原默认值为`512`），
再重新`make`以及烧录即可：
    ````
    Filesystem images  --->
        [*] ubi image containing an ubifs root filesystem
        (0x20000) physical eraseblock size
        (2048) sub-page size
    ````

* **注意**：
    * 上一小节所用的编译选项是`ubifs root filesystem`，生成的文件带的后缀是`.ubifs`，
    可以利用厂商提供的工具进行离线烧写，也可在`uboot`命令行用`ubi`命令烧写（
    详细的命令待补充）。
    * 本节所用的编译选项是`ubi image containing an ubifs root filesystem`，文件后缀是`.ubi`，
    可在`uboot`命令行使用`nand`命令烧写（通过`USB`或`TFTP`将镜像读入内存之后，
    就是简单的`nand erase`和`nand write`操作了，在此不详细叙述，仅需注意填对目标分区的地址或名称即可）。

