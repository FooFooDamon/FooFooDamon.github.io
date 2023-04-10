<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<base target="_blank" />

# Ubuntu 16.04及18.04安装及配置nvidia-docker

## 先安装`docker-ce`（若已安装则忽略）

````
sudo apt update

sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo apt-key fingerprint 0EBFCD88

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt update

sudo apt-get install docker-ce docker-ce-cli containerd.io
````

参考材料：

https://docs.docker.com/install/linux/docker-ce/ubuntu/

## 再加壳：安装`nvidia-docker`

```
# Add the package repositories
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
	sudo apt-key add -

distribution=$(. /etc/os-release;echo $ID$VERSION_ID)

curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
	sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update

# Install nvidia-docker2 and reload the Docker daemon configuration
sudo apt-get install -y nvidia-docker2

sudo pkill -SIGHUP dockerd

# Test nvidia-smi with the latest official CUDA image
sudo docker run --runtime=nvidia --rm nvidia/cuda:9.0-base nvidia-smi
```

以上是从官方网页摘抄并经过实测的安装过程，适用于首次安装的机子。若已有旧版本的docker，
则还需要额外的一些操作，详情见`官方GitHub`：https://github.com/NVIDIA/nvidia-docker

至于GPU镜像，直接下载NVIDIA提供的镜像即可，不必自己下载纯净版的操作系统镜像再安装
显卡驱动和CUDA等，非常麻烦！

至此，nvidia-docker安装完毕。以下是docker常见操作，无兴趣者可忽略。

## 添加更快的安装源（可选）

可以添加阿里云的apt源：

```
$ curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

$ sudo add-apt-repository \
     "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
     $(lsb_release -cs) \
     stable"
```

由于nvidia-docker是在docker基础上进一步的封装，因此在安装过程中有部分基础软件是从docker
的源下载的。对于国内用户，直接使用docker官方的源，可能会出现龟速，所以改用国内的一些第三
方源会有改善，例如以上的阿里云的源。使用第三方源之后，官方源可以清理掉，方法是将
`/etc/apt/sources.list`里相关的行（若有）注释掉，或者将`/etc/apt/sources.list.d/docker.list`
（文件名可能不同，取决于你添加时的命令）移至别处（不必删除，因为日后还想恢复的话会比较容易），
然后再执行`sudo apt-get update`更新即可。

## 配置无需敲sudo即能使用docker命令

1. 创建一个docker组

```
sudo groupadd docker
```

2. 将当前用户加入docker组

```
sudo usermod -aG docker $USER
```

3. 重新登录系统

4. 任意用一条docker命令来验证，例如：

```
docker images
```

## 修改镜像存放位置以及一些常见配置（可选）

1. 修改docker服务配置文件：

```
cd /etc/systemd/system/multi-user.target.wants

sudo vim docker.service

在ExecStart一行添加（或修改）一个选项（的值）：--graph=/Your/directory

```

2. 重启docker服务：

```
sudo systemctl daemon-reload

sudo systemctl restart docker
```

3. 查看更改后的位置：

```
docker info | grep "Root Dir"
```

**注**：在**docker程序更新时，以上配置文件可能会被还原**，为避免此问题，
可将以上选项值配置在`/etc/docker/daemon.json`（文件不存在则新建），
再重启docker服务。镜像存放位置及一些常见的配置项如下（双斜线后为注释，
实际使用时要删除）：

```
"data-root": "/Your/directory" // 镜像存放位置
"storage-driver": "overlay2" // 存储驱动类型，根据实际需求而定
"live-restore": true // docker守护进程（服务）崩溃或重启时，容器保持运行
"insecure-registries": [ "www.yourdomain.com" ] // 通常（临时）用于内部非安全仓库，使用该配置项后可以不用配置安全证书
```

## 拉取想要的镜像

```
# 格式：docker pull <镜像主名称>[:<标签>]
# 不带标签则默认拉取最新（latest）版本的镜像
# 例如：
docker pull ubuntu:16.04
```

## 创建一个新容器并配置中文环境

