<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# 嵌入式Linux前期准备（软件篇）

## 1、可复用应用层开发的技能项

* 安装`Ubuntu`（或其他Linux发行版，本文使用Ubuntu；形式上可以是物理机双系统，
或虚拟机操作系统）：略。

* 简单的Linux`命令`，以及`Shell`脚本：无推荐资料，工多手熟，熟能生巧，多练即可。

* 常用的`VIM`操作：略。

* `GCC`及`Makefile`：推荐《跟我一起写Makefile》（作者：陈皓）。

* `Linux C 编程入门`：推荐`《UNIX环境高级编程》`。

* 代码编辑器：
    * Source Insight：适用于Windows。
    * Visual Studio Code：全平台可用，包括Linux。

## 2、串口调试工具

* Windows版USB转串口驱动：略。

* `SecureCRT`、`PuTTY`、`MobaXTerm`等：适用于Windows，略。

* `minicom`、`PuTTY`：适用于Linux，仅介绍`minicom`如下：

    * 查看串口硬件连接是否被识别：
        ````
        $ lsmod | grep usbserial
        usbserial              45056  3 ch341
        $
        $ dmesg | grep ttyUSB
        [  890.175796] usb 1-11: ch341-uart converter now attached to ttyUSB0
        ````

    * 安装并配置`minicom`：
        ````
        $ sudo apt install minicom
        $
        $ sudo minicom -s # 并对以下标有序号的菜单项按顺序进行操作
                +-----[configuration]------+
                | Filenames and paths      |
                | File transfer protocols  |
                | Serial port setup        | <--- (1)
                | Modem and dialing        |
                | Screen and keyboard      |
                | Save setup as dfl        | <--- (2)
                | Save setup as..          |
                | Exit                     |
                | Exit from Minicom        | <--- (3)
                +--------------------------+
        以下是步骤(1)的明细项:
        +-----------------------------------------------------------------------+
        | A -    Serial Device      : /dev/ttyUSB0                              |
        | B - Lockfile Location     : /var/lock                                 |
        | C -   Callin Program      :                                           |
        | D -  Callout Program      :                                           |
        | E -    Bps/Par/Bits       : 115200 8N1                                |
        | F - Hardware Flow Control : No                                        |
        | G - Software Flow Control : No                                        |
        |                                                                       |
        |    Change which setting?                                              |
        +-----------------------------------------------------------------------+
        ````
        注：以上只是最基础的设备，此外还可进入“Screen and keyboard”里，
        按提示设置“自动换行”，这有利于显示一些字数很多的打印内容
        （例如U-Boot环境变量的打印）。

    * 使用：
        ````
        $ sudo minicom -c on
        Welcome to minicom 2.7.1

        OPTIONS: I18n
        Compiled on Aug 13 2017, 15:25:34.
        Port /dev/ttyUSB0, 12:27:58

        Press CTRL-A Z for help on special keys
        ````
        若有多个串口，且波特率等参数也与默认配置不同，则可通过命令行参数来指定，例如：
        ````
        $ sudo minicom -c on -D /dev/ttyUSB1 -b 38400
        ````

## 3、交叉编译工具链

* 开发板商家会提供压缩包。

* 解压：
    ````
    $ mkdir -p ~/bin
    $ tar -jxvf gcc-linaro-arm-linux-gnueabihf-4.9-2014.09_linux.tar.bz2 -C ~/bin/
    ````

* 最后在`~/.bashrc`加入以下语句：
    ````
    export PATH=${PATH}:~/bin/gcc-linaro-arm-linux-gnueabihf-4.9-2014.09_linux/bin

    export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:~/bin/gcc-linaro-arm-linux-gnueabihf-4.9-2014.09_linux/arm-linux-gnueabihf/libc/lib/arm-linux-gnueabihf:~/bin/gcc-linaro-arm-linux-gnueabihf-4.9-2014.09_linux/arm-linux-gnueabihf/libc/usr/lib/arm-linux-gnueabihf
    ````

## 4、网络服务

### 4.1 SSH服务

* 开发板作为服务器，在开机启动脚本退出之前添加以下语句：
    ````
    /usr/sbin/sshd &
    ````

* 个人电脑作为客户端，需安装SSH客户工具：
    ````
    $ sudo apt install -y openssh-client
    ````

### 4.2 NFS服务及目录挂载

* 参考《[Linux下通过NFS挂载远程目录](Linux下通过NFS挂载远程目录.md)》。

### 4.3 TFTP服务

* 安装：
    ````
    $ sudo apt install tftpd-hpa
    ````

* 创建自己的服务器根目录并赋权限（此处目录仅作举例）：
    ````
    $ mkdir -p /your/tftp/root/dir
    $ chmod -R 777 /your/tftp/root/dir
    ````

* 在`/etc/default/tftpd-hpa`修改服务器根目录：
    ````
    TFTP_DIRECTORY="/your/tftp/root/dir"
    ````

* 重启服务：
    ````
    $ sudo service tftpd-hpa restart
    ````

## 5、禁用看门狗

* 硬件接线：根据板子的说明书或实际原理图，用跳帽或杜邦线短接某处排针，
或断开。

* 软件设置：进入U-Boot命令行，执行`wdt stop`或`setenv watchdog off`
（记得`saveenv`）。此法不是对所有板子都有效，而且要求一定版本以上的U-Boot，
最重要的一点是拼手速（因为此时看门狗是激活的，能操作的时间极短），
可以考虑使用`Expect`脚本等手段来实现。

## 6、U-Boot配置<a id="uboot_settings"></a>

以下环境变量值仅作举例，实际使用时需自行修改：

````
=> setenv bootdelay 3
=> setenv gatewayip 192.168.1.1
=> setenv serverip 192.168.1.2
=> setenv ipaddr 192.168.1.3
=> setenv netmask 255.255.255.0
=> setenv dt_addr 0x83000000
=> setenv img_addr 0x80800000
=> setenv debugboot 'tftp ${dt_addr} debug.dtb; tftp ${img_addr} zImage; bootz ${img_addr} - ${dt_addr};'
=> saveenv
````

后续若需要启动用于调试的系统时，只需执行`run debugboot`即可。

**特别注意**：首次`run debugboot`，或者每次修改并保存环境变量之后，
U-Boot可能会先进行**复位**（终端有可能输出`resetting ...`之类的提示），
然后进行默认启动（即执行`bootcmd`环境变量里的操作），下一次`run debugboot`才恢复正常。

如果想更方便，还可以添加以下设置（记得保存）：

````
=> setenv localboot 'bootcmd原先的内容'
=> setenv bootcmd 'run debugboot'
````

则以后都会以调试的形式进行启动。

