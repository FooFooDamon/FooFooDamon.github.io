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
<a href="手把手教你把Vim改装成一个IDE编程环境(图文) - 吴垠的专栏 - 博客频道 - CSDN.NET.pdf">这里</a>
查看备份文档。

## 常用快捷键备忘录

待补充……

## 快速安装`YouCompleteMe`插件

以下适用于`Ubuntu`系统，版本`14.04`以上：

````
sudo apt-get install vim-addon-manager 
sudo apt-get install vim-youcompleteme
vim-addons install youcompleteme
````

网上很多教程可能因为写得较早的原因，安装过程都比较繁琐。此处则只需几条命令，
不需额外配置，装好后开箱即用。当然，需要更高级、更个性化的定制则除外。

