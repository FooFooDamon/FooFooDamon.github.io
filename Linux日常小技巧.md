<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# Linux日常小技巧

注：如无特殊说明，文中所用命令均运行于`Ubuntu`。

* 按`Ctrl`+`H`可切换`Nautilus`（`Ubuntu`的默认文件管理器）对于隐藏文件（夹）的显示状态。

* 测试网口带宽（部分参数值仅作举例）：
    * 服务侧执行：`iperf3 -s`
    * 客户端执行：`iperf3 -c 192.168.111.111 -i 1 -w 64k -t 60`

* 检测`储存卡`或`U盘`是否是`扩容`卡盘：
    * 安装检测程序：`sudo apt install f3`
    * 检测：`sudo f3probe --time-ops /dev/sdX`，其中，`X`为a、b、c、……，下同
    * 还原真实容量：`sudo f3fix --last-sec=NNNNNNNN /dev/sdX`，其中，`NNNNNNNN`为最后一块扇区号，
    上一条命令的检测结果会显示该值甚至整条还原命令。

* 检测及修复磁盘（包括储存卡）坏块操作举例：
    ````
    $ sudo fdisk -l
    $ sudo badblocks -v /dev/sda10 > badsectors.txt
    $ sudo fsck -l badsectors.txt /dev/sda10
    ````
    摘自：https://www.lxlinux.net/12104.html

* 磁盘分区操作：
    * 列举：`sudo fdisk -l /dev/sdX`
    * 交互式操作：`sudo fdisk /dev/sdX`
        * 获取帮助：输入`m`。
        * 添加新分区：输入`n`，按指示操作。
        * 删除分区：输入`d`，按指示操作。
        * 保存并退出会话：输入`w`。
        * 其余：略。

* 格式化磁盘：`sudo mkfs.yyyy /dev/sdXn`，其中，`yyyy`为`ntfs`、`ext4`等，
`X`为a、b、c等，`n`为1、2、3等。

* 温度监测：
    * 安装监控程序：`sudo apt install lm-sensors`
    * 侦测硬件情况：`sudo sensors-detect`
    * 日常使用：`sensors -A`
    * 扩展：直接读取`/sys/class/hwmon`目录内的参数亦可，以后再补充。

* `SSH`显示远程主机的图形界面：
    * 远程主机若安装的是不带图形界面（又叫`headless`）的操作系统，
    可先安装一个轻量级的`X`窗口服务器（可能需要重启系统才能生效）：`sudo apt install xorg`
    * 在远程主机安装用于测试的`GUI`小程序：`sudo apt install x11-apps`
    * 远程主机开启`X11`转发功能，具体做法是在`/etc/ssh/sshd_config`增加或修改配置项
    （可能需要重启`sshd`服务才能生效）：`X11Forwarding yes`
    * 本地主机运行`ssh`命令时需要加上`-X`选项，即：`ssh -X xxx@xxx.xxx.xxx.xxx`
    * 测试，在远程主机运行：`xclock`

* 使用`apt`安装软件时保留安装包：
    ````
    $ echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' | sudo tee /etc/apt/apt.conf.d/10apt-keep-downloads
    ````

* 挂载磁盘镜像（通常命名带`.img`后缀）：
    ````
    $ sudo apt install kpartx
    $
    $ sudo kpartx -av demo.img # 文件名及运行结果仅用于举例
    add map loop5p1 (252:0): 0 2097152 linear 7:5 61440
    add map loop5p2 (252:1): 0 13823967 linear 7:5 2158592
    $
    $ sudo mount /dev/mapper/loop5p1 /path/to/demo/dir1 # 可选步骤，只在未能自动挂载时才需要手动执行此操作
    $ sudo mount /dev/mapper/loop5p2 /path/to/demo/dir2 # 同上
    $
    $ sudo umount /path/to/demo/dir1 # 先卸载
    $ sudo umount /path/to/demo/dir2
    $
    $ sudo kpartx -dv demo.img # 再删除分区映射信息
    ````

* 将网页保存成`PDF`文档：
    * 安装所需程序：`sudo apt install wkhtmltopdf`
    * 转换在线网页：
        * 示例：`wkhtmltopdf 'http://www.xxx.com/xxx.html' xxx.pdf`
        * 注意事项：对于某些动了手脚干扰页面排版（如`*CDN`）或要求登录/付费才能打印的网页（如`3*0doc`），
        可加上`--disable-javascript`选项来尝试解决。
    * 转换离线网页：待解决，会在读取文件时报`Frame load interrupted by policy change`的错误，
    即使加上`--enable-local-file-access`也不行。

* 观察软/硬中断的发生情况：
    * 应用场景举例：当发现`ksoftirqd/0`或其它与软/硬中断相关的内核线程`CPU`占用率过高时。
    * 命令（需开多个终端窗口分别运行）：
        ````
        $ top # 然后依次按Shift+p、数字1键，并留意hi（硬中断）和si（软中断）两列指标
        $
        $ watch -d cat /proc/softirqs # 监测软中断发生情况
        $
        $ watch -d awk "'{ if (\$2 > 10000) print \$0 }'" /proc/interrupts # 监测硬中断发生情况，由于行数太多，需过滤大多数数值较小的行
        ````

* 解决`Ubuntu` `22.04`连接安卓手机时出现的`the name 1.2721 was not provided by any .service files`报错：
    * 解决方案：在终端执行`nautilus -q`再重新连接手机即可。
    * 原因分析：可能是上一次数据传输过程中，因为线缆松动等原因导致状态异常，
    重启电脑里的文件管理器（`Ubuntu`的`GNOME`桌面环境默认文件管理器是`Nautilus`）即可。

