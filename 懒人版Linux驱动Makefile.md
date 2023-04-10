<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# 懒人版Linux驱动Makefile

[<<<< 返回主页](README.md)

---------------------------------------------------------------------------

## 1、背景

就是懒。看了很多教程示例，自己也亲手写过，觉得见过的`Linux`驱动`Makefile`都大同小异，
既不想每次都要在一大片相同之中找不同而浪费精力，也不想重复这种毫无意义的复制粘贴，
于是乎就催生了这样一份通（懒）用（人）版`Makefile`。

## 2、脚本内容

第一手核心内容先上，原理解析以及完整版脚本内容见后面：

````
STRIP ?= arm-linux-gnueabihf-strip
NDEBUG ?= y

KERNEL_ROOT ?= ${HOME}/src/linux
ifeq (${DRVNAME},)
    export DRVNAME := $(basename $(notdir $(shell find ./ -name "*.c" | grep -v '_app\.c$$' | head -n 1)))
    ifeq (${DRVNAME},)
        $(error DRVNAME not specified and not deductive)
    endif
endif
obj-m := ${DRVNAME}.o
ccflags-y += -D__VER__=\"${__VER__}\"
ifeq (${NDEBUG},)
    ccflags-y += -O0 -g
endif

APP_NAME ?= $(basename $(shell find ./ -name "*.c" | grep '_app\.c$$' | head -n 1))
APP_OBJS ?= ${APP_NAME}.o
APP_CC ?= arm-linux-gnueabihf-gcc
ifeq (${NDEBUG},)
    APP_DEBUG_FLAGS ?= -O0 -g
else
    APP_DEBUG_FLAGS ?= -O2 -DNDEBUG
endif
APP_CFLAGS ?= -D_REENTRANT -D__VER__=\"${__VER__}\" -fPIC -Wall \
    ${APP_DEBUG_FLAGS} ${APP_DEFINES} ${APP_INCLUDES} ${OTHER_APP_CFLAGS}

all: ${DRVNAME}.ko ${APP_NAME}

${DRVNAME}.ko:
	make -C ${KERNEL_ROOT} M=`pwd` modules
	[ -f $@ ] || mv $$(ls *.ko | head -n 1) $@
	[ -z "${NDEBUG}" ] || ${STRIP} -d $@

${APP_NAME}: ${APP_OBJS}
	${APP_CC} -o $@ -fPIE $^ ${APP_LDFLAGS}
	[ -z "${NDEBUG}" ] || ${STRIP} -s $@

%.o: %.c
	${APP_CC} ${APP_CFLAGS} -c -o $@ $<

debug:
	make NDEBUG=""

clean:
	rm -f ${APP_NAME} ${APP_OBJS}
	make -C ${KERNEL_ROOT} M=`pwd` clean
````

将以上内容保存到一个名为`linux_driver.mk`的文件。

## 3、用法

在详细讲解脚本的原理之前，先讲一下如何使用，以获得一些感性认识，
有助于后面的理解。

### 3.1 最简单的小测试场景

这种场景最常见，新手入门和老手日常小测试都属于这类，特点是文件少、
代码简单甚至就是`玩具代码`（`Toy Code`），`Hello World`就是一个典型例子。
假设你有很多小测试代码，目录结构如下：

````
/path/to/test/root/directory
    |-- test1
    |       `-- test1.c
    |-- test2
    |       |-- test2.c
    |       `-- test2_app.c
    .   .
    .   .
    .   .
    `-- testN
            `-- testN.c
````

以上所示的文件其实有一定的规律和命名要求，但现在先不管，
只需知道要为符合这些规律和要求的测试代码写`Makefile`是很简单的事即可，
基本不必做太多的改动和定制化。如果是做`ARM`嵌入式`Linux`驱动的，
**甚至不需要写`Makefile`**，而只需要**作一个软链接即可**，
以`test1`为例（假设`linux_driver.mk`和`Linux`源码均保存在`~/src`目录）：

````
$ ln -s ${HOME}/src/linux_driver.mk /path/to/test1/Makefile
````

如果是做其他平台的驱动，例如`X86`，就需要稍微多一点工作。
可以创建一个`${HOME}/src/x86_driver.mk`，并写入以下内容：

````
STRIP := strip
APP_CC := gcc
include ${HOME}/src/linux_driver.mk
````

最后再为每个测试创建一个`Makefile`软链接，仍以`test1`为例：