```
[foo@foo-pc ~]$ docker run -v /etc/localtime:/etc/localtime:ro --ulimit core=-1 -t -i --runtime=nvidia ubuntu:16.04 /bin/bash
root@6f7fac1bfd80:/#
root@6f7fac1bfd80:/# sed -i "/archive\.ubuntu/s//cn.archive.ubuntu/g" /etc/apt/sources.list
root@6f7fac1bfd80:/#
root@6f7fac1bfd80:/# apt update
root@6f7fac1bfd80:/#
root@6f7fac1bfd80:/# apt install language-pack-zh-hans language-pack-zh-hant
root@6f7fac1bfd80:/#
root@6f7fac1bfd80:/# echo "export LANG=zh_CN.UTF-8" >> /root/.bashrc
root@6f7fac1bfd80:/#
root@6f7fac1bfd80:/# exit
exit
[foo@foo-pc ~]$
```

其中：

* `-v /etc/localtime:/etc/localtime:ro`表示将本地时区文件挂载进容器，以保持与宿主机时区同步，
`ro`表示只读（read-only）；`-v`也可用于挂载目录。

* `--ulimit core=-1`能令docker在程序core dumped时能生成core文件且不限制其大小。

* `-t`选项表示分配一个虚拟终端（terminal）。

* `-i`表示交互式（interactive），不加则无法进行交互式操作，再按`Ctrl + C`可回到宿主机Shell环境，
但该容器还在运行，用`attach`或`exec`可再次进入容器，不过前者仍然无法进行交互式操作。

* `--runtime=nvidia`表示让容器在运行期间使用NVIDIA的设施（即显卡驱动等），若不需要用到则可以不加。

* `sed -i ...`一行的作用是将部分软件源替换成国内源以加快`apt`下载速度。

## 查看、重命名并重新启动刚才的容器

```
[foo@foo-pc ~]$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
6f7fac1bfd80        ubuntu:16.04        "/bin/bash"         5 minutes ago       Exited (0) 9 seconds ago                       jovial_cocks
[foo@foo-pc ~]$
[foo@foo-pc ~]$ docker rename jovial_cocks nvidia_ubuntu_16.04
[foo@foo-pc ~]$
[foo@foo-pc ~]$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
6f7fac1bfd80        ubuntu:16.04        "/bin/bash"         7 minutes ago       Exited (0) 2 minutes ago                       nvidia_ubuntu_16.04
[foo@foo-pc ~]$
[foo@foo-pc ~]$ docker start -i nvidia_ubuntu_16.04
root@6f7fac1bfd80:/#
```

## 退出容器并令其后台运行，然后再次进入的一种方法

不要用exit命令退出，而是依次按`Ctrl+P`和`Ctrl+Q`。再次进入则用attach命令，例如`docker attach nvidia_ubuntu_16.04`。

## 将修改过的容器保存成一个新镜像

```
[foo@foo-pc ~]$ docker commit 6f7fac1bfd80 nvidia_ubuntu:16.04
sha256:e40a5a628d57c366f73b296b7351a976d353911e1e35d22d96fd43225a4ade8c
[foo@foo-pc ~]$ 
[foo@foo-pc ~]$ 
[foo@foo-pc ~]$ docker images 
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nvidia_ubuntu       16.04               e40a5a628d57        14 seconds ago      3.24GB
```

## 将镜像备份为一个文件

````
$ docker save -o nvidia_ubuntu_16.04.docker.img.tar nvidia_ubuntu:16.04
或：
$ docker save > nvidia_ubuntu_16.04.docker.img.tar nvidia_ubuntu:16.04
````

## 从备份文件导出镜像到另一台机器

````
$ docker load -i nvidia_ubuntu_16.04.docker.img.tar
或：
$ docker load < nvidia_ubuntu_16.04.docker.img.tar
````

## 安装SSH服务以便能多窗口登录（仅推荐在开发环境上配置）

````
# 先使用passwd命令来设置docker容器内的用户密码（如果还没有密码）
# 然后执行以下操作：

apt install -y openssh-server

# 再打开SSH服务配置文件：vim /etc/ssh/sshd_config
# 增加或修改以下配置项（去掉前面的井号）：
# PermitRootLogin yes
# UsePAM no

# 最后启动SSH服务，若已运行则使用restart
service ssh start

````

2019/03/22更新：若仅为了能多窗口登录容器，可不必安装SSH，
使用`docker exec -ti nvidia_ubuntu_16.04 bash`即可。不过，某些场景下还是需要用到SSH，
例如docker容器在远程机器，安装上SSH服务后就可以直接连接容器，而就不必先登录远程宿主
机，再执行`docker exec`，多做一步。

## 入门参考材料

<a href="references/Docker——入门实战 - 等一杯咖啡的博客 - CSDN博客.pdf">Docker——入门实战 - 等一杯咖啡的博客</a>

