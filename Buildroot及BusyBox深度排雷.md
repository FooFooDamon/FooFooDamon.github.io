<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<base target="_blank" />

# Buildroot及BusyBox深度排雷

## 1、前言

作为《[嵌入式根文件系统构建实录](嵌入式根文件系统构建实录.md)》的续集，
专排`Buildroot`及`BusyBox`埋的雷。

## 2、详情

* **注1**：以下各个解决方案通常要求重新编译`Buildroot`，
根据实际情况而定，不再赘述。

* **注2**：本文所有的定制化目录及文件均放在`Buildroot`源码根目录下的`custom`文件夹内，
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

待补充。

### 2.5 解决`kobs-ng`编译错误

`kobs-ng`是一个可在目标板内Linux系统将U-Boot烧写到NAND Flash的工具。

编译错误及解决方案待补充。

