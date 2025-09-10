<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# 极简嵌入式ARM知识点<br>——基于NXP公司i.MX6ULL处理器

## 1、前言

* 标题已说明一切。

* 建议先看文末的[温馨提示](#温馨提示)。

## 2、`ARM`版本的特点

* `具体处理器`和`基础架构`分别有各自的版本号：
例如`ARM7-TDMI`和`ARM920T`的架构版本分别为`v4`和`v4T`，`ARM946E-S`为`v5`，
`ARM1136J-S`为`v6`，等等。

* 从`v7`架构开始，处理器版本不再使用单一数字递增的命名方式，
而开始发布`Cortex-A`、`Cortex-R`、`Cortex-M`等系列，其中`A`表示`Application`、
`R`表示`Real-time`、`M`表示`Microcontroller`，应对不同场景。

* 上述每一系列再加上数字形成一个具体型号的处理器，
并且数字的大小不一定反映先后顺序和工艺性能，例如`A8`的数字比`A7`要大，
但`A7`在`A8`之后发布，且性能更强。

* 所以，要对比不同系列、不同型号处理器之间的性能特点，
只能从官方资料查证，不能简单以名称和数字进行推断。

* 可以从下表（源于网上）稍微感受一下命名的混乱（截至`v7`）：

架构版本号 | 处理器家族（不完全列举）
-- | --
`ARMv1` | `ARM1`
`ARMv2` | `ARM2`、`ARM3`
`ARMv3` | `ARM6`、`ARM7`
`ARMv4` | `StrongARM`、`ARM8`、`ARM7-TDMI`、`ARM9-TDMI`
`ARMv5` | `ARM7EJ`、`ARM9E`、`ARM10E`、`XScale`
`ARMv6` | `ARM11`
`ARMv7` | `ARM Cortex-A`、`ARM Cortex-M`、`ARM Cortex-R`

## 3、处理器模式

* 最初只有`7`种模式、归属于`2`类：
    * 非特权模式：
        * `USR`：User，用户模式，绝大部分程序，尤其是应用程序，均处于此模式。
    * 特权模式（两中断、两异常、一管理、一系统）：
        * `FIQ`：Fast Interrupt Request，快速中断模式。
        * `IRQ`：Interrupt Request，中断模式。
        * `SVC`：Supervisor，管理员模式，在使用特权指令访问受限资源时，进入此模式。
        * `ABT`：Abort，中止模式，当发生内存访问异常时，进入此模式。
        * `UND`：Undefined，未定义模式，当执行未定义指令时，进入此模式。
        * `SYS`：System，系统模式，操作系统就运行于此模式。

* 引入`信任区`（`TrustZone`）安全扩展机制后，新增以下`1`种模式：
    * `MON`：Monitor，监护模式，为处理器营造`安全`（`Secure`）和`非安全`（`Non-secure`）
    两种状态（State），其中前者通常运行**通用型**操作系统及应用程序，
    后者则运行各厂商**私有**固件以及安全相关的软件。每种状态下均能使用前述`7`种模式，
    而监控模式作为这两种状态的监护者和转换途径（Gateway）。

* 若再加入`虚拟化`（`Virtualization`）机制，则又增加`1`种模式：
    * `HYP`：Hypervisor，超级管理员模式，此模式介于操作系统与硬件之间，
    并将最初的`7`种模式划分成`2`种（在安全状态下）或`3`种（在非安全状态下）
    `权限级别`（Privilege Level），分别为：
        * `PL0`：应用于`USR`模式。
        * `PL1`：应用于`USR`和`HYP`以外的模式。
        * `PL2`：应用于`HYP`模式。

* 总结得出以下关系（整理自[参考材料2](#ref_2)的“Chapter 3 ARM Processor Modes and Registers”）：
    <table id="处理器模式关系表" border="1">
        <tr>
            <th>权限级别</th>
            <th>状态寄存器模式位</th>
            <th>非安全状态</th>
            <th>安全状态</th>
        </tr>
        <tr>
            <td>PL0</td>
            <td>10000</td>
            <td>USR</td>
            <td>USR</td>
        </tr>
        <tr>
            <td>PL1</td>
            <td>10001<br>10010<br>10011<br>10111<br>11011<br>11111</td>
            <td>FIQ<br>IRQ<br>SVC<br>ABT<br>UND<br>SYS</td>
            <td>FIQ<br>IRQ<br>SVC<br>ABT<br>UND<br>SYS</td>
        </tr>
        <tr>
            <td>PL2</td>
            <td>11010</td>
            <td>HYP</td>
            <td></td>
        </tr>
            <td>PL1</td>
            <td>10110</td>
            <td colspan="2"><center>MON</center></td>
        <tr>
        </tr>
    </table>

## 4、寄存器简介

### 4.1 按功能划分

* 通用寄存器：`R0`～`R15`。

* 帧指针：`FP`（`Frame Pointer`，由`R11`兼任）。

* 栈指针：`SP`（`Stack Pointer`，由`R13`兼任）。

* 链接寄存器：`LR`（`Link Register`，由`R14`兼任）；存放函数返回地址。

* 程序计数器：`PC`（`Program Counter`，由`R15`兼任）；
由于`3`级`流水线`（`取指` -> `译码` -> `执行`）的设计，
实际上存放的是当前正在执行的指令之后第`2`条指令。

* 程序状态寄存器：`xPSR`（`Program Status Register`），其中`x`有：
    * `C`：Current，当前。
    * `A`：Application，应用，是CPSR的替身，供用户模式下的程序使用，
    且仅限访问以下二进制位：`N`、`Z`、`C`、`V`、`Q`、`GE[3:0]`。
    * `S`：Saved，备份，也是CPSR的替身，当异常发生时，自动将之前的状态保存，
    但此寄存器在用户模式下不可访问。二进制位定义如下：

    序号 | 代号 | 作用
    -- | -- | --
    31 | N | Negative；运算结果为负时，置1
    30 | Z | Zero；运算结果为0时，置1
    29 | C | Carry-out；对于加法指令，发生进位则置1，否则清0；对于减法指令，发生借位则清0，否则置1。
    28 | V | oVerflowed；运算结果溢出时，置1。
    27 | Q | cumulative saturation；处于累积饱和状态时，置1。此位又称作粘滞位（sticky）。
    26:25 | IT[1:0] | If-Then；与后面的5位一起，反映条件指令的执行状态。
    24 | J | Jazelle；为1则表示处于Jazelle状态。
    23:20 | Reserved | 预留。
    19:16 | GE[3:0] | Greater Equal；大于等于，用于某些SIMD指令中。
    15:10 | IT[7:2] | 见前面`IT[1:0]`的说明。
    9 | E | Endianness；大端模式则置1，小端则清0。
    8 | A | asynchronous Abort；若要禁止异步中断，则置1。
    7 | I | Interrupt；禁止中断（IRQ）则置1，激活则清0。
    6 | F | Fast interrupt；禁止快速中断（FIQ）则置1，激活则清0。
    5 | T | Thumb；使用Thumb指令则置1，使用ARM指令则清0。
    4:0 | M[4:0] | processor Mode；处理器模式，详见[处理器模式关系表](#处理器模式关系表)。

* 协处理器（Coprocessor）寄存器：`c0`～`c15`，应用编程通常用不上，暂时忽略。

* 系统控制寄存器：`SCTRLR`（`System Control`），暂时忽略，理由同上。

### 4.2 按处理器模式进行分组与共享

<table border="1">
    <tr>
        <th>USR</th>
        <th>SYS</th>
        <th>FIQ</th>
        <th>IRQ</th>
        <th>ABT</th>
        <th>SVC</th>
        <th>UND</th>
        <th>MON</th>
        <th>HYP</th>
    </tr>
    <tr>
        <td colspan="9"><center>R0 ～ R7</center></td>
    </tr>
    <tr>
        <td colspan="2"><center>R8 ～ R12</center></td>
        <td><center>R8_fiq ～ R12_fiq</center></td>
        <td colspan="6"><center>R8 ～ R12</center></td>
    </tr>
        <td colspan="2"><center>R13（SP）</center></td>
        <td>SP_fiq</td>
        <td>SP_irq</td>
        <td>SP_abt</td>
        <td>SP_svc</td>
        <td>SP_und</td>
        <td>SP_mon</td>
        <td>SP_hyp</td>
    <tr>
        <td colspan="2"><center>R14（LR）</center></td>
        <td>LR_fiq</td>
        <td>LR_irq</td>
        <td>LR_abt</td>
        <td>LR_svc</td>
        <td>LR_und</td>
        <td>LR_mon</td>
        <td>R14（LR）</td>
    </tr>
    <tr>
        <td colspan="9"><center>R15（PC）</center></td>
    </tr>
    <tr>
        <td>APSR</td>
        <td colspan="8"><center>CPSR</center></td>
    </tr>
    <tr>
        <td>/</td>
        <td>/</td>
        <td>SPSR_fiq</td>
        <td>SPSR_irq</td>
        <td>SPSR_abt</td>
        <td>SPSR_svc</td>
        <td>SPSR_und</td>
        <td>SPSR_mon</td>
        <td>SPSR_hyp</td>
    </tr>
    <tr>
        <td colspan="8"><center>/</center></td>
        <td>ELR_hyp</td>
    </tr>
</table>

## 5、为何需要汇编语言？

1. **芯片上电**后，**C语言环境**还未准备好；

2. C语言环境主要单元为各个大大小小的**函数**；

3. C函数调用涉及**栈操作**；

4. 栈要由`SP指针`访问，而**SP指针**需要在上电后手动**初始化**，
只能通过汇编指令来实现（汇编与机器码有简单的对应关系，
只需经过简单的翻译，与直接使用机器码无异）。

5. 其他初始化工作，例如**DDR初始化**等，同理。

## 6、`CISC`架构与`RISC`架构

* 全称见后面的[缩略词表格](#缩略词)。

* `X86`基于`CISC`，可以用一条指令实现复杂的操作（一条指令实际上可能分解为若干微操作码），
处理某些特殊任务效率较高，但逻辑复杂，主要特点如下：
    1. 使用**（可）变长（度）**指令（格式不一难于优化），每条指令**可身兼多职**；
    2. 指令基本都**能直接访问储存器**（包括内存），寻址方式也多；
    3. 由于以上特点，有时编程书写较为方便；
    4. 各指令执行时间相差很大，多数需要**多个时钟周期**；
    5. 底层复杂度较高、粒度较粗，**难以**进行编译**优化**。

* 实践中发现了`二八定律`，于是提取出少量简单而又频繁使用的指令，便形成了`RISC`，
且原先复杂的指令也能由这些简单指令组合得到。

* `ARM`基于`RISC`，由少量的**通用**指令集构成，硬件实现简单，且低功耗，主要特点如下：
    1. 使用**（固）定长（度）**指令（格式统一便于优化），每条指令**职责单一**；
    2. **不可直接访问储存器**（包括内存），`加载`和`储存`指令除外，寻址方式也少；
    3. 由于上一点，几乎所有操作均需借助寄存器（但也减少对内存的操作，速度得以提升，
    详见下一点），故**通用寄存器很多**；
    4. 由于以上特点，有时编程书写较为麻烦；
    5. **必备**的**流水线**技术，使得大部分指令可在**一个时钟周期**左右完成，
    再配合**超标量**流水线技术，平均指令时间可降至一个时钟周期以下；
    6. 底层复杂度较低、粒度较细，**容易**进行编译**优化**。

* CISC与RISC的区别并不在于晶体管的多少（因为硬件集成度逐年提高），
或指令集的大小（因为新应用需求不断出现），而在于设计理念：
    * 内核是否足够简单，各项功能是否具有正交性，以便仅通过组合来实现更多特性；
    * 状态是否足够少，甚至无状态，以便并发、乱序执行；
    * `CISC`内部逻辑也有将复杂指令分解成多个微操作的做法，
    `RISC`设计也可以在综合考量成本和预算之后选择添加少量复杂指令，
    两者可以互相借鉴。

## 7、`ARM`指令集与`Thumb`指令集

* `i.MX6ULL`基于`Cortex-A7`，`Cortex-A7`属于`ARMv7`，`ARMv7`属于32位架构；

* `ARM`又是`加载/储存`（`Load/Store`）架构，即**数据处理全在`通用寄存器`进行**，
只有`加载`和`储存`指令会访问内存，而寄存器也是32位；

* 所以，`ARM`指令集是32位指令集，是最初、最基本的指令集。

* `Thumb`指令集则是16位指令集，首次出现在`ARM7TDMI`处理器，程序体积更小，
适用于资源不够丰富的嵌入式系统，但以牺牲部分性能为代价（类似`CISC`与`RISC`）。

* 因此，`ARM`指令集用于**性能至上**的场合（包括异常处理），
而`Thumb`则用于**储存不足**的场合，以及与16位内存、外设直连的场合。

* 在`ARMv6T2`架构引入`Thumb-2`技术（不是指令集！！），对`Thumb`指令集进行了扩展，
加入部分32位指令，使之既有与`ARM`相近的性能，又有与`Thumb`相近的小体积。
在`Cortex-A`系列处理器中，这项技术已是标配。

* 旧式语法会区分`ARM`指令与`Thumb`指令，新式语法已将两者统一，
该语法被称作`统一汇编语言`（Unified Assembly Language，`UAL`）。

* 实际开发中，既可使用`ARM`专有的编译工具，也可使用`GNU`编译工具，
而后者又比较通用，不限于编译`ARM`。在下文的介绍中，**语句格式使用`GNU`格式**，
**具体指令则使用`ARM`指令**。

## 8、`GNU`汇编语言通用格式

* 由一系列语句组成；

* 每行一条语句；

* 每条语句（Statement）的格式（方括号部分表示非必需）：`[<LABEL>:] [<INSTRUCTION>] [@ <COMMENT>]`

* 指令（Instruction）格式：`<NAME> <DESTINATION OPERAND> <SOURCE OPERAND>`

* 除注释（Comment）和寄存器名称外，各部分的书写可以全大写，也可以全小写，但不能大小写混用；

* 访问修饰符（Access Modifier）：
    * 立即数以`#`开头（对于`ARM`可选），但`ARM`的`LDR`指令的立即数要以`=`开头；
    * 其余待补充。

* 伪操作：汇编程序（Assembler）自己的指令，每条指令均以`句点`（`.`）开头，常见伪操作如下：
    * `.align` n：对数据段的数据，或代码段的`NOP`指令，执行`2^n`字节对齐。
    * `.ascii` "string"：定义`ASCII`字符串常量，没有`\0`结尾。
    * `.asciiz` "string"：同上，但有`\0`结尾。
    * `.{byte, hword, word}` expression：定义一个字节/半字/字的数据，多个数据则用逗号隔开。
    * `.data`：指示其后是`数据段`区域。详见[程序布局](#程序布局)。
    * `.end`：表示源文件结束。
    * `.equ` symbol, expression：将后面表达式的结果赋给前面的标签。
    * `.extern` synbol：声明一个外部符号（定义在别的源文件）。
    * `.global` symbol：定义一个全局符号，对其他源文件和链接程序均可见。
    * `.include` "filename"：包含一个文件进来，通常是头文件。
    * `.section`：自定义一个段。
    * `.text`：指示其后是`正文段`区域。详见[程序布局](#程序布局)。
    * 其余伪操作见官网：https://sourceware.org/binutils/docs/as/Pseudo-Ops.html

* 其余特性见官网：https://sourceware.org/binutils/docs/as/index.html

## 9、`ARM`汇编指令解析

* 参数（Parameter）顺序：多数情况下与赋值顺序一样（内存操作指令除外），
即目标操作数在前、源操作数在后，并且第一个操作数必须是寄存器，
第二个操作数既可以是立即数，也可以是寄存器，还可以是经过移位操作的寄存器，
详见[参考材料2](#ref_2)的“5.2.1 Operand 2 and the barrel shifter”小节。

* 几乎所有指令都可以配合`条件后缀`使用（其他架构大多只有分支跳转指令有条件判断），
这样就不必使用条件性分支跳转语句来实现条件判断逻辑，书写更简洁，但值得注意的是，
执行速度在某些处理器（例如`Cortex-A9`）却可能不如分支跳转，尤其是分支预测准确的情形下。
详见[参考材料2](#ref_2)的“5.1.2 Conditional execution”小节。条件后缀有：

    后缀 | 含义 | 受影响的CPSR标志位
    -- | -- | --
    EQ | Equal，等于 | Z = 1
    NE | Not Equal，不等 | Z = 0
    CS | Carry Set，进位置位，与`HS`效果一样 | C = 1
    HS | Unsigned Higher or Same，无符号型的大于或等于 | C = 1
    CC | Carry Clear，进位清零，与`LO`效果一样 | C = 0
    LO | Unsigned Lower，无符号型的小于 | C = 0
    MI | Minus，相减，或结果为负 | N = 1
    PL | Plus，相加，或结果非负 | N = 0
    VS | oVerflow Set，溢出标志置位 | V = 1
    VC | oVerflow Clear，溢出标志清零 | V = 0
    HI | Unsigned Higher，无符号型的大于 | C = 1 且 Z = 0
    LS | Unsigned Lower or Same，无符号型的小于或等于 | C = 0 或 Z = 1
    GE | Signed Greater than or Equal，有符号型的大于或等于 | N = V
    LT | Signed Less Than，有符号型的小于 | N != V
    GT | Signed Greater Than，有符号型的大于 | Z = 0 且 N = V
    LE | Signed Less than or Equal，有符号型的小于或等于 | Z = 1 或 N != V
    AL | Always，总是，默认值 |

* 多数指令还可以带`S`后缀（Set flags，设置标志），若有，则紧跟指令、写在`条件后缀`之前；
若无，则不会影响`CPSR`的相关标志位。

* 内存寻址（使用**方括号**）：
    1. `寄存器寻址`：使用寄存器里的值作为内存地址进行访问。
    例如：`LDR R0, [R1] /* 读取R1的值指向的内存并加载到R0 */`
    2. `前索引寻址`：对寄存器里的值加上一个偏移量，再作为内存地址进行访问，
    该偏移量可正可负、可以是立即数也可以是寄存器（可附带移位操作）。
    例如：`LDR R0, [R1, R2]`。
    3. `前索引寻址且回写`：末尾多写一个感叹号，表示在访问完内存之后，更新参与寻址的寄存器。
    例如：`LDR R0, [R1, #32]! /* R1里的值递增32 */`
    4. `后索引寻址且回写`：偏移量写在方括号之外，比第一种情形多了更新寻址寄存器的步骤。
    例如：`LAR R0, [R1], #32`

* 具体的指令可查阅：
    * [参考材料1](#ref_1)的“A4: The Instruction Sets”章。
    * [参考材料1](#ref_1)的“A8.8: Alphabetical list of instructions”小节。

## 10、程序布局（Layout）<a id="程序布局"></a>

### 10.1 保存在储存器的文件所呈现的布局

* 这里的储存器通常指磁盘（Disk）、闪存（Flash）等；

* 一个可执行文件由多个段组成，如下：

    <table border="1">
        <tr>
            <th>代号</th>
            <th>含义</th>
            <th>作用</th>
        </tr>
        <tr>
            <td>.bss</td>
            <td>以符号作为开头的块区域</td>
            <td>存放未初始化或初始化为0的全局变量和静态变量，在程序执行前自动清零，故此段<strong>不占储存器空间</strong>，仅<strong>标记</strong>所含全部<strong>变量占用的空间大小</strong>。</td>
        </tr>
        <tr>
            <td>.data</td>
            <td>数据段</td>
            <td>存放<strong>已初始化</strong>的<strong>非0</strong>全局变量和静态变量。</td>
        </tr>
        <tr>
            <td>.rodata</td>
            <td>只读数据段</td>
            <td>存放只读数据。是否一定存在，以及存在的位置，均有待确认。</td>
        </tr>
        <tr>
            <td>.text</td>
            <td>正文段</td>
            <td>存放程序代码，以及常量，故该区域通常设置为只读，但某些架构也允许写，即在运行期修改代码。</td>
        </tr>
    </table>

### 10.2 运行期加载到内存时所呈现的布局

* 除了前述的文件布局内容，还增加了以下内容：

    <table border="1">
        <tr>
            <th>代号</th>
            <th>含义</th>
            <th>作用</th>
        </tr>
        <tr>
            <td>heap</td>
            <td>堆</td>
            <td>运行期动态分配的内存段，从低地址向高地址扩张。</td>
        </tr>
        <tr>
            <td>stack</td>
            <td>栈</td>
            <td>存放非静态局部变量，从高地址向低地址扩张。</td>
        </tr>
        <tr>
            <td>mmap</td>
            <td>内存映射</td>
            <td>处于堆栈之间，扩张方向与栈相同。</td>
        </tr>
    </table>

* 总的（虚拟）内存布局（以32位系统为例）如下：

    <table border="1">
        <tr> <th>起始地址</th> <th>分段</th> </tr>
        <tr> <td>0xFFFFFFFF</td> <td>/</td> </tr>
        <tr> <td>0xC0000000</td> <td>kernel space (1GB)</td> </tr>
        <tr> <td>uncertain</td> <td>stack</td> </tr>
        <tr> <td>uncertain</td> <td>mmap</td> </tr>
        <tr> <td>uncertain</td> <td>heap</td> </tr>
        <tr> <td>uncertain</td> <td>.bss</td> </tr>
        <tr> <td>uncertain</td> <td>.data</td> </tr>
        <tr> <td>uncertain</td> <td>.text</td> </tr>
        <tr> <td>0x0</td> <td>/</td> </tr>
    </table>

    值得注意的是，部分段的起始地址不确定，是特意随机化以对抗破解攻击，
    不过32位地址空间有限，削弱了效果。

## 11、应用程序二进制接口（ABI）

待补充。

## 12、浮点数、SIMD、NEON

待补充。

## 13、缓存与缓冲

待补充。

## 14、内存管理单元（MMU）

待补充。

## 15、内存重排序、乱序执行

待补充。

## 16、异常处理

待补充。

## 17、中断机制<a id="中断机制"></a>

* `中断机制`为`突发事件`提供了一个`快速甚至是实时`的`处理手段`，
弥补了`忙等待`和`轮询`方式的不足。

* `中断机制`的核心是`中断服务`，其以`函数`的形式呈现，在`中断信号`产生之后**被调用**，
包含了该**如何处理该信号的操作逻辑**。

* 所有中断服务`函数的入口地址`组成一个表，叫`中断向量表`。
每个表项都是由体厂商预先定好的。当某个中断信号发生时，会自动跳转到对应的表项，
然后执行相应的中断服务函数。

* `中断向量表`放在整个程序的**最前面**。`ARM`处理器是从`0x0`地址开始执行，
但代码却是下载到0x08000000，中断向量表亦是，所以要借助中断向量表`偏移量`（对应
`SCB_VTOR`寄存器和`VECT_TAB_OFFSET`宏定义）进行校正。

* `Cortex-M`系列会将所支持的中断`平铺直叙`地列举到向量表（以确保简单性和实时性？？），
但`A`系列只有`7`个中断，并将所有外设中断都归入`IRQ`中断，
具体是什么中断可通过读取指定的寄存器来判断。`i.MX6ULL`的中断向量表如下：

    向量地址 | 中断类型 | 处理器模式
    -- | -- | --
    `0x00` | **复位**（Reset）| 特权模式（SVC）
    0x04 | 未定义指令（Undefined Instruction）| 未定义模式（UND）
    0x08 | 软中断（Software Interrupt，SWI）| 特权模式（SVC）
    0x0C | 预取指中止（Prefetch Abort）| 中止模式（ABT）
    0x10 | 数据访问中止（Data Abort）| 中止模式（ABT）
    0x14 | 预留未使用（Not Used）| /
    `0x18` | **IRQ**中断 | 外部中断模式（IRQ）
    0x1C | FIQ中断 | 快速中断模式（FIQ）

* `Cortex-A`系列管理中断的模块叫`通用中断控制器`（`Generic Interrupt Controller`），
简称`GIC`。`GIC`从外部接收中断信号并经过处理之后，再报告给`ARM内核`。

* `GIC`接收的中断源可分为`3`类：
    * `共享中断`：Shared Peripheral Interrupt，`SPI`，最常见最多，例如按键中断、串口中断，
    处理器的任一个核都可以对其进行处理。中断标识号范围是`32`～`1019`（为何不是1023？？），
    但实际只用到`127`，完整的列表详见[参考资料3](#ref_3)“3.2 Cortex A7 interrupts”小节。
    * `私有中断`：Private Peripheral Interrupt，`PPI`，每个核独有的中断，不能跨核处理。
    中断标识号范围是`16`～`31`。
    * `软件中断`：Software-Generated Interrupt，`SGI`，由软件触发（写`GICD_SGIR`寄存器），
    可用于多核之间的通信。中断标识号范围是`0`～`15`。

* `GIC`报告给`ARM内核`的有`4`个信号（后两个是针对虚拟化的，没用到的话一般不用管）：
    * `IRQ`：外部中断。
    * `FIQ`：快速中断。
    * `VIRQ`：虚拟外部中断。
    * `VFIQ`：虚拟快速中断。

* 从`逻辑架构`划分，则`GIC`可分为两块：
    * `分发器`：`Distributor`，决定中断事件应分发给哪个`处理器接口`，职责如下：
        1. 所有中断的全局开关；
        2. 每个中断的单独开关；
        3. 每个中断的优先级；
        4. 每个中断的目标处理器列表；
        5. 每个中断的触发模式：`电平`触发还是`边缘`触发；
        6. 每个中断属于组`0`还是组`1`（？？）。
    * `处理器接口`：`CPU Interface`，决定中断事件应发送到哪个`核`，职责如下：
        1. 激活或禁止发送给`处理器核心`的中断请求；
        2. 应答中断；
        3. 通知中断处理完成；
        4. 设置`优先级掩码`，该掩码可决定中断请求是否需要上报给处理器核心；
        5. 定义抢占策略；
        6. 多个中断同时到来时，只（？？）选择优先级最高的中断上报给处理器核心。

* `CP15`协议处理器：
    * 一般用于管理储存系统，但也会在部分中断场景用到。
    * `MRC`、`MCR`指令：待补充。
    * `CP15`寄存器：
        * `c0`：获取处理器内核信息。
        * `c1`：用于开关`内存管理单元`（MMU）、`指令缓存`（I Cache）、
        `数据缓存`（D Cache）等。
        * `c12`：设备中断向量表`偏移量`。
        * `c15`：获取`GIC`**基地址**。
        * 其余寄存器详见[参考资料1](#ref_1)的“B3.17 Organization of the CP15 registers in a VMSA implementation”小节。

* `中断开关`：
    * `全局开关`：除了可以使用`CPSR`的`I`标志位，还可以使用以下方式：
        * `CPSID i`：关闭`IRQ`中断，`ID`即`Interrupt Disabled`。
        * `CPSIE i`：打开`IRQ`中断，`ID`即`Interrupt Enabled`。
        * `CPSID f`：关闭`FIQ`中断。
        * `CPSIE f`：打开`FIQ`中断。
        * `CPS`指令详见[参考资料1](#ref_1)的“B9.3.2 CPS (ARM)”小节。
    * `独立开关`：
        * `GICD_ISENABLER0[0:15]`：负责`软件中断`（`SGI`）。
        * `GICD_ISENABLER0[16:31]`：负责`私有中断`（`PPI`）。
        * `GICD_ISENABLER1`～`GICD_ISENABLER15`：负责`共享（外设）中断`（`SPI`）。

* `中断优先级`：
    * 设置`优先级个数`：利用`GICC_PMR`寄存器的`低8位`（高24位未使用），设置如下：
        <table border="1">
            <tr> <th>bit[7:0]</th> <th>优先级个数</th> </tr>
            <tr> <td>11111111</td> <td>256个优先级</td> </tr>
            <tr> <td>11111110</td> <td>128个优先级</td> </tr>
            <tr> <td>11111100</td> <td>64个优先级</td> </tr>
            <tr> <td>11111000</td> <td><strong>32</strong>个优先级（Cortex-A7内核使用，包括i.MX6ULL）</td> </tr>
            <tr> <td>11110000</td> <td>16个优先级</td> </tr>
        </table>
    * 设置`抢占优先级级数`和`子优先级级数`：利用`GICC_BPR`寄存器的`低3位`（高29位未使用），
    设置如下（XX优先级域是指某个寄存器的某些位？？N级XX优先级表示有`2^N`个XX优先级）：
        <table border="1">
            <tr> <th>寄存器值</th> <th>抢占优先级域</th> <th>子优先级域</th> <th>说明</th> </tr>
            <tr> <td>0</td> <td>[7:1]</td> <td>[0]</td> <td>7级抢占优先级，1级子优先级</td> </tr>
            <tr> <td>1</td> <td>[7:2]</td> <td>[1:0]</td> <td>6级抢占优先级，2级子优先级</td> </tr>
            <tr> <td><strong>2</strong></td> <td>[7:3]</td> <td>[2:0]</td> <td><strong>5级抢占优先级，3级子优先级</strong></td> </tr>
            <tr> <td>3</td> <td>[7:4]</td> <td>[3:0]</td> <td>4级抢占优先级，4级子优先级</td> </tr>
            <tr> <td>4</td> <td>[7:5]</td> <td>[4:0]</td> <td>3级抢占优先级，5级子优先级</td> </tr>
            <tr> <td>5</td> <td>[7:6]</td> <td>[5:0]</td> <td>2级抢占优先级，6级子优先级</td> </tr>
            <tr> <td>6</td> <td>[7:7]</td> <td>[6:0]</td> <td>1级抢占优先级，7级子优先级</td> </tr>
            <tr> <td>7</td> <td>无</td> <td>[7:0]</td> <td>0级抢占优先级，8级子优先级</td> </tr>
        </table>
    * 设置`优先级值`：值范围为`0`～`31`，且值越小，优先级越高，
    可通过`GICD_IPRIORITYR[N]`寄存器进行设置，这是一系列寄存器，共有`512`个，
    可使用中断标识号（ID）作为索引号（即`N`），并且值要进行相应的`左移`，
    例如在`i.MX6ULL`将`40号`中断的优先级设为`5`，则应写成：
    `GICD_IPRIORITYR[40] = 5 << 3;`

## 18、电源管理

待补充。

## 19、倍频、分频、时钟、定时器

* `时钟信号`是一切操作的动力源和速率度量。

* `倍频`技术**可简化**`时钟源`，例如仅使用一个`24MHz`的`晶振`并配合`倍频`电路，
便可输出多路不同频率的时钟信号。`倍频`逻辑可使用`PLL`（`锁相环`）来实现，
属于`提频`。

* `分频`技术则用于产生各种外设所需的频率，属于`降频`，可通过写`寄存器`来细化配置。

* 时钟层级递进逻辑为：`24MHz`晶振 -> `PLL`倍频 -> `1`至`2`级分频 -> 最终的外设频率。

* 时钟树：
    * `Clock Switcher`：对应上一点的`倍频`逻辑。
    * `Clock Root Generator`：对应上一点的`分频`逻辑。
    * `System Clocks`：对应上一点的`外设频率`。

* 时钟相关寄存器：待补充。

* `EPIT`（共`2`路）：`Enhanced Periodic Interrupt Timer`，增强型周期性中断定时器，
可提供精准的、重复性（不支持仅触发一次）的定时中断。
    * `组成`：
        * 时钟源输入（`3`个）：`ipg_clk`、`ipg_clk_32k`、`ipg_clk_highfreq`。
        * 分频电路：`4096`种分频，分别对应`0`～`4095`。
        * 计数器：即`EPIT_CNR`寄存器，向下计数，其初始值源于`EPIT_LR`加载寄存器，
        或从`0xFFFFFFFF`重新开始，视不同模式而定，而模式可由`EPITx_CR`（x = 1, 2）指定。
        * 比较器：即`EPIT_CMPR`寄存器，值相等时会产生一个`引脚输出`信号和一个`中断`信号。
    * 使用`步骤`：
        1. 选择`时钟源`：设置`EPITx_CR`的`CLKSRC`位；
        2. 设置`分频值`：设置`EPITx_CR`的`PRESCALAR`位；
        3. 设置`工作模式`：设置`EPITx_CR`的`RLD`位；
        4. 设置`计数初值来源`：设置`EPITx_CR`的`ENMOD`位；
        5. 打开`比较中断`：设置`EPITx_CR`的`OCIEN`位；
        6. 设置`加载器`和`比较器`：分别是`EPIT_LR`和`EPIT_CMPR`；
        7. 在`GIC`打开对应该路`EPIT`的中断，并设置中断`优先级`（可选），
        详见[中断机制](#中断机制)章节；
        8. 编写中断服务函数；
        9. 打开`EPIT`中断：设置`EPITx_CR`的`EN`位。
    * `寄存器`详情：见[参考资料3](#ref_3)“24.6 EPIT Memory Map/Register Definition”小节。

* `GPT`（共`2`路）：`General Purpose Timer`，通用型定时器，比前述`EPIT`更通用。
    * `组成`：
        * 时钟源输入（`5`个）：`ipg_clk_24M`、`GPT_CLK`（外部时钟）、
        `ipg_clk`、`ipg_clk_32k`和`ipg_clk_highfreq`。
        * 分频电路：`4096`种分频，分别对应`0`～`4095`。
        * 计数器：即`GPTx_CNT`（x = 1, 2，下同）寄存器，向上计数。
        * 输入捕获器（`2`路）：待补充。
        * 输出寄存器（`3`路）：待补充。
        * 比较器（`3`路）：当某一路的`输出寄存器`和`计数器`的值比较相等时，
        会输出`中断信号`，各路中断独立输出。
    * 使用`步骤`：
        1. `复位`定时器：对`GPTx_CR`的`SWR`位写`1`（复位后此位会自动清零）；
        2. 选择`时钟源`：设置`GPTx_CR`的`CLKSRC`位；
        3. 设置`工作模式`：设置`GPTx_CR`的`FRR`位；
        4. 设置`分频值`：设置`GPTx_PR`的`PRESCALAR`位；
        5. 设置`比较器`：设置`GPTx_OCRy`（y = 1, 2, 3）；
        6. 打开`GPTx`中断：设置`GPTx_CR`的`EN`位；
        7. 编写中断服务函数。
    * `寄存器`详情：见[参考资料3](#ref_3)“30.6 GPT Memory Map/Register Definition”小节。

## 20、有线以太网

* 为何要拆分成`MAC`和`PHY`两部分：
    * `MAC`对应链路层，负责通信协议的控制与处理；
    `PHY`对应物理层，通过`数-模`双向转换进行数据收发。
    * `MAC`芯片是全数字器件，而`PHY`还含有大量的模拟器件，
    后者对集成有很大挑战，所以主流方案都是将`MAC`集成到CPU内部，
    而`PHY`作为外部芯片，既降低了成本，又可为`MAC`设置`DMA`大大加速数据包处理。
    * 两者的数据接口是`MII`或`RMII`，控制接口是`MDIO`（类似于`I2C`）。
    * `PHY`对外则会先连接`变压器`、再连`RJ45`接口（即网线接口），
    部分`RJ45`可能会内置`变压器`。
    * 由于`i.MX6ULL`已集成`MAC`（详见后面），而在实际做产品时可能会选用不同的`PHY`芯片，
    所以一般移植的重点是`PHY`的参数调整。由于`PHY`芯片寄存器地址线宽是`5`位，
    亦即支持最多`32`个寄存器，并且`IEEE`规定了前`16`个基础寄存器的功能，
    所以无论采用哪个厂家的`PHY`芯片，单靠前`16`个寄存器就足以驱动起来，
    亦即可以采用通用驱动代码，但要使用芯片的专用特性，就需要原厂驱动了。
    关于`PHY`基础寄存器的说明，详见`IEEE Std 802.3™-2018`文档`SECTION2`第`22.2.4 Management functions`小节。

* `MII`与`RMII`的区别：
    * `MII`即`Media Independent Interface`，**介质独立接口**，
    是最初的`IEEE-802.3`以太网标准接口，特点是收发时钟分开，
    且收发都是`4`条数据线，最大缺点也是线太多，达到`16`条，
    所以已越来越少用。
    * `RMII`即`Reduced MII`，属于**精简型**的`MII`接口，
    特点是收发共用同一时钟，且收发数据线都缩减到`2`条，
    线的数量只有`7`条，但需要**翻倍的时钟频率**以保证传输速率。
    * 此外还有`GMII`、`RGMII`等接口，都是在前两者基础上发展起来的，
    不再赘述。

* 网络分层：
    <table style="text-align: center">
        <tr>
            <th>OSI层级</th>
            <th>OSI模型</th>
            <th>TCP/IP模型</th>
            <th>TCP/IP层级</th>
        </tr>
        <tr>
            <td>7</td>
            <td>应用层</td>
            <td rowspan="3">应用层</td>
            <td rowspan="3">4</td>
        </tr>
        <tr>
            <td>6</td>
            <td>表示层</td>
        </tr>
        <tr>
            <td>5</td>
            <td>会话层</td>
        </tr>
        <tr>
            <td>4</td>
            <td colspan="2">传输层</td>
            <td>3</td>
        </tr>
        <tr>
            <td>3</td>
            <td colspan="2">网络层</td>
            <td>2</td>
        </tr>
        <tr>
            <td>2</td>
            <td>链路层</td>
            <td rowspan="2">链路层</td>
            <td rowspan="2">1</td>
        </tr>
        <tr>
            <td>1</td>
            <td>物理层</td>
        </tr>
    </table>

* `i.MX6ULL`的`10/100-Mbps Ethernet MAC`（`ENET`）功能特性：
    * 实现`三`层网络**硬件级**加速，可加速常见协议（IP、TCP、UDP、ICMP等）数据包的处理。
    通过硬件加速模块来执行关键功能（perform critical functions），
    比在软件层面用大帧头来实现（with large software overhead）相同的功能要好很多。
    * 带`RMON`（`Remote Network Monitoring`）计数功能。
    * 内置`FIFO`缓冲区，减少接收丢包。
    * 高级电源管理，可用于`魔术包`（`magic packet`）检测和`掉电模式`（`power-down` mode）。
    * 内置专用`DMA`，可优化`ENET`核心与`SoC`之间的数据传输，
    还支持`IEEE 1588`标准的`增强型缓冲描述符`（`enhanced buffer descriptor`），
    而且`IEEE 1588`还提供`工业自动化应用`场景下的`分布式控制节点`所需的`精确时钟同步`机制。
    * 自动`流（量）控（制）`（`Flow Control`），不需应用层干预。
    * 可通过`MII`（2.5/25MHz）、`MII-Lite`（同前）或`RMII`（50MHz）接口与各种`PHY`无缝衔接。
    * 内含单播和多播地址过滤，减少高层应用的负担。
    * 能合并`MAC`模块产生的中断，降低CPU负担。
    * 支持`IPv4`和`IPv6`，并且有自动的IP、TCP、UDP、ICMP的校验码生成及检验、数据包和错误统计、
    可配置的错误帧丢弃/收发字节序转换/32位字对齐/剥除填充内容……之类的特性。
    * 其余细节及寄存器详情见[参考资料3](#ref_3)“Chapter 22 10/100-Mbps Ethernet MAC (ENET)”。

## 21、常见的基础通信接口

### 21.1 一些重要概念

* `同步`与`异步`：区别在于是否需要`时钟信号`的`同步机制`：
    * `同步`通信：数据传输的`时序`是由同一个`时钟信号控制`的。收发双方均需在每个时钟周期内，
    按特定顺序收发每一个数据位，所以通信协议不需要`定界符`，但`数据速率`必须与`时钟频率`匹配，
    即`时钟频率`决定`数据速率`。
    * `异步`通信：数据传输不需要时钟信号，而是使用`定界符`（起始位、停止位、
    校验位等）来确定传输情况。
    * **注意**：速率快慢与同步异步无绝对关系，而与`物理电气特性`和`软件协议设计`有关。
    通信距离亦然。

* `单工`与`双工`：
    * 两个设备之间仅支持`单向`数据传输，就是`单工`。
    * 支持`双向`数据传输，就是`双工`，又可进一步分为：
        * `全`双工：`两个方向`的数据传输可`同时进行`。
        * `半`双工：不可同时进行。
        * **注意**：接收与发送**共**用一条线的，**肯定**是`半`双工；
        收与发**各**用一条线的，**基本都是**`全`双工（除非硬件设计和软件协议不支持）。

* `比特率`与`波特（率）`：
    * `比特率`：`Bit Rate`，每秒传输的`二进制位`数，符号是`b/s`或`bps`。
    * `波特（率）`：`Baud (Rate)`，每秒传输的`码元符号`数，符号是`Bd`。
    * 两者关系：一个`码元符号`可由多个`二进制位`（仅指数据位，不含起始、停止位）组成，
    所以两者的关系的表示公式为：`比特率` = `波特率` * `数据位数`。
    * **注1**：通常在`时钟速率`固定的情况下，`比特率`也固定，但`波特率`却要看`编码方式`，
    编码方式决定一个`码元符号`是由几个`比特位`组成的。有的计算标准会考虑起始、停止位；
    还有的标准使用的是`调制状态`占用的`二进制位数`，所以计算公式会使用`对数`，
    即：`比特率` = `波特率` * `log2(M)`，其中`M`是`调制相位数`，对于串口来说，
    电平变化只有`0`和`1`，属`两相调制`，即`M`=`2`，故`比特率`=`波特率`。
    * **注2**：`波特率`一般只用于`串口`（`UART`），其他情况基本上都用`比特率`。
    * **注3**：`波特`已包含`速率`的意思，其实不必再加一个`率`。

### 21.2 串口（UART）

* 全称详见[缩略词表格](#缩略词)。

* 引脚相对于`并（行）（接）口`大大减少，方便布线，但通信速率亦会大大降低。

* 与后面的`I2C`、`SPI`等接口相比，通信距离较远，但也需配合电平转换。

* **最小配置**的**时序**是**异步**通信里面**最简单**的。

* 引脚（全双工最小配置）：
    * `GND`：接地。
    * `RXD`：接收。
    * `TXD`：发送。

* 协议时序：
    1. `空闲`状态：线路处于逻辑`1`（即**高**电平）的状态。
    2. `起始`位：逻辑`0`，即**低**电平。
    3. `数据`位：
        * 位数：可选择`5`～`9`位，由于一个字节是8位，所以通常选择8位。
        * 顺序：先传`低`位，后传`高`位，与**小端**字节序**相似**。
    4. `奇偶校验`位：对数据中`1`的位数进行校验，且校验功能可关闭。
    5. `停止`位：逻辑`1`，位宽可选择`1`、`1.5`或`2`，一般选`1`。

* 接口电平：
    1. `TTL`：`低`电平（`0`V）表示逻辑`0`，`高`电平（`5`V或`3.3`V）表示逻辑`1`。
    常用于设备内部的数据传输，而不像以下两个的跨设备、长距离场景。
    2. `RS-232`（`Recommended Standard 232`）：即`COM口`（`DB9`接口，或`D-Sub 9`，
    但通常只使用其中`3`条线，其余线仅以前用到，引脚与`TTL`兼容，但电平需要转换），
    `+3`～`+15`V表示逻辑`0`，`-3`～`-15`V表示逻辑`1`。速率较低（异步`20 Kbps`），
    抗干扰能力较差，实际中最大通信距离为`50`米左右（视通信速率而定，速率越大，
    距离越短，下同）。连接方式**只能一对一**。
    3. `RS-485`：采用`差分`线（不必用`GND`，抗干扰能力强，通信距离更长，可达`3000`米，
    速度最高可达`10 Mbps`，但属于`半`双工），`+2`～`+6`V表示逻辑`1`，`-2`～`-6`V表示逻辑`0`，
    `0`V表示`空闲`，电压比`RS-232`降低了，更不易损坏芯片，且与`TTL`电平兼容，
    可方便与其相连。连接方式**允许一对多**（但通信时序不是？），最多可达`128`个收发器。

* `寄存器简介`：
    * 分析待补充。
    * 寄存器详情见[参考资料3](#ref_3)“55.15 UART Memory Map/Register Definition”小节。

### 21.3 I2C

* 全称详见[缩略词表格](#缩略词)。

* 引脚数**最少**！

* 引脚（均需上拉电阻）：
    * `SCL`：`Serial Clock`，时钟。
    * `SDA`：`Serial Data`，数据。

* 协议时序（基于地址；应答式）：
    1. 数据的采样全发生在`SCL`为`1`（高电平）期间；以下如无说明均是`主机`视角。
    2. `起始`位：`SDA`输出`下降沿`。
    3. `停止`位：`SDA`输出`上升沿`。
    4. `数据`位：直接采样`SDA`的`电平`值，注意要保持电平稳定，
    数据要变化就只能发生`SCL`为`0`期间。
        * 位数：不固定，由硬件决定，可以在`1`～`32`范围内选择，一般设为`8`。
        * 顺序：先传`高`位，后传`低`位，与**大端**字节序**相似**、与`UART`相反。
    5. `应答`位：紧跟`数据`之`后`的**一个时钟**周期，供`从机`使用，将`SCA`**拉低**即可，
    此时`主机`是处于`输入`状态，读到低电平则表示通信成功而继续下一步，否则进行错误处理。
    6. `单`字节`写`时序：
        1. 发送从机`设备地址`：`起始`位；`7`位地址；（最低位）`1`位`读写位`，
        此处为`0`表示`写`；等待从机`应答`信号。
        2. 发送从机`寄存器地址`：`起始`位；`8`位地址；等待从机`应答`信号。
        3. 发送`寄存器数据`：`8`位数据；等待从机`应答`信号。
        4. 发送`停止`位。
    7. `单`字节`读`时序（比`写`时序多一步）：
        1. 发送从机`设备地址`：与`写`时序一样。
        2. 发送从机`寄存器地址`：与`写`时序一样。
        3. 再发一次从机`设备地址`：除了`读写位`为`1`表示接下来要读数据之外，
        其余与第一步相同。
        4. 读取`寄存器数据`：`8`位数据；且向从机发送`1`位`NO ACK`信号，
        表示不需要从机应答。
        5. 发送`停止`位。
    8. `多`字节`写`时序：
        * 除了在第三步连续发送多个字节的数据（包括每个字节均要等待应答）之外，
        其余与`单`字节`写`时序相同，而且要求主机和从机均支持这种模式。
    9. `多`字节`读`时序：
        * 除了在第四步连续接收多个字节的数据、非末尾字节等待从机应答、
        末尾字节才向从机发送`NO ACK`之外，其余与`单`字节`读`时序相同，
        而且要求主机和从机均支持这种模式。

* `寄存器简介`：
    * 分析待补充。
    * 寄存器详情见[参考资料3](#ref_3)“31.7 I2C Memory Map/Register Definition”小节。

### 21.4 SPI

* 全称详见[缩略词表格](#缩略词)。

* 速率比`I2C`更快，**时序**也是**同步**通信里面**最简单**的（除了按时钟升降沿采样之外，
几乎无额外要求）。

* 引脚：
    * `CS/SS`：`Slave Select / Chip Select`，片选，拉低即选中某从机，
    在多从机场景中，主机可以有多个片选引脚。菊花链模式是另一种多从机连接方式，
    即所有从机的片选引脚均连接到主机的单一片选引脚，而从机之间的数据引脚则串联。
    * `SCK`：`Serial Clock`，时钟。
    * `MOSI/SDO`：`Master Out Slave In / Serial Data Output`，主出从入。
    * `MISO/SDI`：`Master In Slave Out / Serial Data Input`，主入从出。

* 协议时序（无地址；有片选；时钟跳变沿读写数据）：
    1. **拉低片选**信号，选中要通信的从机。
    2. 在相应的时钟**跳变沿**（第`1`还是第`2`个上升或下降沿，见后面配置），
    可同时通过`MOSI`**发送**数据及通过`MISO`**接收**数据。
    收发顺序与`I2C`相同，即先传`高`位，后传`低`位。
        * `CPOL`：`Clock Polarity`，时钟`极性`，决定`SCK`**空闲时**的极性：
        `0`为`低`电平，`1`为`高`电平。
        * `CPHA`：`Clock Phase`，时钟`相位`，决定`数据线`的**采样时机**：
        `0`是在第`1`个跳变沿采样，`1`则在第`2`个跳变沿采样。
        * 至于采样时是上升沿还是下降沿，则要结合以上两个参数来判断。
        例如两个均为`0`，则意味着总线从空闲（低电平）转为就绪状态（高电平）时，
        第一个时钟跳变沿是上升沿，此时即进行数据采样，其余情况可类推。

* `寄存器简介`：
    * 分析待补充。
    * 寄存器详情见[参考资料3](#ref_3)“20.7 ECSPI Memory Map/Register Definition”小节。

### 21.x 其余

待补充。

### 21.y 各通信接口对比

名称 | 引脚 | 连接方式 | 速率 | 协议特色 | 典型用途
-- | -- | -- | -- | -- | --
UART | GND<br>RXD<br>TXD | 一对一 | 典型：115200 Bd | 最简单的异步通信 | 目标板调试、低速率工控设备通讯、蓝牙、GPS、GPRS等
I2C | SCL<br>SDA | 一主多从 | 标准：100 Kb/s<br>高速：400 Kb/s | 基于地址；<br>应答式 | 温度传感器、陀螺仪、加速度计、触摸屏等
SPI | CS/SS<br>SCK<br>MOSI/SDO<br>MISO/SDI | 一主多从 | 一般可达到：x Mb/s | 无地址，有片选；<br>最简单的同步通信 | 传感器、储存器、实时时钟、数模转换等

## 22、其余模块的功能简介、寄存器定义、及内存映射

### 22.1 IO复用<a id="IO复用"></a>

* 所谓的`IO复用`（IO Multiplexing），是指一个`引脚`（`Pin`）可以被多个功能模块所用
（但在给定的任一时刻只能配置成一种功能状态），
在对应的寄存器配置方面有多个`信号选项`（`Signal Option`）可供选择。

* 设置`引脚`与`信号`之间的映射关系的`复用器`（`Multiplexer`）叫`IOMUX`。

* 各个复用选项的详情见[参考资料3](#ref_3)的“4.1.1 Muxing Options”小节，
或具体模块的“**External Signals**”小节。现仅将其分属的模块列举如下：

    代号 | 含义
    -- | --
    PLATFORM | 平台
    CCM | Clock Controller Module，时钟控制器模块
    CSI | CMOS Sensor Interface，CMOS传感器接口
    ECSPI 1~4 | Enhanced Configurable Serial Peripheral Interface，增强型可配置串行外设接口，共4路
    EIM | External Interface Module，外部接口模块
    ENET 1~2 | Ethernet，以太网，共2路
    EPIT 1~2 | Enhanced Periodic Interrupt Timer，增强型周期性中断定时器，共2路
    EPDC | Electrophoretic Display Controller，电泳显示器控制器
    FlexCAN 1~2 | Flexible Controller Area Network，弹性控制器局域网，共2路
    GPIO 1~5 | General Purpose Input/Output，通用型输入输出，共5路（每路引脚数不一样）
    GPT 1~2 | General Purpose Timer，通用型定时器，共2路
    I2C 1~4 | Inter-Integrated Circuit，内部（还是跨域？？）集成电路，共4路
    KPP | Keypad Port，键盘端口
    LCDIF | Liquid Crystal Display Interface，液晶显示器接口
    MMDC | Multi Mode DDR Controller，多模式DDR内存控制器
    MQS | Medium Quality Sound，中等质量音响
    NAND | 与非型闪存
    PWM 1~8 | Pulse Width Modulation，脉冲宽度调制，共8路
    QSPI | Quad Serial Peripheral Interface，四通路（？？）串行外设接口
    SAI 1~3 | Synchronous Audio Interface，同步音频接口
    SDMA | Smart Direct Memory Access Controller，智能型直接内存访问控制器
    SJC | System JTAG Controller，系统JTAG控制器
    SNVS | Secure Non-Volatile Storage，安全型非易失性储存器
    SRC | System Reset Controller，系统重置控制器
    SPDIF | Sony/Philips Digital Interface，索尼/菲利普数字接口
    UART 1~8 | Universal Asynchronous Receiver/Transmitter，通用异步收发器（即串口），共8路
    USB | Universal Serial Bus，通用串行总线
    USDHC 1~2 | Ultra Secured Digital Host Controller，超安全主机控制器（用于读写SD/MMC储存卡），共2路
    WDOG 1~3 | Watchdog，看门狗，共3路
    XTALOSC | Crystal Oscillator，晶（体）振（荡器）

* 在实际操作中，首先要通过`IO复用控制器`（`IOMUX Controller`）来设置引脚复用，
该控制器称作`IOMUXC`，有以下寄存器：
    * `IOMUXC_SW_MUX_CTL_PAD_<PAD NAME>`或`IOMUXC_SW_MUX_CTL_GRP_<GROUP NAME>`：
    （1）配置一个焊盘（`Pad`）或焊盘组的`复用选项`（`Multiplexing Option`，
    在寄存器中表现为`ALT`字段）；（2）激活相应的`输入路径强制`（`Forcing of Input Path`，
    在寄存器中表现为`SION`字段，即`Software Input ON`之意）特性。
    * `IOMUXC_SW_PAD_CTL_PAD_<PAD NAME>`或`IOMUXC_SW_PAD_CTL_GRP_<GROUP NAME>`：
    对一个焊盘或焊盘组进行特定设置，即配置电气特性，例如上下拉电阻、是否开漏输出等。
    * `IOMUXC_GPR_GPR0`～`IOMUXC_GPR_GPR13`：通用寄存器，根据`SoC`实际需求而定。

* 以上涉及的具体寄存器的功能定义及内存映射详见[参考资料3](#ref_3)的以下章节：
    * `32.4 IOMUXC GPR Memory Map/Register Definition`
    * `32.5 IOMUXC SNVS Memory Map/Register Definition`
    * `32.6 IOMUXC Memory Map/Register Definition`

* 最后，配置`具体模块的寄存器`（寄存器明细项及其内存映射详见特定章节，此处列举不了），
进行实际的数据收发。

### 22.2 GPIO

* 相关寄存器：
    * `GPIO_DR`：Data Register，数据寄存器
    * `GPIO_GDIR`：Direction，方向寄存器
    * `GPIO_PSR`：Pad Sample Register，焊盘采样寄存器
    * `GPIO_ICR1`、`GPIO_ICR2`：Interrupt Control Register，中断控制寄存器
    * `GPIO_EDGE_SEL`：Edge Select，边缘选择触发器
    * `GPIO_IMR`：Interrupt Mask，中断掩码寄存器
    * `GPIO_ISR`：Interrupt Status Register，中断状态寄存器

* 编程指导：详见[参考资料3](#ref_3)“28.4.3 GPIO Programming”。

* 寄存器定义及内存映射：详见[参考资料3](#ref_3)“28.5 GPIO Memory Map/Register Definition”。

### 其他

待补充。

## 23、蛋疼的缩略词（需结合语境）<a id="缩略词"></a>

英文缩写 | 英文全称 | 中文翻译（仅供参考）
-- | -- | --
ABI | Application Binary Interface | 应用程序二进制接口
ALU | Arithmetic Logical Unit | 算术逻辑单元
ARM | Advanced RISC Machine | 高级RISC机器
BSS | Block Started by Symbol | 以符号作为开头的块区域
CAN | Controller Area Network | 控制器局域网
CISC | Complex Instruction Set Computer | 复杂指令集计算机
CMOS | Complementary Metal Oxide Semiconductor | 互补金属氧化物半导体
CPU | Central Processing Unit | 中央处理器
CSI | CMOS Sensor Interface | CMOS传感器接口
DDR | Double Data Rate (Synchronous Dynamic Random Access Memory) | 双倍数据速率（同步动态随机访问储存器）
DMA | Direct Memory Access | 直接内存访问
DRAM | Dynamic Random Access Memory | 动态随机访问储存器
EEPROM | Electrically Erasable Programmable Read-Only Memory | 可电气擦除的可编程只读储存器
FIFO | First In First Out | 先入先出
GIC | Generic Interrupt Controller | 通用中断控制器
GPIO | General Purpose Input/Output | 通用型输入/输出
I2C / IIC | Inter-Integrated Circuit | 内部（还是跨域？？）集成电路
JTAG | Joint Test Action Group | 联合测试行动组，一种调试接口标准
LCD | Liquid Crystal Display | 液晶显示器
MMU | Memory Management Unit | 内存管理单元
OSI | Open System Interconnection | 开放式系统互联
PLL | Phase Locked Loop | 锁相环
PPI | Private Peripheral Interrupt | 私有外设中断
PWM | Pulse Width Modulation | 脉冲宽度调制
RAM | Random Access Memory | 随机访问储存器
RISC | Reduced Instruction Set Computer | 精简指令集计算机
ROM | Read-Only Memory | 只读储存器
SDHC | Secure Digital High Capacity | 高容量安全数字（规范/标准），即新一代的SD卡规范/标准
SGI | Software-Generated Interrupt | （由）软件产生的中断
SIMD | Single Instruction Multiple Data | 单指令多数据
SNVS | Secure Non-Volatile Storage | 安全型非易失性储存器
SoC | System on Chip | 片上系统
SPI | Serial Peripheral Interface / Shared Peripheral Interrupt | 串行外（围）设（备）接口 / 共享型外设中断
SDRAM | Synchronous Dynamic Random Access Memory | 同步动态随机访问储存器
SRAM | Static  Random Access Memory | 静态随机访问储存器
TTL | Transistor-Transistor Logic | 晶体管-晶体管逻辑
UART | Universal Asynchronous Receiver/Transmitter | 通用异步收发器，通常简称为串口
USB | Universal Serial Bus | 通用串行总线

`2024-11-25`注：更多缩略词可查阅<a href="万恶的字母缩写.html" target="_blank">这篇文章</a>，此表不再更新。

## 24、参考材料

1. <a id="ref_1" href="references/ARM Architecture Reference Manual ARMv7-A and ARMv7-R edition.pdf" target="_blank">ARM Architecture Reference Manual ARMv7-A and ARMv7-R edition</a>
（宜作为字典，按需查询）

2. <a id="ref_2" href="references/ARM Cortex-A Series Programmer's Guide V4.0.pdf" target="_blank">ARM Cortex-A Series Programmer's Guide V4.0</a>
（建议通读一遍）

4. <a id="ref_3" href="references/i.MX 6ULL Applications Processor Reference Manual.pdf" target="_blank">i.MX 6ULL Applications Processor Reference Manual</a>
（宜作为字典，按需查询）

## 25、温馨提示<a id="温馨提示"></a>

**书山有路找捷径，学海无涯抓重点！**

![秘笈](assets/images/bei3kap1.png)

