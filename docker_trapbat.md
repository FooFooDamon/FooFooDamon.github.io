<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# 一齐来笠飞鼠

## 1、背景

* `笠`作动词时意为拿一个麻包袋或类似东西对着目标人物**当头**`套下去`；
`飞鼠`即`蝙蝠`，代号`BAT`，大家应该不陌生。

* `自然界的`飞鼠通常视为`益兽`，`网上虚拟世界的`飞鼠`却不然`，
尤以其**窥探大众隐私**之行径令人作呕。

* 如果无能力将其扑杀时，只能想办法将其囚入牢笼。

* 飞鼠只是当前的一个典型，对待其余类似的蛇虫鼠蚁，也可如法炮制。

* 寓言式开场白完毕，下面介绍方法技巧。

## 2、自制一个“鼠”笼——支持图形化（GUI）程序的docker容器

### 2.1 适用环境

* 适用于`Linux`，本文以`Ubuntu`为例进行介绍，其余发行版需要适度修改部分命令。

* `Windows`可能需要借助`WSL`子系统，本人没试过，详情可参考后文的链接。

### 2.2 原理简介

* `Linux`图形化桌面环境采用的是`C/S`（`客户端/服务器`）架构，一个的典型解决方案是`X11`。

* 众多的`GUI应用程序`作为客户端，`X11`作为服务器，`应用程序`将内容发给`X11`，
就能显示到显示器上，且两者间的通信不限于进程间通信，也能走网络，从而支持跨机器的场景。

* `Linux`下（几乎）一切皆文件，包括网络`套接字`（`Socket`）。
将宿主系统（须安装并运行图形化环境）图形化相关的`套接字`等文件共享给`docker`容器，
这样容器也能像宿主系统那样使用`X11`机制了。

### 2.3 容器的制法

````
$ export BAT_IMAGE=ubuntu:22.04 # 或其他版本
$
$ export BAT_CONTAINER=fuck_bat # 或其他名称
$
$ export SHARED_DIR=${HOME}/${BAT_CONTAINER} # 或其他目录
$
$ docker run --name=${BAT_CONTAINER} -ti \
    --network=host \
    -v $(realpath ${SHARED_DIR}):$(realpath ${SHARED_DIR}) \
    -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
    -v /run/dbus/system_bus_socket:/run/dbus/system_bus_socket:ro \
    -v /run/user/${UID}:/run/user/${UID}:ro \
    -e DISPLAY=unix${DISPLAY} \
    -e DBUS_SESSION_BUS_ADDRESS=${DBUS_SESSION_BUS_ADDRESS} \
    ${BAT_IMAGE} \
    bash
````

需要解释几点：

* `SHARED_DIR`是一个在宿主系统与容器之间**共享文件和数据的目录**，需要自行创建、指定。
若不需要可删这一行，但建议使用，因为这是导入导出文件和数据的最简单方法。同时，
目录的包含内容不宜太多、范围不宜太广，否则就与限制飞鼠的初衷相违背了。

* 其余几个`-v`选项**挂载的目录和文件**，是实测后可满足使用的最小集合。
若换了不同版本的`Ubuntu`镜像，或者其他`Linux发行版`镜像，有可能需要更多的文件，
可观察应用程序运行时的报错，自行添加。

* 要注意目录和文件**权限**，若`只读`（`ro`）权限就满足使用要求，
就没必要使用`可读可写`（`rw`）权限。

* 除了要挂载必要的目录，还要创建**必要的环境变量**，详见以上含`-e`选项的行。

