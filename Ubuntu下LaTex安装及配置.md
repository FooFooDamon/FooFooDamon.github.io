<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<base target="_blank" />

## 简介

`LaTex`是一种基于`TeX`的异常强悍的排版系统，其主要特（优）点包括但不限于：

* 其源文件内容以***纯文本***的形式进行组织。这意味着：理论上不需安装笨重
的编辑器或IDE，仅用系统自带记事本或VIM就能进行编辑——UNIX/Linux使用者最爱的方式。

* 能呈现丰富的版面效果，尤其对于***复杂表格和数学公式***表现得更突出，
所以在学术界和（也许是部分的）出版界很流行。

* 其配套的命令或编辑器/IDE***能生成多种文档格式***（包括常见的PDF格式）。

* ***编辑器的选择多，而且能运行于Linux、Windows、Mac***等，这对于经常跨平台
工作的人是一大福音，不必再在WPS、Office等软件间切换以及为它们的文档格式的
互相兼容性而抓狂了。

还有更多的特点就不再多说了。LaTex虽好，但也要搭配相应的工具软件方能发挥其
最大的效用，用起来也更称手更爽。所谓工欲善其事必先利其器，下面将介绍LaTex
相关工具的安装及配置。

## 安装及配置

### 基础环境的安装

这里选择的是`texlive`。也许是系统比较新，且经过多年发展成熟的原因，我在
Ubuntu 16.04和18.04上仅用以下命令就能安装，不再需要额外的配置：

```
sudo apt install texlive-full texlive-lang-all
```

唯一需要注意的是可能需要找准时间安装。我第一次在晚上8、9点安装，下载速度
堪比乌龟爬，手动中断，然后在第二天早上再装，速度就赶上兔子了。

### 可视化编辑器的安装

这里选择的是`Texmaker`，同样是一条命令搞定：

```
sudo apt install texmaker
```

这一步没啥好说的，相当于给前面一步的基础环境加个可视化界面的壳，要下载的
东西也不多。

### 中文字体包的安装

本来只打算装纯中文包就够用了，结果看网上教程都是买一送二，大多装CJK
（Chinese、Japanese、Korean）……多了就多了，装！还是一条命令搞定：

```
sudo apt install latex-cjk-all
```

顺便说一下，链接 https://blog.csdn.net/baccon/article/details/51301683 里
`install-tl-unx`、`CJK`、`字体文件`的操作在我这没什么卵用，可能是系统版本
和软件源的关系，其中`install-tl-unx`还贼多东西，浪费了2G多的手机流量。

### 安装小结

以上把安装步骤拆分开来只是为了更清楚地说明需要哪些组件，实际上可以把这些
组件写在同一条命令里运行，实现一条命令搞定一整套的美梦。

### 中文环境配置

在命令行终端输入`texmaker`打开编辑器，然后在`选项` -> `配置Texmaker`对话框
的`命令`选项卡将LaTex命令由`latex`改为`xelatex`，改好后的样子大致如下：

```
xelatex -interaction=nonstopmode %.tex
```

然后，切换到`快速构建`选项卡，将`快速构建命令`选择为`XeLaTex + 查看PDF`。

注意，以上操作的名称根据系统环境及汉化程度的不同，可能会呈现英文或半英半中
的名称，注意变通即可。

## 简单测试

新建一个`xx.tex`文件，写入以下内容：

```
\documentclass{article}
\usepackage{xeCJK}
\setCJKmainfont{AR PL UKai CN}
\begin{document}
你好hello
\end{document}
```

然后点击工具栏上`快速构建`旁边的运行箭头图标，即可预览和生成正常包含中文
的PDF文档，重点在于`\setCJKmainfont`一句，需要正确设置所使用的中文字体方可。
要查看本机已安装的中文字体，可使用`fc-list`命令，命令及输出内容（举例）为：

```
$ fc-list :lang=zh-cn
/usr/share/fonts/truetype/arphic/uming.ttc: AR PL UMing TW MBE:style=Light
/home/foo/.fonts/simsun.ttc: 宋体,SimSun:style=Regular
/usr/share/fonts/opentype/noto/NotoSansCJK-Bold.ttc: Noto Sans CJK JP,Noto Sans CJK JP Bold:style=Bold,Regular
/usr/share/fonts/truetype/arphic/ukai.ttc: AR PL UKai CN:style=Book
```

## 参考材料

<a href="references/分享_10 款 Linux 平台上最好的 LaTeX 编辑器.pdf">分享_10 款 Linux 平台上最好的 LaTeX 编辑器</a>

<a href="references/Linux环境搭建中文LaTeX排版系统 - Blog - CSDN博客.pdf">Linux环境搭建中文LaTeX排版系统</a>

<a href="references/TexMaker安装及中文输出-简单生活.pdf">TexMaker安装及中文输出</a>

<a href="references/解决LaTeX中文输出问题_金玉木石_新浪博客.pdf">解决LaTeX中文输出问题</a>

<a href="references/再说在LaTeX中使用中文字体_jowtte_新浪博客.pdf">再说在LaTeX中使用中文字体</a>

