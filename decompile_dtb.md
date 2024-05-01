<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# 论如何刨设备树的根

## 1、背景

* 在做嵌入式`ARM`系统移植过程中，出于某些原因，有时**需要获取**一个**设备树**二进制文件（`*.dtb`）
的**源码**（`*.dts`），这时就**需要反编译**。

* `Linux`内核源码里的`dtc`程序既可用于编译，亦可用于反编译，
但反编译的结果会**缺失所有标签（label）的引用情况**（原因详见后文分析），
可读性很差，为后续的分析工作带来不便。

* **本文的主要目标**就是**寻找还原标签引用情况的方案**。

## 2、原理简析

### 2.1 利用`dtc`程序进行反编译

命令如下：

````
$ export DTC=/path/to/linux/kernel/source/scripts/dtc/dtc
$
$ ${DTC} --sort --in-format=dtb --out-format=dts --out xxx.decompiled.dts xxx.dtb
````

**注意**：反编译出来的`xxx.decompiled.dts`会**缺失所有标签的引用情况**，
**所有引用的位置会填上一个十六进制数值**，此值实际上是`dtc`程序对某个标签处理后生成的一个**句柄**，
在`Linux`内核中用`phandle`（新）或`linux,phandle`（旧）来表示。之所以没有还原成标签，
是因为`dtc`程序只关心设备树的语法、技术规范，**无法（也没必要）知道每个配置项的格式**
（因为这是上层业务的范畴），从而**无法得知某一个数字的原含义是一个数字还是一个标签**。
例如`xxx.decompiled.dts`可能包含如下内容：

````
cpu@0 {
    /* ... */
    phandle = <0x06>;
};

cpu@100 {
    /* ... */
    phandle = <0x07>;
};

arm-pmu {
    /* ... */
    interrupt-affinity = <0x06 0x07 0x08 0x09 0x0a 0x0b 0x0c 0x0d>;
    phandle = <0x1ee>;
};
````

但以上内容在源文件`xxx.dts`里本应是这样的：

````
cpu_l0: cpu@0 {
    /* ... */
};

cpu_l1: cpu@100 {
    /* ... */
};

arm_pmu: arm-pmu {
    /* ... */
    interrupt-affinity = <&cpu_l0>, <&cpu_l1>, <&cpu_l2>, <&cpu_l3>, <&cpu_b0>, <&cpu_b1>, <&cpu_b2>, <&cpu_b3>;
};
````

为了达到以上的效果，或与之相近的效果，还需要进一步处理，详见后文。

### 2.2 获取每个句柄对应的节点名称

通过观察上一节的反编译示例，不难想到**借助`栈`这种数据结构**来保存节点名称，
并在遇到一个句柄定义语句（`phandle` = <0xNNN>）时就出栈一次，逐渐**构造出一个`phandle`数字作键**、
**节点名称作值的关联数组**，核心逻辑如下：

````
NODE_NAME_CHARSET="[-_@+,.0-9a-zA-Z]"
PHANDLE_ASSIGNMENT_REGEX="^[[:blank:]]*\(linux,\)*phandle = <"
node_name_stack=()
declare -A phandle_map

while read i
do
    if [ $(echo "${i}" | grep -c "${PHANDLE_ASSIGNMENT_REGEX}") -gt 0 ]; then
        phandle=$(echo "${i}" | sed 's/.*<\(0x[0-9a-z]\+\)>.*/\1/')
        phandle_map["${phandle}"]="${node_name_stack[-1]}"
        unset node_name_stack[-1]
    else
        node_name_stack+=($(echo "${i}" | awk '{ print $1 }'))
    fi
done <<< $(grep "${PHANDLE_ASSIGNMENT_REGEX}\|^[[:blank:]]*${NODE_NAME_CHARSET}\+ {$" xxx.decompiled.dts)
````

