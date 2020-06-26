<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# 懒人VIM技巧

## 前言

本文适用于符合以下一点或以上而导致无固定、称手的IDE可长期使用的人士：

1. 经常**跨平台**（Linux、Windows、Mac）工作，且往往需要在**字符终端**下编程

2. 经常在**不同电脑间切换**（个人电脑、公司电脑、远程服务器等）

3. **机子太烂**而不足以运行大型IDE（例如内存杀手`Eclipse`），
或对**IDE升级后界面大改**而抓狂（例如`KDevelop`从3升到4）

4. ***懒***，即使拥有了被誉为“编辑器之神”的VIM，仍不想敲太多命令、记太多快捷键、
做太多配置、写太多代码，照搬别人的配置又是一大坨的很碍眼，还不一定能用

以下内容为一家之言，跟个人口味和习惯有关，***仅供参考***！

`注1`：以下操作**以包管理器安装为主**，宿主系统为`Ubuntu`，其余系统的包管理器安装命令可自行了解。
源码安装方式虽然最通用，但比较麻烦，非懒人所为。

`注2`：不少插件适用于多种编程语言，但某项功能（例如补全、跳转等）可能只需其中一个插件，
也可能需要多个插件协作。限于个人精力及能力，不能一一说清，故在大范围下只能采用
“**所需插件取其并集，最终使用效果及适用语言取其交集**”的方式。目前（已测试过）适用的语言有：C/C++、Python。

## 入门级插件

* `Ctags`：代码跳转。需要生成一些中间文件作为查询的索引，且代码有修改时需要重新生成。
（2020/06/26备注：使用`YouCompleteMe`跳转更方便，且不必生成及更新中间文件。）

* `Cscope`：相当于增强版的`Ctags`，还可查询变量、函数、头文件等的赋值、调用、包含等位置或关系。
与`Ctags`类似，需要生成一些中间文件。

* `TagList`：显示源码缩略图。

* `WinManager`：窗口管理。

