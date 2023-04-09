<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# Ubuntu 16.04 安装 Caffe 1.0

## 下载安装包

1. 浏览器进入其***GitHub发布页***：[https://github.com/BVLC/caffe/releases](https://github.com/BVLC/caffe/releases)

2. 找到`1.0`版本，下载其压缩包，并放到合适的目录。本文下载的是`caffe-1.0.zip`，放到`~/src`目录。

## 编译及安装

### 安装依赖项

Caffe安装说明网页（见后面参考链接）用少数几条命令就能安装好所有的依赖项，本文为了说明依赖项的详情，特意分开安装，安装顺序与Caffe说明网页声明依赖项的顺序基本一致。

1. 安装`CUDA`和`cuDNN`（GPU版本必需）

    可参考：[Ubuntu 16.04 安装 CUDA 8.0 和 cuDNN 7](Ubuntu_16.04安装CUDA_8.0和cuDNN_7.md)

2. 安装`BLAS`

基础版（无多线程加速）则执行：

```
sudo apt install libatlas-base-dev
```

可多线程运行版则执行：

````
sudo apt install libopenblas-dev
````

3. 安装`Boost`

```
sudo apt install --no-install-recommends libboost-all-dev

备注：要求版本 >= 1.55，apt安装的是1.58。
```

4. 安装`protobuf`、`glog`、`gflags`、`hdf5`

```
sudo apt install libprotobuf-dev protobuf-compiler libgoogle-glog-dev libgflags-dev libhdf5-serial-dev

备注：关于protobuf版本的特别说明见后面内容。
```

5. 安装`OpenCV`（可选）

```
sudo apt install libopencv-dev

备注：要求版本 >= 2.4，apt安装的是2.4.9。
```

6. 安装`lmdb`，`leveldb`（可选，其中leveldb还依赖`snappy`）

```
sudo apt install liblmdb-dev libleveldb-dev libsnappy-dev
```

### 关于protobuf的特别说明

若使用apt安装的protobuf，其版本在Ubuntu 16.04下是`2.6.1`，满足BVLC分支1.0版本的caffe，
若是Intel分支1.1.1版本，则要求`3.5.0`以上，直接编译可能会报错。<br>

另外，如果同时装有其它深度学习框架，例如`TensorFlow`，极有可能会出现**protobuf版本打架**的现象！
一种想法是把apt安装的protobuf版本删除，使用自定义安装的版本。然而，却不能保证以后安装的软件会不会
再次自动将默认版本protobuf安装回来，毕竟一些复杂的程序、库或框架的依赖太深，且包管理器一些“自动化”
操作暗地里干了些啥我们也是很难一一获知的。所以，最好有一种允许多个protobuf版本共存的方法。<br>

下面介绍我个人用的一种方法：<br>

首先，确定好自定义安装的protobuf的位置（最好不是系统目录），例如安装在用户家目录`$HOME`，
即protoc装在`$HOME/bin`，头文件装在`$HOME/include`，库文件装在`$HOME/lib`，编译方法一般是采用源码
编译，这里不详细介绍。装好之后，在编译caffe之前，进行以下设置：

```
export CMAKE_INCLUDE_PATH=$HOME/include
export CMAKE_LIBRARY_PATH=$HOME/lib
export PATH=$HOME/bin:$PATH
```

即可让caffe使用自定义安装的protobuf版本了。这里不得不赞一下BVLC版本的caffe，其使用的`proto/caffe.pb.h`
及对应的源文件是在编译时实时生成的，而不是预先生成并放进源码压缩包。实时生成的灵活性不必多说，可以适应
不同版本的protobuf。另外，其`CMakeLists.txt`也写得很好，否则以上的`CMAKE_INCLUDE_PATH`和`CMAKE_LIBRARY_PATH`
环境变量就算设置了也没有用处，也就切换不到装在非系统目录的protobuf库了。

### 安装caffe

用cmake方式安装，如下：

```
cd ~/src
unzip caffe-1.0.zip
cd caffe-1.0
mkdir build
cd build
# 要求系统先装好2.8.7版本以上的cmake
# 若只使用CPU，则需要加上：-DCPU_ONLY=ON
# 若前面安装的是openblas，则需要加上：-DBLAS=open
# 若要安装到非系统目录，例如/usr/local，则可加上：-DCMAKE_INSTALL_PREFIX=/usr/local
# 更多编译选项可自行查阅CMakeLists.txt或Makefile
cmake ..
make
sudo make install
```

还可以直接用make的方式进行安装，需要修改`Makefile.config`，详见参考链接。

### 验证安装情况

```
make runtest
```

## 参考

[http://caffe.berkeleyvision.org/installation.html](http://caffe.berkeleyvision.org/installation.html)

[http://caffe.berkeleyvision.org/install_apt.html](http://caffe.berkeleyvision.org/install_apt.html)

## 备注

Caffe分为几个版本，本文所用的是原版，即BVLC（Berkeley Vision and Learning Center）。另有Intel、OpenCL、Windows版本，在原版的基础上有特定方面的优化或定制化，详情可由[BVLC的GitHub主页](https://github.com/BVLC/caffe.git)跳转到相应版本的页面去了解，这里不详述。

此外，Caffe的运行方式可分为`仅CPU运行`和`有GPU参与`两种方式，安装要求和方式有所不同，后续将进行补充。

