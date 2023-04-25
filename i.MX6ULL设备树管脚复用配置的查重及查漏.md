<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />

# i.MX6ULL设备树管脚复用配置的查重及查漏

## 1、背景

* `i.MX6ULL`是一款基于`Cortex-A7`、具有很高性价比的`ARM`处理器，
从市场采用率和培训教程数量就可见一斑，这里不再展开介绍。

* 对于设备树中的管脚复用配置，既不能在不同功能模块中同时引用同一引脚，
又不能在某一功能模块中漏掉部分引脚，否则均不能正常工作，要避免这两种情况，
就必须进行**（检）查重（复）**和**查漏（补缺）**。

* 人肉检查既繁琐又不可靠，还不能复用，所以能用脚本解决就绝不人肉检查、
能自动化就绝不手工操作。

* 本文的思路和脚本主要针对`i.MX6ULL`，但稍加修改，
理论上亦可应用于其他具备相同格式规范（例如采用相同的管脚复用理念、
有预定义的管脚头文件等等）的`ARM`芯片，尤其是`i.MX`系列，但精力所限，
未能一一验证、适配。

## 2、查重

由于管脚`复用逻辑`都**集中定义在**若干个**头文件**，
且以**功能**和**地址（偏移量）**进行组织，所以可从这两方面入手，
即：先利用头文件枚举出所有`地址（偏移量）`，再查找出每个地址所关联的`复用特性名称`，
最后再逐个名称地在目标设备树文件进行`搜寻`，同一地址的管脚出现`2`次以上的复用则为重复。
核心脚本内容如下：

````
DTS_FILE=$1 # 通过命令行参数传递待检查的.dts文件的实际路径
KERNEL_ROOT=${HOME}/src/linux # 实际的Linux内核源码根目录路径
HEADER_FILES=$(ls ${KERNEL_ROOT}/arch/arm/boot/dts/imx6ul*-pinfunc*.h)

grep "^#define[[:space:]]\+MX6UL" ${HEADER_FILES} | awk '{ print $3 }' | sort | uniq | while read i
do
    echo ">>> 正在对寄存器地址偏移量为[${i}]的引脚复用进行查重……"
    grep "^#define[[:space:]]\+MX6UL" ${HEADER_FILES} | awk "{ if (\"${i}\" == \$3) print \$2 }" | sort | uniq | while read j
    do
        grep -n --color=auto "${j}" "${DTS_FILE}"
    done
done
````

更灵活、更完善的脚本详见
<a href="https://github.com/FooFooDamon/lazy_coding_skills" target="_blank">懒编程秘笈</a>
项目的`scripts/check_arm_iomux_repetition.sh`及其关联配置`scripts/.script_as_config`，
其详细用法可带上`-h`选项进行查看。

## 3、查漏

仍是基于**头文件**。仔细观察可发现，每个`复用配置项`的**命名均是有规律的**，即：
* 以`XXX_PAD_`开头；
* 紧跟`XXX_PAD_`之后、在**双下划线之前**的内容表示**主导性的**复用配置；
* **双下划线之后**的内容表示**最终的**复用配置；
* 将以上两种复用配置合并、去重、过滤子项，就得出所有的**复用配置组**；
* 以某一复用配置组为关键字进行搜索，即得出该组所需的所有引脚**复用配置项**。

按以上规律可写出核心脚本内容如下：

````
IOMUX=$1 # 通过命令行参数传递待查找的复用功能名称
KERNEL_ROOT=${HOME}/src/linux # 实际的Linux内核源码根目录路径
HEADER_FILES=$(ls ${KERNEL_ROOT}/arch/arm/boot/dts/imx6ul*-pinfunc*.h)
MASTER_GROUPS=(
    $(grep "^#define[[:space:]]\+MX6UL" ${HEADER_FILES} \
        | awk '{ print $2 }' | sed 's/^[^_]\+_PAD_\([^_]\+\)_.\+/\1/' | sort | uniq)
)
RESULT_GROUPS=(
    $(grep "^#define[[:space:]]\+MX6UL" ${HEADER_FILES} \
        | awk '{ print $2 }' | sed 's/^.\+__\([^_]\+\)_.\+/\1/' | sort | uniq)
)

if [ $(echo ${MASTER_GROUPS[@]} ${RESULT_GROUPS[@]} | sed 's/ /\n/g' | sort | uniq | grep -c "^${IOMUXC}\$") -eq 0 ]
then
    echo "*** 该复用功能名称不存在：${IOMUXC}" >&2
    exit 1
fi

grep "^#define[[:space:]]\+[A-Z0-9_]\+__${IOMUXC}" ${HEADER_FILES} | awk '{ print $2 }' | sort | uniq
````

以上内容仅用于说明核心逻辑，执行结果并不十分精确，更灵活、更完善的脚本详见
<a href="https://github.com/FooFooDamon/lazy_coding_skills" target="_blank">懒编程秘笈</a>
项目的`scripts/get_arm_iomux_pins.sh`及其关联配置`scripts/.script_as_config`，
其详细用法可带上`-h`选项进行查看。

理论上来说，若头文件的定义完整且无差错，则以上脚本可给出指定功能模块所需的全部引脚复用配置。
当然，如果还不放心，可以自行到后面的参考材料的相关章节查阅核对。若有脚本未尽之处，
后续会在本文进行文字补充。

## 4、使用建议

* 在添加某一功能所需的引脚复用配置时，可借助查漏脚本快速获得。

* 添加完所有功能的复用配置之后，再执行查重脚本检查不同功能之间是否存在引脚冲突。

## 5、注意事项

* 由于查漏脚本尽量按统一规则进行查找，而且部分功能可以在多个地方进行配置，
例如`FLEXCAN1`在`MX6UL_PAD_ENET1_*`、`MX6UL_PAD_LCD_*`、`MX6UL_PAD_SD1_*`、
`MX6UL_PAD_UART3_*`均可配置，所以会有冗余内容输出，自行选择其中一个即可。
而且，`i.MX6ULL`有绝大部分内容是引用`i.MX6UL`的，所以遇到极少数除前缀外的同名项，
注意要采用`MX6ULL`前缀的项即可。

* 查漏脚本仅输出引脚复用宏定义及其所在的头文件（方便查找），
但引脚本身的电气特性配置无法预知，需要使用者根据项目实际情况自行指定。
同时，查漏脚本是从`功能分组`的角度去查找信息，相当于头文件的`索引`，
与官方文档`以寄存器为单位`的组织形式不同，但两者配合使用会有很好的效果。

* 为了脚本正常工作，设备树的书写需要一定的规范，即严格使用官方头文件里的定义、
每个引脚配置内容单独成行。

* 所有路径都不能包含空格！不过包含中文字符应该没问题。

## 6、参考材料

* 《<a href="references/i.MX 6ULL Applications Processor Reference Manual.pdf" target="_blank">i.MX 6ULL Applications Processor Reference Manual</a>》
以下章节：
    * 32.5 IOMUXC SNVS Memory Map/Register Definition
    * 32.6 IOMUXC Memory Map/Register Definition