````
$ ln -s ${HOME}/src/x86_driver.mk /path/to/test1/Makefile
````

### 3.2 用于正式项目的场景

这种场景稍微复杂一些，不能直接套用模板，要做一些定制化，
例如组织多个源文件、修改编译参数、定义应用程序或驱动的版本号等，
但也不难。假设项目目录层次如下：

````
/path/to/project/source/code/directory
    |-- common
    |       `-- __ver__.mk # 内含软件版本号的定义，这里不必深究
    |-- app
    |-- kernel # Linux内核源码目录
    `-- drivers
            |-- driver1
            |       |-- driver1_main.c
            |       |-- driver1_utils.c
            |       |-- driver1_app_main.c
            |       |-- driver1_app_utils.c
            |       `-- Makefile
            |-- driver2 # 内容与driver1相似
            .       .
            .       .
            .       .
            `-- driverN # 内容与driver1相似
````

先将`linux_driver.mk`放入以上`common`目录。

再按实际情况修改每个测试的`Makefile`，以`driver1`为例：

````
KERNEL_ROOT := ${PWD}/../../kernel
DRVNAME := driver1
${DRVNAME}-objs := driver1_main.o driver1_utils.o
APP_NAME := driver1_app
APP_OBJS := driver1_app_main.o driver1_app_utils.o
# 根据需要再设置其它变量：STRIP、APP_CC、APP_DEFINES、……
include ${PWD}/../../common/__ver__.mk
include ${PWD}/../../common/linux_driver.mk
````

## 4、原理解析

* **核心**：**尽可能地自动推断出**待编译的**目标**和相关**资源**！
为此，对文件和目录的名称、路径等组织形式要作一定的约束，如下：
    * 若`Linux`内核源码目录的路径是`${HOME}/src/linux`，则可自动使用，
    否则要对`KERNEL_ROOT`变量手动赋值以指明路径。
    * 若驱动源码文件只有一个且文件名不以`_app.c`结尾，
    则可自动推断最终的驱动名称，否则要对`DRVNAME`变量手动赋值以指定驱动名。
    详见`ifeq (${DRVNAME},)`语句块。
    * 驱动名称确定之后，`obj-m`变量的值也随之确定。
    * 若配备用于演示驱动用法的应用程序，并且其源码文件只有一个、
    文件名以`_app.c`结尾，则可自动推断最终的应用程序名称，
    否则要对`APP_NAME`变量手动赋值以指定应用名称。反之，若不配备应用程序，
    则按以上逻辑就会推断出一个空名称，亦即导致了一个空目标的出现，
    在编译阶段会自动跳过，无需手动置空。
    * 应用程序的中间产物默认为`${APP_NAME}.o`，若是多个源码文件的场景，
    需要对`APP_OBJS`变量手动赋值。

* 其他：
    * 尽可能地利用`Linux`内核的编译框架，不随便重复造轮子增加复杂度，
    重点语句是：
        ````
        make -C ${KERNEL_ROOT} M=`pwd` modules
        ````
    * 利用`ccflags-y`变量为驱动追加编译参数，例如自定义版本号、
    编译时保持调试符号等，以便于程序调试和版本管理。
    * 一个`Makefile`同时集成驱动和应用的编译逻辑，
    而网上常见的教程通常会分开两个`Makefile`，或直接使用命令来编译应用程序，
    错失了`make`程序可自动检测依赖关系链之中每个节点是否有更新的机制带来的好处。
    * 利用`Makefile`自带的`模式规则`语法，使得应用程序的`.c`转`.o`更简洁、通用，
    详见`%.o: %.c`语句块。
    * 增加了`调试/发布`模式的选择逻辑，以便在发布时能剥除调试符号，
    既缩小文件的体积，也增加了破解难度。

## 5、完整脚本

详见[懒编程秘笈](https://github.com/FooFooDamon/lazy_coding_skills)项目的`makefile/linux_driver.mk`，
内容与本文的脚本无多大差别，主要是参数和注释更详细，且后续若有更新，仅更新`GitHub`项目，
本文的脚本内容不会再同步。

此外，由于`Markdown`插件的影响，本文的某些`水平制表符`（`Tab`）可能会转成空格，
导致直接复制本文的脚本内容来使用可能会报错，建议直接使用`GitHub`项目的脚本。

---------------------------------------------------------------------------

[<<<< 返回主页](README.md)

