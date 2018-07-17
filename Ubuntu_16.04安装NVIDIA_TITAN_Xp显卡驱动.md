<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# Ubuntu 16.04 安装 NVIDIA TITAN Xp 显卡驱动

## 下载驱动

1. 浏览器进入***驱动下载页***（网址仅供参考，以后有可能会失效）：[https://www.nvidia.cn/Download/index.aspx?lang=cn](https://www.nvidia.cn/Download/index.aspx?lang=cn)

2. 在页面手动选择产品类型、产品系列、产品家族、操作系统等参数，然后点击搜索按钮跳转到对应的下载页。要注意的是操作系统这一项，如果精确选择“Linux 64-bit Ubuntu 16.04”搜不到驱动时，可退而求其次，重新选择“Linux 64-bit”。

3. 跳转到下载页面进行下载。撰写本文之时（2018.7.17）能下到的最新版本是390.77，发布日期是2018.7.16，文件名是`NVIDIA-Linux-x86_64-390.77.run`。然后，将其放到用户家目录下（其它目录也可以，但***路径最好不要带中文***，后面会说明原因）。

## 安装驱动

1. 先卸载旧驱动（可选，无则跳过）：

```
sudo apt-get remove --purge nvidia*
```

2. 禁用系统默认安装并使用的`nouveau`集成显卡驱动。只有禁用默认的驱动才能顺利安装NVIDIA显卡驱动。禁用方法如下：

```
a) 打开相关文件：sudo gedit /etc/modprobe.d/blacklist.conf

b) 将nouveau加入黑名单：在末尾加上 blacklist nouveau。有网友还加了以下几行，供参考：

  blacklist vga16fb

  blacklist rivafb

  blacklist rivatv

  blacklist nvidiafb

c) 保存退出，并执行以下命令使之生效：sudo update-initramfs -u
```

3. 重启，并在登录后切换到字符界面（按`Ctrl+Alt+Fx`，其中x为1～6）。注意在字符界面下中文极有可能显示不出来，这就是前文强烈建议驱动文件的路径不要带中文的原因。

4. 停掉图形桌面服务：

```
sudo service lightdm stop
```

5. 安装驱动：

```
cd ~

head -1 NVIDIA-Linux-x86_64-390.77.run # 看Shell脚本解析器是bash还是sh，后面会用到

sudo sh NVIDIA-Linux-x86_64-390.77.run # 用bash还是sh，根据上面看到的脚本解析器而定
```

## 验证驱动

重启，并运行以下命令进行验证：

```
nvidia-smi
```

若能显示类似以下内容，说明安装成功：

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.77                 Driver Version: 390.77                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  TITAN Xp            Off  | 00000000:01:00.0  On |                  N/A |
| 27%   43C    P8    15W / 250W |    857MiB / 12180MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0      1003      G   /usr/lib/xorg/Xorg                           692MiB |
|    0      1757      G   compiz                                       162MiB |
+-----------------------------------------------------------------------------+
```

若是出现类似以下的报错，说明已扑街：

```
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver. Make sure that the latest NVIDIA driver is installed and running.
```

## 参考

[https://blog.csdn.net/u012442845/article/details/78855573/](https://blog.csdn.net/u012442845/article/details/78855573/)

[https://blog.csdn.net/javahaoshuang3394/article/details/76425009](https://blog.csdn.net/javahaoshuang3394/article/details/76425009)

[https://blog.csdn.net/stories_untold/article/details/78521925](https://blog.csdn.net/stories_untold/article/details/78521925)

## 后记

其它型号的显卡也可参考本文，并根据具体情况进行必要的变通。