最后，若不想敲一大堆命令，可下载“[懒编程秘笈](https://github.com/FooFooDamon/lazy_coding_skills)”项目的
`scripts/docker_trapbat.sh`脚本来执行。

### 2.4 容器参数的修改（有需要时才使用）

正如前一小节所说，在后续的调试中有可能要根据报错的内容来挂载更多的目录或文件。
若先删除容器，再用前述的命令加上新的挂载目录或文件的选项重新创建容器，还要安装必要的依赖项，
显然费时费力，不值得提倡。更好的做法是修改已有容器的参数，直接增加待挂载的目录或文件，
操作如下：
````
$ # 关闭docker服务
$ sudo systemctl stop docker.*
$
$ # 获取目标容器的ID
$ export BAT_CONTAINER_ID=$(docker ps -a --no-trunc --filter name="^/${BAT_CONTAINER}$" --format '{{.ID}} {{.Names}}' | awk '{ print $1 }')
$
$ # 打开参数文件并修改，修改前最好先备份一下。
$ # 若要添加挂载目录，可在MountPoints字典增加一个子项；
$ # 若要添加环境变量，可在Env数组增加一个子项。
$ sudo vim /var/lib/docker/containers/${BAT_CONTAINER_ID}/config.v2.json
$
$ # 重新启动docker服务
$ sudo systemctl start docker.service
$
$ # 肉眼检查参数是否修改成功
$ docker inspect ${BAT_CONTAINER}
````

### 2.5 宿主系统的配置

````
$ # 安装X11的访问控制程序
$ sudo apt install x11-xserver-utils
$
$ # 先查看一下允许访问的客户端列表（即白名单）
$ xhost
$
$ # 再将前文创建的docker容器账户加入白名单。
$ # 注意：由于容器是本地，所以第二个字段用localuser，而第三个字段则是容器的用户名，
$ #      通常是root，其余字段可查阅xhost的使用手册（执行命令：man xhost）。
$ xhost +SI:localuser:root
$
$ # 检查一下白名单是否已包含刚才所加的账户
$ xhost
$
$ # 最后开启仅白名单可访问的模式
$ xhost -
````

## 3、进一步完善“鼠”笼的环境

以下操作均在容器内进行。由于是`root`用户，命令行提示符也由`美元符号`（`$`）变成`井号`（`#`），
但为了与`注释前导符`（也是`#`）区分开，还是继续使用`$`。

### 3.1 换软件源以获取更快的软件包下载速度

````
$ # 先备份软件源配置文件
$ cp /etc/apt/sources.list /etc/apt/sources.list.original
$
$ # 再换成清华源，读者也可使用阿里云或其他的源
$ sed -i "s/http:\/\/[a-zA-Z0-9.]*ubuntu\.com/https:\/\/mirrors.tuna.tsinghua.edu.cn/g" /etc/apt/sources.list
$
$ # 更新包管理器缓存
$ apt update -y
````

### 3.2 配置中文环境

````
$ # 安装语言包
$ apt install -y language-pack-zh-hans language-pack-zh-hant
$
$ # 设置环境变量
$ export LANG=zh_CN.UTF-8
$ echo "export LANG=zh_CN.UTF-8" >> $HOME/.bashrc
$
$ # 安装常用的中文字体
$ apt install -y fonts-wqy-microhei fonts-wqy-zenhei fonts-arphic-ukai fonts-arphic-uming
````

需要区分一下：
* `语言包`和`环境变量`是为了解决中文**乱码**或**显示成一串问号**的问题（通常发生在**终端**）。
* `中文字体`是为了解决中文**显示成方块**的问题（通常发生在具体软件的图形界面上）。

### 3.3 安装其他工具程序（可选，但推荐）

````
$ # Linux下非常流行的编辑器
$ apt install -y --install-recommends vim
$
$ # 常用的网络小工具
$ apt install -y net-tools iputils-ping curl wget
$
$ # 编译套件，含make和gcc
$ apt install -y build-essential
````

## 4、“调教”飞鼠

假设有只飞鼠叫`xx`，其安装包叫`xx.deb`，那么可将其放入前文的共享目录，然后安装：
````
$ dpkg -i ${SHARED_DIR}/xx.deb
````

毫无悬念会报错，因为复杂的软件会有很多依赖项，而这些依赖项在原始的`Ubuntu`系统中肯定没有安装。
挑选其中一个看看，例如`libgtk-3-0`，上网查一下，或者在宿主系统利用`水平制表符`（`Tab`）补全机制看一看，
其对应的软件包也叫`libgtk-3-0`，于是安装一下：
````
$ apt install libgtk-3-0
````

出现更多的报错，皆因依赖项还有自己的依赖项。不过这次也出现了更有用的提示，
按提示来修复一下：
````
$ apt --fix-broken install
````

下载安装了一大堆依赖项，再继续安装飞鼠软件，就能成功安装了。
安装后的程序通常位于`/opt/xx`目录下，快捷启动方式则位于`/usr/share/applications`。
假设其主程序仍叫`xx`，运行一下看看会发生什么：
````
$ /opt/xx/xx
````

出现以下报错：
````
/opt/xx/xx: error while loading shared libraries: libgbm.so.1: cannot open shared object file: No such file or directory
````

按照前面安装缺失依赖项的思路，安装`gdm`库：
````
$ apt install libgbm1
````

还没完，又报告缺失另一个库：
````
/opt/xx/xx: error while loading shared libraries: libasound.so.2: cannot open shared object file: No such file or directory
````

继续安装`asound`库：
````
$ apt install libasound2
````

若还有缺失，继续如法炮制。一番操作过后，有了新进展——报错内容变了：
````
Running as root without --no-sandbox is not supported. See https://crbug.com/638180.
````

看上去“熟口熟面”，一查果然是`Electron`框架所导致的错误，大致原因与`沙盒`机制有关。
由于`docker`容器已经作了资源隔离，与`沙盒`类似，所以在容器内即使禁用`沙盒`也没什么问题，
按其要求加上相应的选项，再运行一次就行了：

````
$ /opt/xx/xx --no-sandbox
````

至此，飞鼠应该已经“调教”好，能正常玩耍了。如果有一只以上的飞鼠，可以让以上命令后台运行，
甚至忽略大部分程序日志，例如：
````
$ /opt/xx/xx --no-sandbox > /dev/null &
````

完事之后可以按`Ctrl`+`p`、再按一下`q`来退出容器。再次进入则为`docker attach ${BAT_CONTAINER}`。

## 5、参考链接

* https://blog.csdn.net/zzw3354353337/article/details/129056426

* https://bbs.huaweicloud.com/blogs/393738?utm_source=zhihu&utm_medium=bbs-ex&utm_campaign=other&utm_content=content

