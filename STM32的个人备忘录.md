<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />
<!-- <font size=5> -->

# STM32的个人备忘录

## 创建工程的的必要步骤（基于`STM32CubeIDE`）

* 选择`驱动类型`：打开`*.ioc`，依次点击`Project Manager` -> `Advanced Settings`，根据界面的提示，
对相应项选择`HAL`或`LL`即可。**注意**：在设置并保存之后，若没有自动弹出询问是否生成代码的对话框，
则需要手动对`*.ioc`右击，然后点击`Generate Code`来触发生成动作，后续需要修改`*.ioc`的步骤亦同理。

* 配置`时钟`：打开`*.ioc`，然后：
    * 切换到`Pinout & Configuration`页面，先（在`Pinout view`标签页）将对应的时钟引脚设置成接外部晶振的状态，
    再依次点击`System view` -> `RCC`，找到`RCC Mode and Configuration`，为`High Speed Clock (HSE)`选择`Crystal/Ceramic Resonator`。
    * 切换到`Clock Configuration`页面，根据界面的提示，设置好时钟路径、倍频参数、分频参数，
    即可实时看到各路时钟的最终频率。

* 创建`用户`代码`目录`：
    * 对工程右击，依次选择`New` -> `Source Folder`，目录名填`User`，最后点击`Finish`按钮。
    * 右击该目录，选择`Add/remove include path...`，将其加入编译器可搜索的头文件目录列表。

* 创建`业务入口`代码：在`User`目录下创建`biz_entry.h`和`biz_entry.c`，并分别编写`void do_biz(void)`的声明和函数体，
然后在`Core/Src/main.c`文件主函数的死循环里进行调用。

* 创建`版本头文件`：在`User`目录下创建`versions.h`，内容详见[懒编程秘笈](https://github.com/FooFooDamon/lazy_coding_skills)
的`vim/templates/h.tpl.list/versions.tpl`。

* 创建`纯内存烧录`的`链接脚本`：
    * 将`STM32*_FLASH.ld`复制一份，并重命名为`STM32*_RAM.ld`，然后将所有的`>FLASH`和`>RAM AT> FLASH`均改成`>RAM`。
    * 对工程右击，依次选择`Properties` -> `C/C++ Build` -> `Settings` -> `Tool Settings` -> `MCU GCC Linker` -> `General`，
    找到`Linker Script (-T)`并将方框内的`STM32*_FLASH.ld`改成`STM32*_RAM.ld`，最后点击`Apply and Close`按钮。
    **注意**：仅当需要纯内存烧录时才执行此步骤，当用于生产环境时，还是要恢复成`STM32*_FLASH.ld`。

* 创建`makefile.init`：
    * 在工程根目录下创建，内容详见[懒编程秘笈](https://github.com/FooFooDamon/lazy_coding_skills)
    的`vim/templates/mk.tpl.list/stm32_init.tpl`。
    * 对工程右击，依次选择`Properties` -> `C/C++ Build` -> `Settings` -> `Tool Settings` -> `MCU GCC Compiler` -> `Include paths`，
    找到`Include files (-include)`并新建一项`"generated.h"`（注意此处的文件名需要用英文双引号括起），
    最后点击`Apply and Close`按钮。

* 创建`扩展Makefile`：在工程根目录下创建，名为`Makefile`，内容详见[懒编程秘笈](https://github.com/FooFooDamon/lazy_coding_skills)
的`vim/templates/mk.tpl.list/stm32cubeide.tpl`。

## 串口编程的陷阱

待补充。

<!-- </font> -->
<a target="_self" href="#" style="position: fixed; bottom: 20px; right: 20px; z-index: 1000;">返回顶部</a>

