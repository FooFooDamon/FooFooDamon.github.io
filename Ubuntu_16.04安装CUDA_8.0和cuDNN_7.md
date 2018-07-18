<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# Ubuntu 16.04 安装 CUDA 8.0 和 cuDNN 7

## 下载及安装CUDA

1. 浏览器进入***CUDA下载页***（网址仅供参考，以后有可能会失效）：[https://developer.nvidia.com/cuda-toolkit-archive](https://developer.nvidia.com/cuda-toolkit-archive)

2. 点击8.0的版本链接（该版本还细分为GA1和GA2，本文使用GA2），跳转到另一个页面，手动选择以下选项：`Operating System`为`Linux`、`Architecture`为`x86_64`、`Distribution`为`Ubuntu`、`Version`为`16.04`、`Installer Type`为`runfile (local)`，出现最终的下载按钮，点击即可下载，得到`cuda_8.0.61_375.26_linux-run`并放到用户家目录下。

3. 按以下操作进行安装：

```
cd ~

sudo sh cuda_8.0.61_375.26_linux-run

根据提示选择及设置一些参数即可，以下示例仅供参考：

（省略安装协议内容……）
-------------------------------------------------------------
Do you accept the previously read EULA?
accept/decline/quit: accept

Install NVIDIA Accelerated Graphics Driver for Linux-x86_64 375.26?
(y)es/(n)o/(q)uit: n

Install the CUDA 8.0 Toolkit?
(y)es/(n)o/(q)uit: y

Enter Toolkit Location
 [ default is /usr/local/cuda-8.0 ]: 

Do you want to install a symbolic link at /usr/local/cuda?
(y)es/(n)o/(q)uit: y

Install the CUDA 8.0 Samples?
(y)es/(n)o/(q)uit: y

Enter CUDA Samples Location
 [ default is /home/foo ]: /home/foo/src/cuda

Installing the CUDA Toolkit in /usr/local/cuda-8.0 ...
Installing the CUDA Samples in /home/foo/src/cuda ...
Copying samples to /home/foo/src/cuda/NVIDIA_CUDA-8.0_Samples now...
Finished copying samples.

===========
= Summary =
===========

Driver:   Not Selected
Toolkit:  Installed in /usr/local/cuda-8.0
Samples:  Installed in /home/foo/src/cuda

Please make sure that
 -   PATH includes /usr/local/cuda-8.0/bin
 -   LD_LIBRARY_PATH includes /usr/local/cuda-8.0/lib64, or, add /usr/local/cuda-8.0/lib64 to /etc/ld.so.conf and run ldconfig as root

To uninstall the CUDA Toolkit, run the uninstall script in /usr/local/cuda-8.0/bin

Please see CUDA_Installation_Guide_Linux.pdf in /usr/local/cuda-8.0/doc/pdf for detailed information on setting up CUDA.

***WARNING: Incomplete installation! This installation did not install the CUDA Driver. A driver of version at least 361.00 is required for CUDA 8.0 functionality to work.
To install the driver using this installer, run the following command, replacing <CudaInstaller> with the name of this run file:
    sudo <CudaInstaller>.run -silent -driver

Logfile is /tmp/cuda_install_15336.log
```

4. ***注意事项***：

a) 以上操作步骤有一项：***Install NVIDIA Accelerated Graphics Driver for Linux-x86_64 375.26?***，最好输入`n`跳过，不然安装起来非常麻烦，容易与当前系统的一些东西产生冲突。该图形驱动可在CUDA之前单独安装，可参考 [Ubuntu 16.04 安装 NVIDIA TITAN Xp 显卡驱动](https://github.com/FooFooDamon/FooFooDamon.github.io/blob/master/Ubuntu_16.04安装NVIDIA_TITAN_Xp显卡驱动.md) 一文。

b) 安装完成时若报某些库缺失，类似这样的信息：

```
Installing the CUDA Toolkit in /usr/local/cuda-8.0 …
Missing recommended library: libGLU.so
Missing recommended library: libXi.so
Missing recommended library: libXmu.so
```

可先把这些依赖库装上，再重新安装CUDA，如下：

```
sudo apt-get install freeglut3-dev libxmu-dev libxi-dev libgl1-mesa-glx libglu1-mesa libglu1-mesa-dev

sudo /usr/local/cuda-8.0/bin/uninstall_cuda_8.0.pl
rm -rf ~/src/cuda

sudo sh cuda_8.0.61_375.26_linux-run
```

若想确认依赖库已装上，可用apt-file命令，例如：

```
$ apt-file search libGLU.so
libglu1-mesa: /usr/lib/x86_64-linux-gnu/libGLU.so.1
libglu1-mesa: /usr/lib/x86_64-linux-gnu/libGLU.so.1.3.1
libglu1-mesa-dev: /usr/lib/x86_64-linux-gnu/libGLU.so
```

若还发现缺失库，可再按以上方法给装上（库对应的包名可用搜索引擎查询），再重装CUDA，直到警告消失，表明安装成功。


## 下载及安装cuDNN

1. 浏览器进入***cuDNN下载页***（网址仅供参考，以后有可能会失效）：[https://developer.nvidia.com/rdp/cudnn-archive](https://developer.nvidia.com/rdp/cudnn-archive)

2. 点击v7.0.x的版本链接（本文使用`v7.0.5 for CUDA 8.0`），选择适用于`Ubuntu 16.04`的库下载即可。***注意可能需要注册或登录***，过程不详述，最终下载到的文件为`libcudnn7_7.0.5.15-1+cuda8.0_amd64.deb`（也可根据自己的需要选择其它格式），并放到用户家目录下。

3. 安装操作很简单，用`dpkg`命令即可，以下是示例及其输出：

```
$ sudo dpkg -i libcudnn7_7.0.5.15-1+cuda8.0_amd64.deb 
(正在读取数据库 ... 系统当前共安装有 271165 个文件和目录。)
正准备解包 libcudnn7_7.0.5.15-1+cuda8.0_amd64.deb  ...
正在将 libcudnn7 (7.0.5.15-1+cuda8.0) 解包到 (7.0.5.15-1+cuda8.0) 上 ...
正在设置 libcudnn7 (7.0.5.15-1+cuda8.0) ...
正在处理用于 libc-bin (2.23-0ubuntu10) 的触发器 ...
```

这种方式的安装如果报错，一般是某些依赖关系没满足，根据其报错的信息补足依赖关系即可，无法在此详述。

## 参考

[https://blog.csdn.net/10km/article/details/61915535](https://blog.csdn.net/10km/article/details/61915535)

[https://blog.csdn.net/lucifer_zzq/article/details/76675239](https://blog.csdn.net/lucifer_zzq/article/details/76675239)

