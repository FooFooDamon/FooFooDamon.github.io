<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

# Linux下通过NFS挂载远程目录

该文章是通过参考 https://www.linuxidc.com/Linux/2016-04/130504.htm 并结合亲身实践而写成。转载请注明本文及其参考材料的出处。

## 环境

本地机器和远程服务器均是Ubuntu 16.04，其余Linux发行版所用命令可能有所不同，其余操作系统本文不涉及。

## 服务器配置

1、安装NFS服务：

```
sudo apt-get install nfs-kernel-server nfs-common
```

2、修改服务器参数：

```
sudo vim /etc/exports
```

加上目标目录：

```
/home/foo/git *(insecure,rw,sync,no_root_squash)
```

注：该目录是NFS服务的根目录。星号表示允许所有网段访问，也可指定具体的IP。

3、刷新服务器配置：

```
sudo exportfs -r
```

4、检查配置是否已生效：

```
showmount -e
```

若生效，则会输出类似以下信息：

```
Export list for ubuntu:
/home/foo/git *
```

5、重启NFS服务：

```
/etc/init.d/nfs-kernel-server restart
```


## 本地机器配置

1、安装NFS应用包：

```
sudo apt-get install nfs-utils
```

如果找不到，则尝试安装nfs-common。

2、将远程目录挂载到本地，本地目录自行确定：

```
sudo mount -t nfs -o nolock,soft xxx.xxx.xxx.xxx:/home/foo/git /home/foo/local_git
```

其中，xxx.xxx.xxx.xxx指的是服务器的IP，`-o nolock,soft`则是防止服务端不可用时客户端卡住
的情况（待验证）。为方便起见，可将这条命令加入开机启动设置（例如/etc/rc.local）。

如果挂载出现权限错误，即类似：

```
mount.nfs: access denied by server while mounting xxx.xxx.xxx.xxx:/home/foo/git
```

可先尝试修改远程目录权限，赋予读、写和执行权限，即：

```
chmod -R 777 /home/foo/git
```

再刷新服务器配置。

若还是报权限错误，可在上述服务器配置文件加入insecure选项（前述内容已添加该选项），再刷新服务器配置。


配置完毕。更详细的说明可参考文章开头的网址。


## 若干注意事项

1. 远程目录及文件的权限问题。如前所述，设置不当可能导致无法访问或修改。

2. 电脑**关机前最好先卸载NFS目录**（用`umount`命令，为避免设备繁忙的报错，可增加`-l`选项，
要强制卸载还可增加`-f`选项），否则可能卡住关不了机。

