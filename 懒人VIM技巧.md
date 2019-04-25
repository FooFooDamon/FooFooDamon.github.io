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

## 入门级插件

* `Ctags`：代码跳转

* `TagList`：高效浏览源代码

* `WinManager`：窗口管理

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

### 添加自定义头文件目录

新建或追加`~/.vim/.ycm_extra_conf.py`：

````
flags = [
    '-isystem',
    '/your/header/path1',

	'-isystem',
    '/your/header/path2',

	<More ...>
]
````

值得说明的是，若已经将这些目录`set`进`path`里（见后面配置），则该配置操作可以不做。

## `jedi-vim`——用于python的补全及跳转

### 创建VIM插件目录（若已存在则跳过）

````
mkdir -p ~/.vim/autoload ~/.vim/bundle
````

### 安装`pathogen`用于管理插件

把`pathogen`相关的VIM脚本下载到插件自动加载目录下即可完成安装：

````
curl -LSso ~/.vim/autoload/pathogen.vim https://tpo.pe/pathogen.vim
````

然后，在`~/.vimrc`文件添加以下语句：

````
execute pathogen#infect()
````

并且注意若`~/.vimrc`包含以下语句，则上述语句应放在其前面：

````
syntax on
filetype plugin indent on
````

### 安装及配置`jedi-vim`

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

* 追加自定义的头文件目录。示例：`set path+=$HOME/include`

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

### `jedi-vim`

见`jedi-vim`一节。

### 其它

待补充