需要说明的是：此处**为何要获取**的是**节点名称**而非标签名？无他，仅因为前者**更易获取**而方便行文，
但要获取后者则还需要搜索`__symbols__`信息以及解析所遇节点的层级结构（为了处理不同节点重名的问题），
再加上要兼顾运行效率还会使用一些较为晦涩的语法，使复杂度增加不少，所以获取标签名的逻辑需要查阅文末的脚本链接，
而不会体现在文章内容。

### 2.3 整理含有标签引用的配置项格式

标签的引用分`3`种情况：

* 只在首位引用**一个**标签，例如：`reset-gpio = <&gpio6 7 GPIO_ACTIVE_HIGH>`

* 无间隔引用**多个**标签，例如前文的：`interrupt-affinity = <&cpu_l0>, <&cpu_l1>, <&cpu_l2>, <&cpu_l3>, <&cpu_b0>, <&cpu_b1>, <&cpu_b2>, <&cpu_b3>`

* 按**分组**引用，例如：`io-channels = <&adc 0>, <&adc 1>, <&adc 2>`

由于数量众多，通过命令行来指定肯定不现实；又不能写死在脚本里，这样达不到通用化的目的。
所以，自然而然的想法当然是**按一定的格式写入一个文件作为脚本的配置文件**，示例如下：

````
[single]
    reset-gpio
    ...
[/single]

[multiple]
    interrupt-affinity
    ...
[/multiple]

[in-groups-2-1]
    io-channels
    ...
[/in-groups-2-1]

# 还可按需添加in-groups-3-1、in-groups-3-2、in-groups-3-3、in-groups-3-1-2、
# in-groups-3-2-3、in-groups-4-1等等，只要目标设备树确实存在某种格式的配置项即可
````

至于**有哪些配置项，就需要人肉查找并写入**，有点繁琐，但难度并不大，这里就不展开说明了。
不过，这个配置文件是一次创建、重复使用，且型号相近的开发板还能互相参考。

在本节完结之前还剩下一个问题，就是脚本该**如何使用这个配置文件**？总不能配置方便编写但脚本逻辑难以理解吧？
其实不必有此担心，**还是使用关联数组**即可搞掂，核心逻辑如下：

````
declare -A PHANDLE_CONF_MAP
declare -A GROUP_SIZE_MAP
declare -A GROUP_POSITIONS_MAP

