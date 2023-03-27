<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# 极简嵌入式ARM知识点——基于NXP公司i.MX6ULL处理器

## 1、前言

* 标题已说明一切。

* 建议先看文末的[温馨提示](#温馨提示)。

## 2、`ARM`版本的特点

* `具体处理器`和`基础架构`分别有各自的版本号：
例如`ARM7TDMI`和`ARM920T`的架构版本分别为`v4`和`v4T`，`ARM946E-S`为`v5`，
`ARM1136J-S`为`v6`，等等。

* 从`v6`架构开始，处理器版本不再使用单一数字递增的命名方式，
而开始发布`Cortex-A`、`Cortex-R`、`Cortex-M`等系列，其中`A`表示`Application`、
`R`表示`Real-time`、`M`表示`Microcontroller`，应对不同场景。

* 上述每一系列再加上数字形成一个具体型号的处理器，
并且数字的大小不一定反映先后顺序和工艺性能，例如`A8`的数字比`A7`要大，
但`A7`在`A8`之后发布，且性能更强。

* 所以，要对比不同系列、不同型号处理器之间的性能特点，
只能从官方资料查证，不能简单以名称和数字进行推断。

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

* 总结得出以下关系：
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
    AL | Always，总是，默认值 | -

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

    代号 | 含义 | 作用
    -- | -- | --
    .bss | 以符号作为开头的块区域 | 存放未初始化或初始化为0的全局变量和静态变量，在程序执行前自动清零，故此段**不占储存器空间**，仅**标记**所含全部**变量占用的空间大小**。
    .data | 数据段 | 存放**已初始化**的**非0**全局变量和静态变量。
    .rodata | 只读数据段 | 存放只读数据。是否一定存在，以及存在的位置，均有待确认。
    .text | 正文段 | 存放程序代码，以及常量，故该区域通常设置为只读，但某些架构也允许写，即在运行期修改代码。

### 10.2 运行期加载到内存时所呈现的布局

* 除了前述的文件布局内容，还增加了以下内容：

    代号 | 含义 | 作用
    -- | -- | --
    heap | 堆 | 运行期动态分配的内存段，从低地址向高地址扩张。
    stack | 栈 | 存放非静态局部变量，从高地址向低地址扩张。
    mmap | 文件映射 | 处于堆栈之间，扩张方向与栈相同。

* 总的（虚拟）内存布局（以32位系统为例）如下：

    起始地址 | 分段
    -- | --
    0xFFFFFFFF | /
    0xC0000000 | kernel space (1GB)
    uncertain | stack
    uncertain | mmap
    uncertain | heap
    uncertain | .bss
    uncertain | .data
    uncertain | .text
    0x0 | /

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

## 17、中断服务

待补充。

## 18、电源管理

待补充。

## 19、分频、时钟、定时器

待补充。

## 20、通信接口

待补充。

## 21、其余功能模块简介，以及相应的地址空间

待补充。

## 22、蛋疼的缩略词（需结合语境）<a id="缩略词"></a>

英文缩写 | 英文全称 | 中文翻译（仅供参考）
-- | -- | --
ABI | Application Binary Interface | 应用程序二进制接口
ALU | Arithmetic Logical Unit | 算术逻辑单元
ARM | Advanced RISC Machine | 高级RISC机器
BSS | Block Started by Symbol | 以符号作为开头的块区域
CAN | Controller Area Network | 控制器局域网
CISC | Complex Instruction Set Computer | 复杂指令集计算机
CPU | Central Processing Unit | 中央处理器
DDR | Double Data Rate (Synchronous Dynamic Random Access Memory) | 双倍数据速率（同步动态随机访问储存器）
DRAM | Dynamic Random Access Memory | 动态随机访问储存器
EEPROM | Electrically Erasable Programmable Read-Only Memory | 可电气擦除的可编程只读储存器
GIC | Generic Interrupt Controller | 通用中断控制器
I2C / IIC | Inter-Integrated Circuit | 内部（还是跨域？？）集成电路
MMU | Memory Management Unit | 内存管理单元
RAM | Random Access Memory | 随机访问储存器
RISC | Reduced Instruction Set Computer | 精简指令集计算机
ROM | Read-Only Memory | 只读储存器
PPI | Private Peripheral Interrupt | 私有外设中断
PWM | Pulse Width Modulation | 脉冲宽度调制
SGI | Software-Generated Interrupt | （由）软件产生的中断
SIMD | Single Instruction Multiple Data | 单指令多数据
SPI | Serial Peripheral Interface / Shared Peripheral Interrupt | 串行外（围）设（备）接口 / 共享型外设中断
SDRAM | Synchronous Dynamic Random Access Memory | 同步动态随机访问储存器
SRAM | Static  Random Access Memory | 静态随机访问储存器
UART | Universal Asynchronous Receiver/Transmitter | 通用异步收发器，通常简称为串口
USB | Universal Serial Bus | 通用串行总线

## 23、参考材料

1. <a id="ref_1" href="references/ARM Architecture Reference Manual ARMv7-A and ARMv7-R edition.pdf">ARM Architecture Reference Manual ARMv7-A and ARMv7-R edition</a>
（宜作为字典，按需查询）

2. <a id="ref_2" href="references/ARM Cortex-A Series Programmer's Guide V4.0.pdf">ARM Cortex-A Series Programmer's Guide V4.0</a>
（建议通读一遍）

4. <a id="ref_3" href="references/i.MX 6ULL Applications Processor Reference Manual.pdf">i.MX 6ULL Applications Processor Reference Manual</a>
（宜作为字典，按需查询）

## 24、温馨提示<a id="温馨提示"></a>

**书山有路找捷径，学海无涯抓重点！**

![秘笈](bei3kap1.png)

