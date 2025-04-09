<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# 个人`Linux`工作环境

本文旨在记录笔者个人`Linux`工作环境的定制过程，方便在每次重装系统时能快速配置。
撰写之时所基于的操作系统是`Ubuntu 24.04`，因此若无特别说明，本文列举的命令均针对`Ubuntu 24.04`。
后续若使用更新版本的`Ubuntu`或改用其他发行版，会按需增加内容。

## 1、更换`软件源`

* 备份原来的`/etc/apt/sources.list`，然后将[清华源](https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/)的内容填入该文件。

* 刷新系统缓存：`sudo apt update -y`

## 2、安装`五笔`输入法

安装[小企鹅输入法](https://fcitx-im.org/wiki/Fcitx_5/zh-cn)：

````
$ sudo apt install fcitx-table-wubi
````

注意：
* `Ubuntu`默认的输入法是`ibus`，若要使用`fcitx`，需在“区域与语言”（不同版本的名称可能不一样）手动设置，
并重启或注销重登录方能生效。并且，“区域与语言”仅保留一个输入源才能让`ibus`消失，
不然`ibus`与`fcitx`共存会引起输入法冲突。
* 当`fcitx`定义的快捷键与系统键盘快捷键相同时，可能会不生效，
此时清空相应的系统键盘快捷键（打字 -> 切换至上/下个输入源）即可。

## 3、安装`网络工具`

````
$ sudo apt install net-tools # 包含ifconfig、netstat等命令
$ sudo apt install openssh-server && sudo systemctl start ssh.service # SSH服务端程序
$ sudo apt install wget curl rsync iperf3 wireshark nethogs
$ sudo apt install aria2 # 即aria2c命令
$ sudo apt install tftpd-hpa
````

## 4、构建**日常开发环境**

### 4.1 安装基础的`编译器`以及配套工具

````
$ sudo apt install build-essential
$ sudo apt install gcc g++
$ sudo apt install make cmake
$ sudo apt install bison flex
````

### 4.2 安装`版本控制系统`程序

````
$ sudo apt install git # Git
$ sudo apt install subversion # SVN
````

### 4.3 安装`源码编辑器`

* `VIM`：详见[这篇文章](https://foofoodamon.github.io/懒人VIM技巧.html)。

* `gedit`：`sudo apt install gedit`

### 4.4 安装`源码对比程序`

````
$ sudo apt install meld
````

### 4.5 安装`源码静态检查工具`

````
$ sudo apt install clang cppcheck
````

### 4.6 安装`脚本`工具

````
$ sudo apt install python3-full python3-pip

$ sudo apt install expect
````

### 4.7 其他

````
$ sudo apt install valgrind # 内存泄漏检测

$ sudo apt install bear # 配合clangd或VIM插件YouCompleteMe使用

$ sudo apt install libssl-dev # 编译Linux内核用到
````

## 5、安装`影音软件`

````
$ sudo apt install smplayer # 视频播放器

$ sudo apt install obs-studio # 屏幕录播软件
````

## 6、安装`图形图像`处理工具

### 6.1 应用程序`库`及`框架`

````
$ sudo apt install qmake6 && \
    sudo update-alternatives --install $(dirname $(which qmake6))/qmake qmake $(which qmake6) 100
$ sudo apt install qt6-base-dev qt6-multimedia-dev # 基础库、多媒体（影音）库
$ sudo apt install qt6-tools-dev-tools # Qt助手、设计器

$ sudo apt install libopencv-dev libfreeimage-dev
````

### 6.2 `绘图`及`建模`软件

````
$ sudo apt install gimp krita

$ sudo apt install kicad blender
````

### 6.3 `摄像头调试`工具

````
$ sudo apt install gstreamer1.0-libav
$ sudo apt install gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-ugly
$ sudo apt install gstreamer1.0-plugins-bad
````

## 7、安装`嵌入式`相关工具

````
$ sudo apt install minicom # 串口数据收发工具

$ sudo apt install gcc-arm-none-eabi
$ sudo apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu # 还可指定版本号，例如：gcc-11-aarch64-linux-gnu

$ sudo apt install stm32flash # STM32系列芯片的命令行烧录程序
$ sudo apt install openocd # 一款开源的嵌入式片上调试工具
````

## 8、其他

````
$ sudo apt install dos2unix
$ sudo apt install parallel

$ sudo apt install procps # 包含watch命令
$ sudo apt install lm-sensors stress sysbench hwinfo # 硬件信息采集、状态监控、基准/压力测试

$ sudo apt install wkhtmltopdf # PDF转换
````

