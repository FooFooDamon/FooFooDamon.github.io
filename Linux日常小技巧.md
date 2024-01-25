<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# Linux日常小技巧

* 按`Ctrl`+`H`可切换`Nautilus`（`Ubuntu`的默认文件管理器）对于隐藏文件（夹）的显示状态。

* 测试网口带宽（部分参数值仅作举例）：
    * 服务侧执行：`iperf3 -s`
    * 客户端执行：`iperf3 -c 192.168.111.111 -i 1 -w 64k -t 60`

* 检测`储存卡`或`U盘`是否是`扩容`卡盘：
    * 安装检测程序（以Ubuntu为例）：`sudo apt install f3`
    * 检测：`sudo f3probe --time-ops /dev/sdX`，其中，`X`为a、b、c、……，下同
    * 还原真实容量：`sudo f3fix --last-sec=NNNNNNNN /dev/sdX`，其中，`NNNNNNNN`为最后一块扇区号，
    上一条命令的检测结果会显示该值甚至整条还原命令。

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

