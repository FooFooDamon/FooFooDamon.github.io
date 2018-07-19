<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# Ubuntu 16.04 安装 Caffe 1.0

## 下载安装包

1. 浏览器进入其***GitHub发布页***：[https://github.com/BVLC/caffe/releases](https://github.com/BVLC/caffe/releases)

2. 找到`1.0`版本，下载其压缩包，并放到合适的目录。本文下载的是`caffe-1.0.zip`，放到`~/src`目录。

## 编译及安装

### 安装依赖项

Caffe安装说明网页（见后面参考链接）用少数几条命令就能安装好所有的依赖项，本文为了说明依赖项的详情，特意分开安装，安装顺序与Caffe说明网页声明依赖项的顺序基本一致。

1. 安装`CUDA`和`cuDNN`（GPU版本必需）

    可参考：[Ubuntu 16.04 安装 CUDA 8.0 和 cuDNN 7](https://github.com/FooFooDamon/FooFooDamon.github.io/blob/master/Ubuntu_16.04安装CUDA_8.0和cuDNN_7.md)

2. 安装`BLAS`

```
sudo apt install libatlas-base-dev
```

3. 安装`Boost`

```
sudo apt install --no-install-recommends libboost-all-dev

备注：要求版本 >= 1.55，apt安装的是1.58。
```

4. 安装`protobuf`、`glog`、`gflags`、`hdf5`

```
sudo apt install libprotobuf-dev protobuf-compiler libgoogle-glog-dev libgflags-dev libhdf5-serial-dev

备注：apt安装的protobuf版本是2.6.1，满足BLVC分支1.0版本的caffe，若是Intel分支1.1.1版本，则要求3.5.0以上，所以不适合在Ubuntu 16.04安装。
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

### 安装caffe

用cmake方式安装，如下：

```
cd ~/src
unzip caffe-1.0.zip
cd caffe-1.0
mkdir build
cd build
# 要求系统先装好2.8.7版本以上的cmake
cmake ..
make all
make install

最好再加上：
sudo cp -a -r include/* /usr/local/include/
sudo cp -a lib/libcaffe.* /usr/local/lib/
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

Caffe分为几个版本，本文所用的是原版，即BLVC（[b]erke[l]ey[v]ision [c]affe）。另有Intel、OpenCL、Windows版本，在原版的基础上有特定方面的优化或定制化，详情可由[BLVC的GitHub主页](https://github.com/BVLC/caffe.git)跳转到相应版本的页面去了解，这里不详述。

此外，Caffe的运行方式可分为`仅CPU运行`和`有GPU参与`两种方式，安装要求和方式有所不同，后续将进行补充。

