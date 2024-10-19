<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# Linux日常小技巧

注：如无特殊说明，文中所用命令均运行于`Ubuntu`。

* 按`Ctrl`+`H`可切换`Nautilus`（`Ubuntu`的默认文件管理器）对于隐藏文件（夹）的显示状态。

* 同步主机时间：
    * 使用`NTP`（普通精度）：
        * 服务器：
            ```
            $ sudo apt install ntp
            $ systemctl status ntp
            $ sudo systemctl start ntp
            ```
        * 客户端：
            ```
            $ sudo apt install ntpdate
            $ sudo ntpdate 192.168.1.2 # 先手动同步一次（注意此处填上实际的服务器地址，下同）
            $ # 然后按前面方法安装ntpd（以便后续自动同步），并在/etc/ntp.conf添加一行：pool 192.168.1.2
            $ # 最后执行：sudo systemctl restart ntp && ntpq -p
            ```
        * 注：`ntpdate`对系统时间的校正是**跳变**的，这对于依赖连续时钟的应用程序是个很严重的问题，
        所以使用时要多加小心！而`ntpd`则是有一套特定的算法来一点一点地微调时间，是**渐变**的，
        会安全得多，更加推荐使用。
    * 使用`PTP`（高精度，但需要网卡硬件支持）：
        * 查看网卡硬件是否支持`PTP`（`eth0`仅用于举例，下同）：`ethtool -T eth0`
        * 安装`ptp4l`：`sudo apt install linuxptp`
        * 主节点设备：`sudo ptp4l -i eth0 -H`
        * 从节点设备：`sudo ptp4l -i eth0 -H -s`

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

* 删除`PDF`文档的部分页：`pdftk input.pdf cat 1-2 4-7 9-end output output.pdf # 删除第3和第8页`

* 合并多个`PDF`文档：`pdftk xx1.pdf xx2.pdf xx3.pdf output xx.pdf`

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

* **进程**级别的网络流量监控：
    * 使用`nethogs`：
        * 安装：`sudo apt install nethogs`
        * 使用示例：`sudo nethogs eth0`
        * 更多命令行选项及交互式指令详见：`man nethogs`

* 禁用`Ubuntu`的**无人值守升级程序**：
    * 原因：对于通过手机热点上网的用户，可节省流量资费。
    * 停止：`sudo systemctl stop unattended-upgrades.service`
    * 禁用：`sudo systemctl disable unattended-upgrades.service`
    * 注：禁用之后要定期手动更新系统，以确保安全。

* 禁用`snap`服务：
    * 理由与前面`无人值守升级程序`的类似，操作命令则是：
        ```
        $ sudo systemctl stop snapd.socket # 注：若直接停止*.service的话，仍会自动被这个*.socket拉起
        ```

* 视频转`GIF`动图：
    * 可使用`FFmpeg`，例如：`ffmpeg -i xx.mp4 -vf "fps=5,scale=800:-1:flags=lanczos" -f gif xx.gif`
        * `-i xx.mp4`：指定输入的视频文件（路径）。
        * `-f gif`：指定输出格式是`GIF`。最后的`xx.gif`则是输出的图片文件（路径）。
        * `-vf "fps=5,scale=800:-1:flags=lanczos"`：`-vf`选项指定视频过滤器参数，如下：
            * `fps=5`：将帧率设置为每秒`5`帧，更常用的值是`15`或`30`，对流畅度和文件体积有明显影响，
            可根据需要指定。
            * `scale=800:-1:flags=lanczos`：将视频缩放到宽度`800`像素，同时保持纵横比，
            并使用`Lanczos`插值算法来平滑缩放。
            * 若需指定其他选项，可在命令行输入`man ffmpeg-filters`，
            并使用`VIDEO FILTERS`或更具体的关键词来搜索相应的内容。