关于以上插件的安装和配置，这里不再重复造轮子，借用一个网友`吴垠`的博客
[《手把手教你把 Vim 改装成一个 IDE 编程环境 ( 图文 )》](http://blog.csdn.net/wooin/article/details/1858917)
放在此供有需要的人参考，并对原作者表示感谢！若该链接失效，可点击
<a href="references/手把手教你把Vim改装成一个IDE编程环境(图文) - 吴垠的专栏 - 博客频道 - CSDN.NET.pdf">这里</a>
查看备份文档。

## 安装及配置`YouCompleteMe`插件

### 快速安装

以下适用于`Ubuntu`系统，版本`14.04`以上：

````
sudo apt-get install vim-addon-manager 
sudo apt-get install vim-youcompleteme
vim-addons install youcompleteme
````

网上很多教程可能因为写得较早的原因，安装过程都比较繁琐。此处则只需几条命令，
不需额外配置，装好后开箱即用。当然，需要更高级、更个性化的定制则除外。

为简化说明，后面将用`YCM`指代`YouCompleteMe`。

### 编写插件配置文件

YCM的配置文件是一个Python文件，所以它**既是数据，也是代码**，
换而言之，只要搞清其原理并恰当运用Python提供的机制，该配置文件可以达到既简洁又灵活的效果。
以下是一个与网上多数文章不同的配置方案。

#### 标准配置

````
import os

flags = [
    "-Wall"
    , "-std=c++11"
    , "-x", "c++"
]

for inc_dir in os.popen("echo | $(g++ --print-prog-name=cc1plus) -v 2>&1 | grep '/usr/.*include' | grep -v '^ignoring' | sed 's/^[ ]*//g'").read().split("\n"):
    if "" != inc_dir:
        flags.append("-I")
        flags.append(inc_dir)

SOURCE_EXTENSIONS = [ ".c", ".C", ".cc", ".cpp", ".cxx", ".c++" ]

def FlagsForFile(filename, **kwargs):
    return { "flags": flags, "do_cache": True }
````

这是一个适用于C/C++且可动态适配不同版本操作系统及编译器（gcc或g++）的配置，前提是编译器必须先行安装好，
否则YCM在读取配置时会报错或读到空白内容。`flags`列表（即数组）里的内容按照需要传递给相关语言编译器的参数填写即可，
对于C/C++编译器而言通常包括警告选项、头文件路径选项等，动态适配部分指的就是头文件路径
（不同版本操作系统，不同版本编译器，其头文件位置可能不同）。请自行将以上内容保存到一个文件，
例如：`/home/foo/etc/ycm_cpp_basic_conf.py`，然后执行`ln -s /home/foo/etc/ycm_cpp_basic_conf.py /usr/.ycm_extra_conf.py`。
注意这个链接操作是很有必要，因为若在目标系统头文件目录及其上级目录没有YCM配置文件，则在打开系统头文件时会有
“NoExtraConfDetected: No .ycm_extra_conf.py file detected”的警告且不能正确跳转，而直接复制配置文件又会有重复且后续修改麻烦，
所以使用链接操作。

该配置的后续更新详见[https://github.com/FooFooDamon/cpp_assistant]()项目的`conf/ycm_basic_conf.py`文件。

其余语言的标准配置可参考上述内容自行设计。

#### 具体项目的配置

在项目的根目录（针对简单项目只需要一个配置的场景）或者某个子模块的根目录（适用于复杂项目下不同子模块有不同配置的场景）
创建一个名为`.ycm_extra_conf.py`的文件，这样做的好处是不必手动在`~/.vimrc`
进行类似`let g:ycm_global_ycm_extra_conf='/Your/Project/.ycm_extra_conf.py'`的设置，尤其是有多个不同项目的情况下。

然后，写入以下内容（仅作示范）：

````
import os, sys
sys.path.append("/home/foo/etc/ycm_cpp_basic_conf.py")
from ycm_cpp_basic_conf import *
MODULE_ROOT = os.path.abspath(os.path.dirname(__file__)) # 子模块根目录
flags.extend([ "-I", os.path.join(MODULE_ROOT, "include") ]) # 头文件路径需自行确定，推荐使用相对路径
# 同上，可继续添加多个头文件路径，或更多的编译器选项
````

即可进行代码的补全及跳转。

### `~/.vimrc`设置

* 关闭YCM配置加载询问：`let g:ycm_confirm_extra_conf = 0`

* 跳转到定义或声明位置快捷键：`nnoremap <leader>d :YcmCompleter GoToDefinitionElseDeclaration<CR>`。
注：此处特意设成与Python插件`jedi-vim`（见后面）跳转定义处的快捷键一样，经实测并不会产生冲突。
此外，YCM单独跳转到定义或声明位置的命令分别是`YcmCompleter GoToDefinition`和`YcmCompleter GoToDeclaration`，
注意别漏了前置冒号和结尾`<CR>`。

## `jedi-vim`——用于Python的补全及跳转（在Python方面可能比YCM更专业）

### 创建VIM插件目录（若已存在则跳过）

````
mkdir -p ~/.vim/autoload ~/.vim/bundle
````

### 安装`pathogen`用于管理插件


直接用apt命令来安装（系统较新时推荐使用）：

````
sudo apt install vim-pathogen
````

或者把`pathogen`相关的VIM脚本下载到插件自动加载目录下亦可：

````
curl -LSso ~/.vim/autoload/pathogen.vim https://tpo.pe/pathogen.vim
````

安装完成后，在`~/.vimrc`文件添加以下语句：

````
execute pathogen#infect()
````

并且注意若`~/.vimrc`包含以下语句，则上述语句应放在其前面：

````
syntax on
filetype plugin indent on
````

### 安装及配置`jedi-vim`

#### 方法一（较简单，推荐在较新系统上使用，若行不通再使用方法二）：

````
sudo apt install vim-python-jedi
````

#### 方法二：

更新`jedi`：

````
pip install jedi
````

再安装`jedi-vim`：

````
git clone --recursive https://github.com/davidhalter/jedi-vim.git ~/.vim/bundle/jedi-vim
````

完毕。

### `jedi-vim`常用快捷键（摘自其官方GitHub）

* Completion ``<C-Space>``
* Goto assignments ``<leader>g`` (typical goto function)
* Goto definitions ``<leader>d`` (follow identifier as far as possible, includes imports and statements)
* Show Documentation/Pydoc ``K`` (shows a popup with assignments)
* Renaming ``<leader>r``
* Usages ``<leader>n`` (shows all the usages of a name)
* Open module, e.g. ``:Pyimport os`` (opens the ``os`` module)

其中，``<leader>``在VIM默认是反斜杠“\”。

### 参考材料

* https://github.com/tpope/vim-pathogen

* https://github.com/davidhalter/jedi-vim

## 常见配置

* 追加自定义的头文件目录（`Ctags`所需）。示例：`set path+=$HOME/include`，多个路径则用英文逗号隔开。

## 常用快捷键备忘录

### 内置

* 命令模式转为编辑模式：`i`（即：insert）

* 命令模式转为可视化模式：`v`（即：visual）

* 恢复命令模式：`Esc`

* 撤销、重做更改：分别为`u`（即：undo）、`Ctrl + r`（即：redo）

* 左、下、上、右移动光标：分别为`h`、`j`、`k`、`l`

* 向前、后翻屏：分别为`Ctrl + f`（即：forward）、`Ctrl + b`（即：backward）

* 跳到文件开头、结尾：分别为`gg`、`Shift + g`

* 跳到行开头、结尾：分别为`Shift + ^`、`Shift + $`

* 回跳（到光标上次所在）：`Ctrl + o`

* 前跳（与`Ctrl + o`相对）：`Ctrl + i`

* 跳到文件：`gf`（即：goto file。若是非系统目录里的头文件，还需设置`path`，见前面配置）

* 跳转到函数定义：`Ctrl + ]`

* 回跳：`Ctrl + t`

* 单行局部、单行整行、N行的复制：分别为`y`（需先在可视化模式下选定内容）、`yy`、`Nyy`
（N为具体行数，y为yank）之后，再`p`（即：paste）

* 单行局部、单行整行、N行的剪切/删除：与上相似，将`y`换成`d`（即：delete）即可，
且在删除时不需要加`p`

* 列模式插入：`Ctrl + v` -> 移动光标选取多行 -> `Shift + i` -> 输入要插入的内容 -> 两次`Esc`

* 列模式删除：`Ctrl + v` -> 移动光标选取多行 -> `d`

### `YouCompleteMe`

见`YouCompleteMe`一节。

### `jedi-vim`

见`jedi-vim`一节。

### 其它

暂无。

