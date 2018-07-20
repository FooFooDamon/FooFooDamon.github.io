<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# Ubuntu 16.04 安装 C++ 版 TensorFlow 1.4.1

## 下载安装包

1. 浏览器进入其***GitHub发布页***：[https://github.com/tensorflow/tensorflow/releases](https://github.com/tensorflow/tensorflow/releases)

2. 翻页，找到`1.4.1`版本，下载其压缩包，并放到合适的目录。本文下载的是`tensorflow-1.4.1.zip`，放到`~/src`目录。

## 编译及安装

### 安装依赖项

1. 安装`JDK 8`（若已装则略过）

```
sudo apt-get install openjdk-8-jdk
```

2. 安装`bazel`（至关重要！！！）

```
echo "deb [arch=amd64] http://storage.googleapis.com/bazel-apt stable jdk1.8" | sudo tee /etc/apt/sources.list.d/bazel.list

curl https://bazel.build/bazel-release.pub.gpg | sudo apt-key add -

sudo apt-get update

sudo apt-get install bazel
```

需要说明的是，如果编译器的功能、接口、语法等已趋于稳定，用以上方法是最方便的。然而现实并没那么美好，经本人亲测，这样安装得到的编译器用来编译低版本的`TensorFlow`，会报一大堆错。所以，必须安装对应版本的编译器才行（`TensorFlow`与`bazel`的版本对应关系见第6个参考链接）。

直接用`apt`来装低版本的`0.5.4`极有可能失败，因为有一大坨依赖关系没法满足，所以需要考虑直接下载其`Shell脚本`、`deb包`或者`源码`。尽管直接从官网下载可能会比较慢，这里还是贴出 [bazel官网下载地址](https://github.com/bazelbuild/bazel/releases) ，推荐在***早上***下载。如果出现龟速，可直接用度娘来搜，一般度娘网盘、*SDN、*浪网盘之流还是能找到资源的，你缺的可能就是一个账号或几个积分……废话不多说，本人下的是`bazel_0.5.4-linux-x86_64.deb`包，放到并进入`~/src`目录，然后执行以下命令即可安装：

```
sudo dpkg -i bazel_0.5.4-linux-x86_64.deb
```

然而，不保证这种方法对所有人所有环境都行得通。如果行不通，请自行尝试`Shell脚本`和`源码`的安装方式，后面的参考链接里有详细的说明，这里不再展开。

如果想固定`bazel`的版本，防止`apt`自动更新，可以删除它的源：

```
sudo rm /etc/apt/sources.list.d/bazel.list

sudo apt-get update
```

3. 安装`Python依赖项`

```
sudo apt-get install python-numpy python-dev python-pip python-wheel
```

4. 安装`CUDA`和`cuDNN`（GPU版本必需）

    可参考：[Ubuntu 16.04 安装 CUDA 8.0 和 cuDNN 7](https://github.com/FooFooDamon/FooFooDamon.github.io/blob/master/Ubuntu_16.04安装CUDA_8.0和cuDNN_7.md)


### 配置TensorFlow的编译条件

```
cd ~/src

unzip tensorflow-1.4.1.zip

cd tensorflow-1.4.1

# 执行配置脚本并按提示进行配置，以下示例仅供参考：

$ ./configure
WARNING: --batch mode is deprecated. Please instead explicitly shut down your Bazel server using the command "bazel shutdown".
You have bazel 0.15.2 installed.
Please specify the location of python. [Default is /usr/bin/python]: 


Found possible Python library paths:
  /usr/local/lib/python2.7/dist-packages
  /usr/lib/python2.7/dist-packages
Please input the desired Python library path to use.  Default is [/usr/local/lib/python2.7/dist-packages]

Do you wish to build TensorFlow with jemalloc as malloc support? [Y/n]: 
jemalloc as malloc support will be enabled for TensorFlow.

Do you wish to build TensorFlow with Google Cloud Platform support? [Y/n]: n
No Google Cloud Platform support will be enabled for TensorFlow.

Do you wish to build TensorFlow with Hadoop File System support? [Y/n]: n
No Hadoop File System support will be enabled for TensorFlow.

Do you wish to build TensorFlow with Amazon S3 File System support? [Y/n]: n
No Amazon S3 File System support will be enabled for TensorFlow.

Do you wish to build TensorFlow with XLA JIT support? [y/N]: 
No XLA JIT support will be enabled for TensorFlow.

Do you wish to build TensorFlow with GDR support? [y/N]: 
No GDR support will be enabled for TensorFlow.

Do you wish to build TensorFlow with VERBS support? [y/N]: 
No VERBS support will be enabled for TensorFlow.

Do you wish to build TensorFlow with OpenCL support? [y/N]: 
No OpenCL support will be enabled for TensorFlow.

Do you wish to build TensorFlow with CUDA support? [y/N]: y
CUDA support will be enabled for TensorFlow.

Please specify the CUDA SDK version you want to use, e.g. 7.0. [Leave empty to default to CUDA 8.0]: 


Please specify the location where CUDA 8.0 toolkit is installed. Refer to README.md for more details. [Default is /usr/local/cuda]: 


Please specify the cuDNN version you want to use. [Leave empty to default to cuDNN 6.0]: 7


Please specify the location where cuDNN 7 library is installed. Refer to README.md for more details. [Default is /usr/local/cuda]:/usr/lib/x86_64-linux-gnu              


Please specify a list of comma-separated Cuda compute capabilities you want to build with.
You can find the compute capability of your device at: https://developer.nvidia.com/cuda-gpus.
Please note that each additional compute capability significantly increases your build time and binary size. [Default is: 6.1]


Do you want to use clang as CUDA compiler? [y/N]: 
nvcc will be used as CUDA compiler.

Please specify which gcc should be used by nvcc as the host compiler. [Default is /usr/bin/gcc]: 


Do you wish to build TensorFlow with MPI support? [y/N]: 
No MPI support will be enabled for TensorFlow.

Please specify optimization flags to use during compilation when bazel option "--config=opt" is specified [Default is -march=native]: 


Add "--config=mkl" to your bazel command to build with MKL support.
Please note that MKL on MacOS or windows is still not supported.
If you would like to use a local MKL instead of downloading, please set the environment variable "TF_MKL_ROOT" every time before build.
Configuration finished
```

### 编译TensorFlow

执行：

```
bazel build -c opt --verbose_failures //tensorflow:libtensorflow_cc.so
```

注意最好要指定`--verbose_failures`，这样当出错时（极大几率会出错）会打印详细信息。

话没说完就报了个bazel版本“过低”的错误，如下：

```
ERROR: /home/foo/src/tensorflow-1.4.1/WORKSPACE:15:1: Traceback (most recent call last):
	File "/home/foo/src/tensorflow-1.4.1/WORKSPACE", line 15
		closure_repositories()
	File "/home/foo/.cache/bazel/_bazel_foo/8afcca331dcb8a9f02f9a9565832584e/external/io_bazel_rules_closure/closure/repositories.bzl", line 69, in closure_repositories
		_check_bazel_version("Closure Rules", "0.4.5")
	File "/home/foo/.cache/bazel/_bazel_foo/8afcca331dcb8a9f02f9a9565832584e/external/io_bazel_rules_closure/closure/repositories.bzl", line 172, in _check_bazel_version
		fail(("%s requires Bazel >=%s but was...)))
Closure Rules requires Bazel >=0.4.5 but was 0.15.2
ERROR: Error evaluating WORKSPACE file
Closure Rules requires Bazel >=0.4.5 but was 0.15.2
ERROR: Error evaluating WORKSPACE file
ERROR: /home/foo/src/tensorflow-1.4.1/WORKSPACE:41:1: Traceback (most recent call last):
	File "/home/foo/src/tensorflow-1.4.1/WORKSPACE", line 41
		tf_workspace()
	File "/home/foo/src/tensorflow-1.4.1/tensorflow/workspace.bzl", line 146, in tf_workspace
		check_version("0.5.4")
	File "/home/foo/src/tensorflow-1.4.1/tensorflow/workspace.bzl", line 56, in check_version
		fail("\nCurrent Bazel version is {}, ...))

Current Bazel version is 0.15.2, expected at least 0.5.4
```

事实上并非bazel版本过低，而是编译脚本的逻辑缺陷，即简单地进行纯数学的小数比较，导致得出`0.5.4 > 0.15.2`的结论。解决方法是打开上述的`workspace.bzl`文件，定位到报错的那一行（上面为第146行），将`check_version("0.5.4")`改为`check_version("0.0.0")`，再用同样的方法修改`repositories.bzl`文件，最后重新调用bazel进行编译即可。

如果出现如下标签（label）错误：

```
ERROR: /home/foo/.cache/bazel/_bazel_foo/8afcca331dcb8a9f02f9a9565832584e/external/local_config_sycl/sycl/BUILD:4:1: First argument of 'load' must be a label and start with either '//', ':', or '@'.
ERROR: /home/foo/.cache/bazel/_bazel_foo/8afcca331dcb8a9f02f9a9565832584e/external/local_config_sycl/sycl/BUILD:6:1: First argument of 'load' must be a label and start with either '//', ':', or '@'.
ERROR: /home/foo/.cache/bazel/_bazel_foo/8afcca331dcb8a9f02f9a9565832584e/external/local_config_sycl/sycl/BUILD:4:1: file 'platform' was not correctly loaded. Make sure the 'load' statement appears in the global scope in your file
ERROR: /home/foo/.cache/bazel/_bazel_foo/8afcca331dcb8a9f02f9a9565832584e/external/local_config_sycl/sycl/BUILD:6:1: file 'platform' was not correctly loaded. Make sure the 'load' statement appears in the global scope in your file
```

可在`~/.cache/bazel`目录下搜索名称带`platform`的`bzl`文件，如下：

```
$ find /home/foo/.cache/bazel -name "*platform*"
# 省略一部分输出
/home/foo/.cache/bazel/_bazel_foo/8afcca331dcb8a9f02f9a9565832584e/external/local_config_sycl/sycl/platform.bzl
```

找到后就打开以上`BUILD`文件，将出错的`load("platform", ...)`语句改为`load("@local_config_sycl//sycl:platform.bzl", ...)`。bazel语法详见第8个参考链接。

若出现某个依赖项有多个匹配结果时，如下：

```
ERROR: /home/foo/.cache/bazel/_bazel_foo/8afcca331dcb8a9f02f9a9565832584e/external/jpeg/BUILD:122:12: Illegal ambiguous match on configurable attribute "deps" in @jpeg//:jpeg:
@jpeg//:k8
@jpeg//:armeabi-v7a
Multiple matches are not allowed unless one is unambiguously more specialized.
```

可打开`BUILD`文件，定位到出错行附近，如下：

```
122     deps = select({
123         ":k8": [":simd_x86_64"],
124         ":armeabi-v7a": [":simd_armv7a"],
125         ":arm64-v8a": [":simd_armv8a"],
126         "//conditions:default": [":simd_none"],
127     }), 
128 )
```

将`armeabi-v7a`所在行删除。

如无意外，此时应进入真正的编译操作。视`bazel`的版本而定，如果不是`0.5.4`（若是用`apt`来安装，默认安装最新，即以上的`0.15.2`，指定版本号来安装则可能出现依赖条件不满足而安装不了），还有可能出幺蛾子，例如：

```
ERROR: /home/foo/.cache/bazel/_bazel_foo/8afcca331dcb8a9f02f9a9565832584e/external/jpeg/BUILD:40:1: C++ compilation of rule '@jpeg//:jpeg' failed (Exit 1): crosstool_wrapper_driver_is_not_gcc failed: error executing command 
  (cd /home/foo/.cache/bazel/_bazel_foo/8afcca331dcb8a9f02f9a9565832584e/execroot/org_tensorflow && \
  exec env - \
    LD_LIBRARY_PATH=/home/foo/lib:/usr/local/lib:/usr/local/cuda/lib64 \
    PATH=/home/foo/git/lazy_script/details/private:/home/foo/git/lazy_script/details/shortcuts:/home/foo/git/lazy_script/details:/home/foo/bin:/home/foo/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/home/foo/linux/bin:/usr/local/cuda/bin \
    PWD=/proc/self/cwd \
  external/local_config_cuda/crosstool/clang/bin/crosstool_wrapper_driver_is_not_gcc -U_FORTIFY_SOURCE '-D_FORTIFY_SOURCE=1' -fstack-protector -fPIE -Wall -Wunused-but-set-parameter -Wno-free-nonheap-object -fno-omit-frame-pointer -g0 -O2 -DNDEBUG -ffunction-sections -fdata-sections -MD -MF bazel-out/host/bin/external/jpeg/_objs/jpeg/external/jpeg/jcapimin.pic.d -fPIC -iquote external/jpeg -iquote bazel-out/host/genfiles/external/jpeg -iquote external/bazel_tools -iquote bazel-out/host/genfiles/external/bazel_tools -g0 -O3 -w -D__ARM_NEON__ '-march=armv7-a' '-mfloat-abi=softfp' -fprefetch-loop-arrays -no-canonical-prefixes -Wno-builtin-macro-redefined '-D__DATE__="redacted"' '-D__TIMESTAMP__="redacted"' '-D__TIME__="redacted"' -fno-canonical-system-headers -c external/jpeg/jcapimin.c -o bazel-out/host/bin/external/jpeg/_objs/jpeg/external/jpeg/jcapimin.pic.o)
gcc: error: unrecognized command line option '-mfloat-abi=softfp'
```

看`crosstool/clang/bin/crosstool_wrapper_driver_is_not_gcc`、`-march=armv7-a`、`-mfloat-abi=softfp`几个地方，觉得是交叉编译出了问题，但之前执行`configure`脚本时，明明使用了默认的`-march=native`选项，这就蛋疼了……于是，生平第一次怀疑编译器，而且还真怀疑对了。首先`bazel`在解析其`.bzl`文件时不应该报那么多错误，而且看起来是高版本与低版本严重不兼容；其次编译时的参数貌似没有完全按照之前`configure`的要求来做。种种现象只能归结为要么是`bazel`编译器的问题，要么是`.bzl`编译脚本的写法有问题，尽管编译器版本不是公开可用的`1.0`以上的版本，但作为一个大厂的出品搞出这么多蛋疼的问题，还是着实令人费解。后来和别人交流一下，发现人家同样是`TensorFlow 1.4.1`，编译的时候顺风顺水，再看他系统装的`bazel`版本，赫然是`0.5.4`！果断卸载原来的`bazel`，重新装上`0.5.4`的版本（见前面），最后编译的结果是一路绿灯（当然，一些编译警告还是有的，没有警告的代码是不存在的，这辈子都不可能看到，除非是`Hello World`）！

### 手动拷贝头文件和库文件

```
待补充
```

### 验证安装情况

```
待补充
```

## 参考（部分链接可能需要翻墙）

[https://docs.bazel.build/versions/master/install.html](https://docs.bazel.build/versions/master/install.html)

[https://docs.bazel.build/versions/master/install-ubuntu.html](https://docs.bazel.build/versions/master/install-ubuntu.html)

[https://docs.bazel.build/versions/master/install-compile-source.html](https://docs.bazel.build/versions/master/install-compile-source.html)

[https://www.tensorflow.org/install/](https://www.tensorflow.org/install/)

[https://www.tensorflow.org/install/install_linux#determine_which_tensorflow_to_install](https://www.tensorflow.org/install/install_linux#determine_which_tensorflow_to_install)

[https://www.tensorflow.org/install/install_sources](https://www.tensorflow.org/install/install_sources)

[https://github.com/bazelbuild/bazel/issues/4834](https://github.com/bazelbuild/bazel/issues/4834)

[https://blog.csdn.net/u013510838/article/details/80102438](https://blog.csdn.net/u013510838/article/details/80102438)

## 备注

1. 本文针对的是C++版本的TensorFlow，另有Python版本（使用最广泛）和C版本（偏重于简洁性和一致性，安装最简单，但调用时不太方便），详见官方说明文档。

2. 操作之前必须仔细阅读官方说明文档，所用的依赖库和编译工具的版本尽量与之一致，这样能踩少很多坑。如果还有坑，就需要分析报错信息，以及自行搜索官方文档未及之处的资料。

