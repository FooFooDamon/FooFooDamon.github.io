<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# glibc编译演示

## 1、为何要自己编译glibc

原因不一而足，但都是由实际需要引发的，总结起来无非就两大类：

1. 学习、钻研源码。

2. 做一些刺激的事——至于有哪些刺激的事，一般人我不告诉他。

## 2、找准对象

1. 确定你的程序依赖于哪个`libc`：

    ````
    $ ldd some_program
    	linux-gate.so.1 (0xf7f83000)
    	libpthread.so.0 => /lib/i386-linux-gnu/libpthread.so.0 (0xf7f5c000)
    	libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xf7c00000)
    	/lib/ld-linux.so.2 (0xf7f85000)
    ````

2. 确定这个`libc`的版本号：

    ````
    $ strings /lib/i386-linux-gnu/libc.so.6 | grep GLIBC | tail -n 1
    GNU C Library (Ubuntu GLIBC 2.35-0ubuntu3.1) stable release version 2.35.
    ````

## 3、下载源码包

* 既可从官网下载（但龟速）：http://ftp.gnu.org/gnu/glibc/

* 亦可从国内某些大学的镜像下载：http://mirrors.nju.edu.cn/gnu/libc/

## 4、编译（注：本文使用`Ubuntu 22.04`，x86_64系统。）

### 4.1 解压及创建必要的目录

以下命令仅作为示例：

````
$ tar -zxvf glibc-2.35.tar.gz
$ cd glibc-2.35 && mkdir .build && cd .build
$ mkdir ~/myglibc
````

### 4.2 运行配置脚本

* 示例1（适用于32位X86平台）：

    ````
    $ CC="gcc -m32" CFLAGS="-g -O2" ../configure --host=i686-linux-gnu --prefix=$HOME/myglibc
    ... # 省略一大堆内容
    configure: error:
    *** These critical programs are missing or too old: gawk bison
    *** Check the INSTALL file for required versions.
    ````

    注意：

        1. gcc的-m32选项是必需的，且CC最好放在configure前面作为环境变量，
            而不是放在后面作为传递给configure的参数。CFLAGS同理。

        2. 可通过--host选项指定编译格式。有些文章使用--target=i386-linux，
            经实测，--target没起作用，并且i386-*也已经过时。

* 示例2（适用于32位ARM嵌入式平台）：

    ````
    $ CC=arm-linux-gnueabihf-gcc CFLAGS="-g -O2" ../configure --host=arm-linux-gnueabihf --prefix=$HOME/myglibc
    ... # 输出信息略
    ````

    注意：

        1. “ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- ../configure”的形式不能被配置脚本识别，
            编译出来的仍是宿主机的平台格式，即64位X86。

### 4.3 安装缺失的依赖组件（无缺失则跳过）

* 通常缺失以下两个：

    ````
    $ sudo apt install gawk bison
    ````

### 4.4 编译及安装

````
$ make -j $(grep -c processor /proc/cpuinfo)
````

### 4.5 验证

* 示例1（适用于32位X86平台）：

    ````
    $ file libc.so*
    libc.so:   ELF 32-bit LSB shared object, Intel 80386, ...
    libc.so.6: symbolic link to libc.so
    ````

* 示例2（适用于32位ARM嵌入式平台）：

    ````
    $ file libc.so*
    libc.so:   ELF 32-bit LSB shared object, ARM, ...
    libc.so.6: symbolic link to libc.so
    ````

### 4.6 安装

````
$ make install
````

### 4.7 使用

````
$ export LD_LIBRARY_PATH=$HOME/myglibc:$LD_LIBRARY_PATH
````

此时再按前述说明，使用`ldd`命令查看目标程序所依赖的动态库，
即可看到已变成这个手动编译的库。

### 4.8 卸载

````
$ rm -r ~/myglibc/*
````