for i in single multiple $(grep "\[in-groups-[0-9-]\+\]" "${conf_file}" | sed "s/\[\(.\+\)\]/\1/")
do
    if [ "${i:0:9}" = "in-groups" ]; then
        tmp_positions=($(echo ${i#in-groups-*} | sed 's/-/ /g'))
        tmp_size=${tmp_positions[0]}
        unset tmp_positions[0]
    fi

    for j in $(sed -n "/\[${i}\]/,/\[\/${i}\]/p" "${conf_file}" | grep -v "\[\/*${i}\]")
    do
        PHANDLE_CONF_MAP["${j}"]="${i}"
        [ "${i:0:9}" = "in-groups" ] || continue
        GROUP_SIZE_MAP["${j}"]="${tmp_size}"
        GROUP_POSITIONS_MAP["${j}"]="${tmp_positions[*]}"
    done
done
````

至于配置文件`${conf_file}`更详细的内容，可查阅[此处](https://github.com/FooFooDamon/linux_porting/blob/main/orange-pi-5/5.10.160/arch/arm64/boot/dts/rockchip/fdt_phandle_rules.conf)。

### 2.4 找出含有标签引用的行，并进行还原

原理并不复杂，一句话概括就是**利用`正则表达式`、`grep`、`awk`、`sed`进行目标定位、**
**信息提取以及字符串拼接和替换**。经过前面的铺垫和准备，
**这一节的核心**其实只剩下**确定标签引用位置**的**算法**，
尤其是按分组引用时使用的`取余`运算（分组大小和标签位置的提取逻辑已在前一节给出）。
脚本核心逻辑如下：

````
grep -n "^[[:blank:]]*${NODE_NAME_CHARSET}\+ = <" xxx.decompiled.dts | while read i
do
    line_target_phandle=($(echo "${i}" | awk '{ print $1, $2, $4 }'))
    phandle_value=${line_target_phandle[2]:1}
    target_name=${line_target_phandle[1]}
    target_type=${PHANDLE_CONF_MAP["${target_name}"]}

    [ -n "${target_type}" ] && linenum=${line_target_phandle%%:*} || continue

    if [ "${target_type}" = "single" ]; then
        phandle_value=${phandle_value%%>*}
        phandle_name=${phandle_map[${phandle_value}]}

        sed -i "${linenum}s/^\([ \t]*${target_name} = <\)${phandle_value}\([^>]*>;\)/\1\&${phandle_name}\2/" xxx.decompiled.dts
    else
        [ "${target_type}" = "multiple" ] && group_size=0 || group_size=${GROUP_SIZE_MAP["${target_name}"]}
        [ ${group_size} -eq 0 ] && position_array=() || position_array=(${GROUP_POSITIONS_MAP["${target_name}"]})
        phandle_array=()
        index=0

        for j in $(echo "${i}" | sed "s/^${linenum}:[ \t]*${target_name} = <\([^>]\+\)>;/\1/")
        do
            if [ ${group_size} -eq 0 ]; then
                phandle_name=${phandle_map[${j}]}
            else
                [ $(echo "${position_array[*]}" | grep -c "\<$((${index} % ${group_size} + 1))\>") -eq 0 ] \
                    && phandle_name="" || phandle_name=${phandle_map[${j}]}
                index=$((index + 1))
            fi

            [ -z "${phandle_name}" ] && phandle_array+=("${j}") || phandle_array+=("\&${phandle_name}")
        done

        sed -i "${linenum}s/^\([ \t]*${target_name} = <\)[^>]\+\(>;\)/\1${phandle_array[*]}\2/g" xxx.decompiled.dts
    fi
done
````

### 2.5 性能优化要点

1. **循环内部**尽量**避免**派生**子进程**，因为创建进程**是相对重量级的操作**，
所以能用`Shell`（准确地说是`Bash`）的内置语法就尽量用，例如字符串的截取、替换使用`${VAR%%*}`、
`${VAR/xx/XX}`之类的语法，而不用`sed`、`awk`命令加管道，前者虽然不那么直观，但立竿见影、提速显著。

2. 在不会爆内存的前提下**尽量使用内存型文件**而非磁盘型文件，尤其是在机械磁盘读写大文件的情况。
通常`/tmp`目录是一个内存型文件系统，频繁的文件读写操作可考虑放到该目录下进行。

3. **尽量将搜索对象限制在小规模数据集**，例如先将数量较少的候选数据从原先的大文件提取出来存到一个单独文件，
后续的多轮匹配只从这个小文件中找。

4. 若情况允许，可考虑**将多种操作合并到同一个循环内进行**，例如查找所有节点对应的`phandle`、
在节点名称前面添加标签、创建`phandle`与标签的映射等操作都需要借助`栈`且循环条件接近。

5. `Python`、`Perl`等语言的运行效率通常比`Shell`要高，在大规模工程中几乎都是优于`Shell`的不二选择，
但在小规模工程和特定场合下与`Shell`差异不大，且复杂度比`Shell`要高，需要自行斟酌权衡，
本文使用`Shell`就因为业务逻辑用`Shell`足以描述，且运行耗时在可接受的范围内，
感兴趣的读者可自行尝试使用`Python`、`Perl`等语言重新实现一遍对比一下代码逻辑和运行耗时的差异。

6. 在脚本内容较多时要先**划分出若干个局部区块**并**测量出具体耗时**（例如使用`time`命令）再优化
——这一条无需多言，过早优化是万恶之源，无的放矢的优化等于白搞，前人不知强调多少次。

## 3、脚本详情

详见“[懒编程秘笈](https://github.com/FooFooDamon/lazy_coding_skills)”项目的`scripts/decompile_dtb.sh`。
本文为了简洁易懂，算法有所删减，而且使用了一些耗时语句，与原脚本有所差异，一切以脚本为准，
且后续若有更新，将不会同步到本文。

