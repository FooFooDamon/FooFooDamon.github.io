<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# <center>git命令摘录</center>

<center><a href="README.md">&lt;&lt;&lt; 返回主页</a></center>

**特别注意**：

1. 以`${}`括起来的内容表示要根据实际情况填写，填写之后要把`${}`去掉。

2. 以`[]`括起来的内容表示可选，不填或填写之后都要把`[]`去掉。

## 1、克隆

### 1.1 基本用法（含全量历史）

````
$ git clone '${URL}' ['${LOCAL-NAME}']
````

### 1.2 仅克隆最新一版

````
$ git clone --depth=1 '${URL}' ['${LOCAL-NAME}']
````

### 1.3 指定标签

````
$ git clone -b '${TAG}' '${URL}' ['${LOCAL-NAME}']
````

### 1.4 递归

主要用于包含**仅以`链接`/`地址`/`快捷方式`的形式呈现的`子`项目**的项目，
可用以下命令将`主`项目和所有`子`项目一次性克隆下来：

````
$ git clone --recursive '${URL}' ['${LOCAL-NAME}']
````

## 2、拉取（即更新）

### 2.1 基本用法

````
$ git pull
````

## 3、加入版本库（本地提交的前置操作）

### 3.1 基本用法

````
$ git add '${PATH1}'[ '${PATH2}' '${PATH3}' ...]
````

## 4、本地提交

### 4.1 全量

````
$ git commit -a -m '${COMMENT}'
````

### 4.2 部分

````
$ git commit '${PATH1}'[ '${PATH2}' '${PATH3}' ...] -m '${COMMENT}'
````

### 4.3 仅修改注释（必须在未提交到远程之前）

````
$ git commit --amend -m '${COMMENT}'
````

### 4.4 补充本应同一批次提交的遗漏文件（必须在未提交到远程之前）

````
$ git add '${PATH1}'[ '${PATH2}' '${PATH3}' ...]
$ git commit --amend --no-edit
````

## 5、提交到远程

### 5.1 基本用法

````
$ git push
````

## 6、标签

### 6.1 新增

````
$ git tag -a '${TAG}' -m '${COMMENT}'
$ git push origin '${TAG}'
````

### 6.2 删除

````
$ git tag -d '${TAG}'
$ git push origin ':refs/tags/${TAG}'
````

<center><a href="README.md">&lt;&lt;&lt; 返回主页</a></center>

