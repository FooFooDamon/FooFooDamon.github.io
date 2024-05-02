<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# Shell脚本编程的高级技巧与陷阱

<a href="记不住又用得多的东东.html" target="_top">&lt;&lt;&lt;&lt; 返回上层</a>

---------------------------------------------------------------------------

**注**：本文的`Shell`既可表示`外壳`（相对于操作系统内核而言）接口**解释器**，
也可表示其**语法规范**，且以**桌面版**（相对于裁剪过的嵌入式版本更完整）`bash`为主，
尽量兼容`sh`，至于`csh`、`zsh`等暂不涉及。

## 1、条件判断

* `test`、`[`、`[[`的区别：
    * `[`是`test`的同义词（均可通过执行`man test`来查看其使用说明），但`[`必须与`]`配对使用。
    * `[[`是`bash`特有（`sh`无）的语法，比`[`更强大（但若考虑兼容性则仍推荐`[`），
    例如：
        * 扩充了条件表达式的写法：
            * 例1：既支持`[[ $var -gt 1 ]]`（或使用`[]`），也支持`[[ $var > 1 ]]`。
            * 例2：既支持`[[ $var -gt 1 -a $var -lt 5 ]]`（或使用`[]`。另：逻辑“或”使用`-o`，
            逻辑“非”使用`!`），也支持`[[ $var > 1 && $var < 5 ]]`。
        * 容错性更高，但也同时带来一些隐患：例如上面的`var`若未定义则`$var`为空，
        按`NUL`字符的`ASCII`码与`1`的`ASCII`码进行比较，不会报错，但`[`语法会报错，
        两者有利有弊，可根据使用场景进行选用。

* 应使用单等号（`=`）还是双等号（`==`）：推荐前者，后者在`sh`中会报错（`unexpected operator`）。

## 2、变量

* 变量赋值时，等号两边不允许有空格，即只能写成`VAR=VALUE`，而不能写成`VAR = VALUE`。

* 只有函数内部的变量才允许使用`local`关键字修饰。

* `$*`作为一个参数，`$@`作为多个参数（以数组的形式），两者在使用双引号括起时区别最明显，例如：
    ````
    $ cat test1.sh
    for i in "$*"
    do
        echo "${i}"
    done
    $
    $ sh test1.sh aa "bb cc" dd ee
    aa bb cc dd ee
    $
    $ cat test2.sh
    for i in "$@"
    do
        echo "${i}"
    done
    $
    $ sh test2.sh aa "bb cc" dd ee
    aa
    bb cc
    dd
    ee
    ````

* 特殊变量：
    * `$*`、`$@`、`$#`：`$*`和`$@`见上面，`$#`表示位置参数（见下面）的数量。
    * `$1`、`$2`、`$3`、……：位置参数。注意编号为两位数以上的参数，最好用花括号将编号括起。
    * `$0`：当前脚本路径。
    * `$?`：上一个操作的返回值。
    * `$$`：当前进程（既可为交互式`Shell`环境，也可为运行中的脚本）的进程号（即`PID`）。
    * `$!`：截至目前在当前`Shell`运行的最后一个后台进程的`PID`。

* 确保变量值非空：可使用`${VAR-FALLBACK}`形式来获取变量值，例如：
    ````
    $ echo ${nonexistent_var-OOXX}
    OOXX
    ````

## 3、字符串

* 删除：以`filename=aa.bb.cc.txt`为例：
    * 删除前缀（保留后缀）：
        * 从**左**到右非贪婪匹配：
            ````
            $ echo ${filename#*.}
            bb.cc.txt
            ````
        * 贪婪匹配：
            ````
            $ echo ${filename##*.}
            txt
            ````
    * 删除后缀（保留前缀）：
        * 从**右**到左非贪婪匹配：
            ````
            $ echo ${filename%.*} # 与Makefile的“茎”语法类似，可借此助记
            aa.bb.cc
            ````
        * 贪婪匹配：
            ````
            $ echo ${filename%%.*}
            aa
            ````

* 替换：以`filename=aa.bb.cc.txt`和`path=/aa/bb/cc/dd`为例：
    * 非贪婪匹配：
        ````
        $ echo ${filename/./-}
        aa-bb.cc.txt
        $
        $ echo ${path/\//\\}
        \aa/bb/cc/dd
        ````
    * 贪婪匹配：
        ````
        $ echo ${filename//./-}
        aa-bb-cc-txt
        $
        $ echo ${path//\//\\}
        \aa\bb\cc\dd
        ````

* 获取子符串长度：`${#string}`

* 提取子串：`${string:<start>:<count>}`，不能使用负数作为索引。

## 4、顺序型数组

* 获取数组长度：`${#array[@]}`

* 提取子集：`${array[@]:<start>:<count>}`，不能使用负数作为索引。

* 访问单个元素：`${array[<i>]}`，正数索引范围为`[0, count)`，`>= count`不会报错但值为空；
负数索引范围为`[-count, -1]`，`< -count`会报错。

* 获取所有索引（下标）：`${!array[@]}`

* 判断是否包含某元素：可使用`正则等号`（即`=~`）和`双层方括号`（即`[[`和`]]`），有`2`种写法：
    * `[[ " ${array[*]} " =~ " xx " ]]`：通过在两端添加空格，使每个元素具有相同的格式。
    * `[[ ${array[*]} =~ (^|[[:space:]])xx($|[[:space:]]) ]]`：写法更复杂，
    但仍是对正则表达式的运用。

* 删除元素：`unset array[<i>]`。特别地，删除最后一个元素既可用`unset array[$((${#array[@]} - 1))]`，
也可用`unset array[-1]`。

* 关于顺序型数组更详细的介绍可查阅[这篇文章](https://www.mybluelinux.com/bash-guide-to-bash-arrays/)或其[备份文档](references/bash-guide-to-bash-arrays.pdf)。

## 5、关联数组

待补充

## 6、控制选项

* 在交互式环境（即命令行界面）默认启用此机制；非交互式环境（即脚本）则默认禁用，
若要启用则需要使用`shopt -s expand_aliases`。

* 关于`set`命令的详细介绍可查阅[这篇文章](https://phoenixnap.com/kb/linux-set#:~:text=The%20set%20command%20is%20a%20built-in%20Linux%20shell,%28sh%29%2C%20C%20shell%20%28csh%29%2C%20and%20Korn%20shell%20%28ksh%29)或其[备份文档](references/linux_set_command_usage.pdf)。
还要注意`set -x`的输出顺序不一定准确，不要被误导。

## 7、正则表达式

* 若要在`grep`命令使用`水平制表符`，则不能直接用`\t`，而要用`$'\t'`，
但表达式稍复杂一点可能会显得很混乱，所以在允许同时搜`空格`和`水平制表符`的前提下，
更推荐使用`[[:blank:]]`（等同于`[ \t]`）或`[[:space:]]`（等同于`[ \t\r\n\v\f]`）。

---------------------------------------------------------------------------

<a href="记不住又用得多的东东.html" target="_top">&lt;&lt;&lt;&lt; 返回上层</a>

