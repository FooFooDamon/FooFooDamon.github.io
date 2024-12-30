<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# MIPI摄像头内核驱动笔记

* 本文所用硬件是香橙派`5`开发板（主控芯片是`RK3588S`）和`OV13855`摄像头。

* 在开发板一端，功能模块大致能分成`PHY`、`CSI`主控、视频捕获（`Video Capture`，
简称`VICAP`）、图像信号处理器（`Image Signal Processor`，简称`ISP`）。

* 在摄像头一端，决定出图质量的重要部件有：镜头（`Lens`）、
图像处理芯片 （常称之为`DSP`，可包含`ISP`，但此`ISP`与板端`ISP`的侧重点不同）、
传感器（`Sensor`）。

## 1、架构

### 1.1 `CSI-2`协议层级（纵向）

英文名称 | 中文名称 | 备注
-- | -- | --
Application Layer | 应用层 | 图像处理算法，例如`3A`、裁剪、翻转等
Pixel/Byte Packing/Unpacking Layer | 像素/字节转换层 | 发送方：将不同格式的像素数据**打包**成**字节流**<br>接收方：将字节流**解包**得到**像素数据**
Low Level Protocol | 低级协议层 | 发送方：为字节流**加上**包头包尾<br>接收方：**拆掉**包头包尾还原出字节流
Lane Management Layer | 通道管理层 | 发送方：将字节流**拆分**到多个通道进行传输<br>接收方：将所有通道的字节流**合并和排序**
PHY Layer | 物理层 | 发送方：将字节流**序列化**成符合`D-PHY`（或`C-PHY`）规范的二进制电信号（含`SoT`、<br>`EoT`等）并传输<br>接收方：对二进制电信号进行**反序列化**还原出字节流

### 1.2 模块关系（横向）

````
+--------+           +---------------+            +------------------------+     IPI    +-------+
|        |           |               |     PPI    |                        | <--------> |       |
| Sensor | --------> |   {C,D}-PHY   | <--------> |  CSI2 Host Controller  |     or     | VICAP | <----+
|        |           |               |            |                        | <--------> |       |      |
+--------+           +---------------+            +------------------------+     IDI    +-------+      |
                                                                                                       |
                                                                                                       |
    +--------------------------------------------------------------------------------------------------+
    |
    |                                            +---------------------+
    |                                            |    MP (Main Path)   |
    |      +--------------------------+    +---> | +------+ +--------+ | ----------+
    |      |           ISP            |    |     | | Crop | | Resize | |           |
    +----> | +---------+ +----------+ | ---+     | +------+ +--------+ |           V
           | | In Crop | | Out Crop | |          +---------------------+    +------------+
           | +---------+ +----------+ | ---+     +---------------------+    | DDR Memory |
           +--------------------------+    |     |    SP (Self Path)   |    +------------+
                                           +---> | +------+ +--------+ |           ^
                                                 | | Crop | | Resize | |           |
                                                 | +------+ +--------+ | ----------+
                                                 +---------------------+
````

注：

* `PPI`：`PHY-Protocol Interface`，物理层协议接口。

* `IPI`：`Image Pixel Interface`，图像像素接口。

* `IDI`：`Image Data Interface`，图像数据接口。

* 从`Sensor`出来的数据的原始格式叫`Bayer Raw`（或`Raw Bayer`，平时简称为`Raw`）。
根据内部单色`像点`（不是处理过的图像的一个个像素）的排列方式，
可分为`RGGB`、`BGGR`、`GRBG`等。

* `ISP`能对在水平和垂直方向相邻的若干个`像点`进行一定的`插值`运算，
从而合成相同尺寸且具备红绿蓝色彩分量的`RGB`格式图像。不过，出于带宽占用的考虑，
通常不会直接输出`RGB`数据，而是输出`Raw`或`YUV`。

### 1.3 媒体节点拓扑结构

````
$ v4l2-ctl --list-devices
rkisp-statistics (platform: rkisp):
	/dev/video18
	/dev/video19

rkcif (platform:rkcif-mipi-lvds):
	/dev/video0
	/dev/video1
	/dev/video2
	/dev/video3
	/dev/video4
	/dev/video5
	/dev/video6
	/dev/video7
	/dev/video8
	/dev/video9
	/dev/video10
	/dev/media0

rkisp_mainpath (platform:rkisp0-vir0):
	/dev/video11
	/dev/video12
	/dev/video13
	/dev/video14
	/dev/video15
	/dev/video16
	/dev/video17
	/dev/media1

$ media-ctl -p -d /dev/media0
Media controller API version 6.1.43

Media device information
------------------------
driver          rkcif
model           rkcif-mipi-lvds
serial          
bus info        platform:rkcif-mipi-lvds
hw revision     0x0
driver version  6.1.43

Device topology
- entity 1: stream_cif_mipi_id0 (1 pad, 11 links)
            type Node subtype V4L flags 0
            device node name /dev/video0
	pad0: Sink
		<- "rockchip-mipi-csi2":1 [ENABLED]
		<- "rockchip-mipi-csi2":2 []
		<- "rockchip-mipi-csi2":3 []
		<- "rockchip-mipi-csi2":4 []
		<- "rockchip-mipi-csi2":5 []
		<- "rockchip-mipi-csi2":6 []
		<- "rockchip-mipi-csi2":7 []
		<- "rockchip-mipi-csi2":8 []
		<- "rockchip-mipi-csi2":9 []
		<- "rockchip-mipi-csi2":10 []
		<- "rockchip-mipi-csi2":11 []

- entity 5: stream_cif_mipi_id1 (1 pad, 11 links)
            type Node subtype V4L flags 0
            device node name /dev/video1
	pad0: Sink
		<- "rockchip-mipi-csi2":1 []
		<- "rockchip-mipi-csi2":2 [ENABLED]
		<- "rockchip-mipi-csi2":3 []
		<- "rockchip-mipi-csi2":4 []
		<- "rockchip-mipi-csi2":5 []
		<- "rockchip-mipi-csi2":6 []
		<- "rockchip-mipi-csi2":7 []
		<- "rockchip-mipi-csi2":8 []
		<- "rockchip-mipi-csi2":9 []
		<- "rockchip-mipi-csi2":10 []
		<- "rockchip-mipi-csi2":11 []

- entity 9: stream_cif_mipi_id2 (1 pad, 11 links)
            type Node subtype V4L flags 0
            device node name /dev/video2
	pad0: Sink
		<- "rockchip-mipi-csi2":1 []
		<- "rockchip-mipi-csi2":2 []
		<- "rockchip-mipi-csi2":3 [ENABLED]
		<- "rockchip-mipi-csi2":4 []
		<- "rockchip-mipi-csi2":5 []
		<- "rockchip-mipi-csi2":6 []
		<- "rockchip-mipi-csi2":7 []
		<- "rockchip-mipi-csi2":8 []
		<- "rockchip-mipi-csi2":9 []
		<- "rockchip-mipi-csi2":10 []
		<- "rockchip-mipi-csi2":11 []

- entity 13: stream_cif_mipi_id3 (1 pad, 11 links)
             type Node subtype V4L flags 0
             device node name /dev/video3
	pad0: Sink
		<- "rockchip-mipi-csi2":1 []
		<- "rockchip-mipi-csi2":2 []
		<- "rockchip-mipi-csi2":3 []
		<- "rockchip-mipi-csi2":4 [ENABLED]
		<- "rockchip-mipi-csi2":5 []
		<- "rockchip-mipi-csi2":6 []
		<- "rockchip-mipi-csi2":7 []
		<- "rockchip-mipi-csi2":8 []
		<- "rockchip-mipi-csi2":9 []
		<- "rockchip-mipi-csi2":10 []
		<- "rockchip-mipi-csi2":11 []

- entity 17: rkcif_scale_ch0 (1 pad, 11 links)
             type Node subtype V4L flags 0
             device node name /dev/video4
	pad0: Sink
		<- "rockchip-mipi-csi2":1 []
		<- "rockchip-mipi-csi2":2 []
		<- "rockchip-mipi-csi2":3 []
		<- "rockchip-mipi-csi2":4 []
		<- "rockchip-mipi-csi2":5 [ENABLED]
		<- "rockchip-mipi-csi2":6 []
		<- "rockchip-mipi-csi2":7 []
		<- "rockchip-mipi-csi2":8 []
		<- "rockchip-mipi-csi2":9 []
		<- "rockchip-mipi-csi2":10 []
		<- "rockchip-mipi-csi2":11 []

- entity 21: rkcif_scale_ch1 (1 pad, 11 links)
             type Node subtype V4L flags 0
             device node name /dev/video5
	pad0: Sink
		<- "rockchip-mipi-csi2":1 []
		<- "rockchip-mipi-csi2":2 []
		<- "rockchip-mipi-csi2":3 []
		<- "rockchip-mipi-csi2":4 []
		<- "rockchip-mipi-csi2":5 []
		<- "rockchip-mipi-csi2":6 [ENABLED]
		<- "rockchip-mipi-csi2":7 []
		<- "rockchip-mipi-csi2":8 []
		<- "rockchip-mipi-csi2":9 []
		<- "rockchip-mipi-csi2":10 []
		<- "rockchip-mipi-csi2":11 []

- entity 25: rkcif_scale_ch2 (1 pad, 11 links)
             type Node subtype V4L flags 0
             device node name /dev/video6
	pad0: Sink
		<- "rockchip-mipi-csi2":1 []
		<- "rockchip-mipi-csi2":2 []
		<- "rockchip-mipi-csi2":3 []
		<- "rockchip-mipi-csi2":4 []
		<- "rockchip-mipi-csi2":5 []
		<- "rockchip-mipi-csi2":6 []
		<- "rockchip-mipi-csi2":7 [ENABLED]
		<- "rockchip-mipi-csi2":8 []
		<- "rockchip-mipi-csi2":9 []
		<- "rockchip-mipi-csi2":10 []
		<- "rockchip-mipi-csi2":11 []

- entity 29: rkcif_scale_ch3 (1 pad, 11 links)
             type Node subtype V4L flags 0
             device node name /dev/video7
	pad0: Sink
		<- "rockchip-mipi-csi2":1 []
		<- "rockchip-mipi-csi2":2 []
		<- "rockchip-mipi-csi2":3 []
		<- "rockchip-mipi-csi2":4 []
		<- "rockchip-mipi-csi2":5 []
		<- "rockchip-mipi-csi2":6 []
		<- "rockchip-mipi-csi2":7 []
		<- "rockchip-mipi-csi2":8 [ENABLED]
		<- "rockchip-mipi-csi2":9 []
		<- "rockchip-mipi-csi2":10 []
		<- "rockchip-mipi-csi2":11 []

- entity 33: rkcif_tools_id0 (1 pad, 11 links)
             type Node subtype V4L flags 0
             device node name /dev/video8
	pad0: Sink
		<- "rockchip-mipi-csi2":1 []
		<- "rockchip-mipi-csi2":2 []
		<- "rockchip-mipi-csi2":3 []
		<- "rockchip-mipi-csi2":4 []
		<- "rockchip-mipi-csi2":5 []
		<- "rockchip-mipi-csi2":6 []
		<- "rockchip-mipi-csi2":7 []
		<- "rockchip-mipi-csi2":8 []
		<- "rockchip-mipi-csi2":9 [ENABLED]
		<- "rockchip-mipi-csi2":10 []
		<- "rockchip-mipi-csi2":11 []

- entity 37: rkcif_tools_id1 (1 pad, 11 links)
             type Node subtype V4L flags 0
             device node name /dev/video9
	pad0: Sink
		<- "rockchip-mipi-csi2":1 []
		<- "rockchip-mipi-csi2":2 []
		<- "rockchip-mipi-csi2":3 []
		<- "rockchip-mipi-csi2":4 []
		<- "rockchip-mipi-csi2":5 []
		<- "rockchip-mipi-csi2":6 []
		<- "rockchip-mipi-csi2":7 []
		<- "rockchip-mipi-csi2":8 []
		<- "rockchip-mipi-csi2":9 []
		<- "rockchip-mipi-csi2":10 [ENABLED]
		<- "rockchip-mipi-csi2":11 []

- entity 41: rkcif_tools_id2 (1 pad, 11 links)
             type Node subtype V4L flags 0
             device node name /dev/video10
	pad0: Sink
		<- "rockchip-mipi-csi2":1 []
		<- "rockchip-mipi-csi2":2 []
		<- "rockchip-mipi-csi2":3 []
		<- "rockchip-mipi-csi2":4 []
		<- "rockchip-mipi-csi2":5 []
		<- "rockchip-mipi-csi2":6 []
		<- "rockchip-mipi-csi2":7 []
		<- "rockchip-mipi-csi2":8 []
		<- "rockchip-mipi-csi2":9 []
		<- "rockchip-mipi-csi2":10 []
		<- "rockchip-mipi-csi2":11 [ENABLED]

- entity 45: rockchip-mipi-csi2 (12 pads, 122 links)
             type V4L2 subdev subtype Unknown flags 0
             device node name /dev/v4l-subdev0
	pad0: Sink
		[fmt:SBGGR10_1X10/4224x3136 field:none
		 crop.bounds:(0,0)/4224x3136
		 crop:(0,0)/4224x3136]
		<- "rockchip-csi2-dphy0":1 [ENABLED]
	pad1: Source
		-> "stream_cif_mipi_id0":0 [ENABLED]
		-> "stream_cif_mipi_id1":0 []
		-> "stream_cif_mipi_id2":0 []
		-> "stream_cif_mipi_id3":0 []
		-> "rkcif_scale_ch0":0 []
		-> "rkcif_scale_ch1":0 []
		-> "rkcif_scale_ch2":0 []
		-> "rkcif_scale_ch3":0 []
		-> "rkcif_tools_id0":0 []
		-> "rkcif_tools_id1":0 []
		-> "rkcif_tools_id2":0 []
	pad2: Source
		-> "stream_cif_mipi_id0":0 []
		-> "stream_cif_mipi_id1":0 [ENABLED]
		-> "stream_cif_mipi_id2":0 []
		-> "stream_cif_mipi_id3":0 []
		-> "rkcif_scale_ch0":0 []
		-> "rkcif_scale_ch1":0 []
		-> "rkcif_scale_ch2":0 []
		-> "rkcif_scale_ch3":0 []
		-> "rkcif_tools_id0":0 []
		-> "rkcif_tools_id1":0 []
		-> "rkcif_tools_id2":0 []
	pad3: Source
		-> "stream_cif_mipi_id0":0 []
		-> "stream_cif_mipi_id1":0 []
		-> "stream_cif_mipi_id2":0 [ENABLED]
		-> "stream_cif_mipi_id3":0 []
		-> "rkcif_scale_ch0":0 []
		-> "rkcif_scale_ch1":0 []
		-> "rkcif_scale_ch2":0 []
		-> "rkcif_scale_ch3":0 []
		-> "rkcif_tools_id0":0 []
		-> "rkcif_tools_id1":0 []
		-> "rkcif_tools_id2":0 []
	pad4: Source
		-> "stream_cif_mipi_id0":0 []
		-> "stream_cif_mipi_id1":0 []
		-> "stream_cif_mipi_id2":0 []
		-> "stream_cif_mipi_id3":0 [ENABLED]
		-> "rkcif_scale_ch0":0 []
		-> "rkcif_scale_ch1":0 []
		-> "rkcif_scale_ch2":0 []
		-> "rkcif_scale_ch3":0 []
		-> "rkcif_tools_id0":0 []
		-> "rkcif_tools_id1":0 []
		-> "rkcif_tools_id2":0 []
	pad5: Source
		-> "stream_cif_mipi_id0":0 []
		-> "stream_cif_mipi_id1":0 []
		-> "stream_cif_mipi_id2":0 []
		-> "stream_cif_mipi_id3":0 []
		-> "rkcif_scale_ch0":0 [ENABLED]
		-> "rkcif_scale_ch1":0 []
		-> "rkcif_scale_ch2":0 []
		-> "rkcif_scale_ch3":0 []
		-> "rkcif_tools_id0":0 []
		-> "rkcif_tools_id1":0 []
		-> "rkcif_tools_id2":0 []
	pad6: Source
		-> "stream_cif_mipi_id0":0 []
		-> "stream_cif_mipi_id1":0 []
		-> "stream_cif_mipi_id2":0 []
		-> "stream_cif_mipi_id3":0 []
		-> "rkcif_scale_ch0":0 []
		-> "rkcif_scale_ch1":0 [ENABLED]
		-> "rkcif_scale_ch2":0 []
		-> "rkcif_scale_ch3":0 []
		-> "rkcif_tools_id0":0 []
		-> "rkcif_tools_id1":0 []
		-> "rkcif_tools_id2":0 []
	pad7: Source
		-> "stream_cif_mipi_id0":0 []
		-> "stream_cif_mipi_id1":0 []
		-> "stream_cif_mipi_id2":0 []
		-> "stream_cif_mipi_id3":0 []
		-> "rkcif_scale_ch0":0 []
		-> "rkcif_scale_ch1":0 []
		-> "rkcif_scale_ch2":0 [ENABLED]
		-> "rkcif_scale_ch3":0 []
		-> "rkcif_tools_id0":0 []
		-> "rkcif_tools_id1":0 []
		-> "rkcif_tools_id2":0 []
	pad8: Source
		-> "stream_cif_mipi_id0":0 []
		-> "stream_cif_mipi_id1":0 []
		-> "stream_cif_mipi_id2":0 []
		-> "stream_cif_mipi_id3":0 []
		-> "rkcif_scale_ch0":0 []
		-> "rkcif_scale_ch1":0 []
		-> "rkcif_scale_ch2":0 []
		-> "rkcif_scale_ch3":0 [ENABLED]
		-> "rkcif_tools_id0":0 []
		-> "rkcif_tools_id1":0 []
		-> "rkcif_tools_id2":0 []
	pad9: Source
		-> "stream_cif_mipi_id0":0 []
		-> "stream_cif_mipi_id1":0 []
		-> "stream_cif_mipi_id2":0 []
		-> "stream_cif_mipi_id3":0 []
		-> "rkcif_scale_ch0":0 []
		-> "rkcif_scale_ch1":0 []
		-> "rkcif_scale_ch2":0 []
		-> "rkcif_scale_ch3":0 []
		-> "rkcif_tools_id0":0 [ENABLED]
		-> "rkcif_tools_id1":0 []
		-> "rkcif_tools_id2":0 []
	pad10: Source
		-> "stream_cif_mipi_id0":0 []
		-> "stream_cif_mipi_id1":0 []
		-> "stream_cif_mipi_id2":0 []
		-> "stream_cif_mipi_id3":0 []
		-> "rkcif_scale_ch0":0 []
		-> "rkcif_scale_ch1":0 []
		-> "rkcif_scale_ch2":0 []
		-> "rkcif_scale_ch3":0 []
		-> "rkcif_tools_id0":0 []
		-> "rkcif_tools_id1":0 [ENABLED]
		-> "rkcif_tools_id2":0 []
	pad11: Source
		-> "stream_cif_mipi_id0":0 []
		-> "stream_cif_mipi_id1":0 []
		-> "stream_cif_mipi_id2":0 []
		-> "stream_cif_mipi_id3":0 []
		-> "rkcif_scale_ch0":0 []
		-> "rkcif_scale_ch1":0 []
		-> "rkcif_scale_ch2":0 []
		-> "rkcif_scale_ch3":0 []
		-> "rkcif_tools_id0":0 []
		-> "rkcif_tools_id1":0 []
		-> "rkcif_tools_id2":0 [ENABLED]

- entity 58: rockchip-csi2-dphy0 (2 pads, 2 links)
             type V4L2 subdev subtype Unknown flags 0
             device node name /dev/v4l-subdev1
	pad0: Sink
		[fmt:SBGGR10_1X10/4224x3136@10000/300000 field:none
		 crop.bounds:(0,0)/4224x3136]
		<- "m01_b_ov13855@7-0036":0 [ENABLED]
	pad1: Source
		-> "rockchip-mipi-csi2":0 [ENABLED]

- entity 63: m01_b_ov13855@7-0036 (1 pad, 1 link)
             type V4L2 subdev subtype Sensor flags 0
             device node name /dev/v4l-subdev2
	pad0: Source
		[fmt:SBGGR10_1X10/4224x3136@10000/300000 field:none
		 crop.bounds:(0,0)/4224x3136]
		-> "rockchip-csi2-dphy0":0 [ENABLED]

$ media-ctl -p -d /dev/media1
Media controller API version 6.1.43

Media device information
------------------------
driver          rkisp0-vir0
model           rkisp0
serial          
bus info        platform:rkisp0-vir0
hw revision     0x0
driver version  6.1.43

Device topology
- entity 1: rkisp-isp-subdev (4 pads, 10 links)
            type V4L2 subdev subtype Unknown flags 0
            device node name /dev/v4l-subdev3
	pad0: Sink
		[fmt:SBGGR10_1X10/4224x3136 field:none
		 crop.bounds:(0,0)/4224x3136
		 crop:(0,0)/4224x3136]
		<- "rkisp_rawrd0_m":0 []
		<- "rkisp_rawrd2_s":0 []
		<- "rkisp_rawrd1_l":0 []
		<- "rkcif-mipi-lvds":0 [ENABLED]
	pad1: Sink
		<- "rkisp-input-params":0 [ENABLED]
	pad2: Source
		[fmt:YUYV8_2X8/4224x3136 field:none colorspace:smpte170m quantization:full-range
		 crop.bounds:(0,0)/4224x3136
		 crop:(0,0)/4224x3136]
		-> "rkisp_mainpath":0 [ENABLED]
		-> "rkisp_selfpath":0 [ENABLED]
		-> "rkisp_fbcpath":0 [ENABLED]
		-> "rkisp_iqtool":0 [ENABLED]
	pad3: Source
		-> "rkisp-statistics":0 [ENABLED]

- entity 6: rkisp_mainpath (1 pad, 1 link)
            type Node subtype V4L flags 0
            device node name /dev/video11
	pad0: Sink
		<- "rkisp-isp-subdev":2 [ENABLED]

- entity 12: rkisp_selfpath (1 pad, 1 link)
             type Node subtype V4L flags 0
             device node name /dev/video12
	pad0: Sink
		<- "rkisp-isp-subdev":2 [ENABLED]

- entity 18: rkisp_fbcpath (1 pad, 1 link)
             type Node subtype V4L flags 0
             device node name /dev/video13
	pad0: Sink
		<- "rkisp-isp-subdev":2 [ENABLED]

- entity 24: rkisp_iqtool (1 pad, 1 link)
             type Node subtype V4L flags 0
             device node name /dev/video14
	pad0: Sink
		<- "rkisp-isp-subdev":2 [ENABLED]

- entity 30: rkisp_rawrd0_m (1 pad, 1 link)
             type Node subtype V4L flags 0
             device node name /dev/video15
	pad0: Source
		-> "rkisp-isp-subdev":0 []

- entity 36: rkisp_rawrd2_s (1 pad, 1 link)
             type Node subtype V4L flags 0
             device node name /dev/video16
	pad0: Source
		-> "rkisp-isp-subdev":0 []

- entity 42: rkisp_rawrd1_l (1 pad, 1 link)
             type Node subtype V4L flags 0
             device node name /dev/video17
	pad0: Source
		-> "rkisp-isp-subdev":0 []

- entity 48: rkisp-statistics (1 pad, 1 link)
             type Node subtype V4L flags 0
             device node name /dev/video18
	pad0: Sink
		<- "rkisp-isp-subdev":3 [ENABLED]

- entity 54: rkisp-input-params (1 pad, 1 link)
             type Node subtype V4L flags 0
             device node name /dev/video19
	pad0: Source
		-> "rkisp-isp-subdev":1 [ENABLED]

- entity 60: rkcif-mipi-lvds (1 pad, 1 link)
             type V4L2 subdev subtype Unknown flags 0
             device node name /dev/v4l-subdev4
	pad0: Source
		[fmt:SBGGR10_1X10/4224x3136@10000/300000 field:none]
		-> "rkisp-isp-subdev":0 [ENABLED]
````

上述节点之中，`video`主要用于图像操作，对应的数据结构是`struct video_device`；
`v4l-subdev`主要用于控制`sensor`、`PHY`、`ISP`等，对应的数据结构是`struct v4l2_subdev`。

顺便给出若干常用调试命令示例：

````
$ # 列出设备的详细信息
$ v4l2-ctl --all -d /dev/video11
Driver Info:
	Driver name      : rkisp_v6
	Card type        : rkisp_mainpath
	Bus info         : platform:rkisp0-vir0
	Driver version   : 2.4.0
	Capabilities     : 0x84201000
		Video Capture Multiplanar
		Streaming
		Extended Pix Format
		Device Capabilities
	Device Caps      : 0x04201000
		Video Capture Multiplanar
		Streaming
		Extended Pix Format
Media Driver Info:
	Driver name      : rkisp0-vir0
	Model            : rkisp0
	Serial           :
	Bus info         : platform:rkisp0-vir0
	Media version    : 6.1.43
	Hardware revision: 0x00000000 (0)
	Driver version   : 6.1.43
Interface Info:
	ID               : 0x03000007
	Type             : V4L Video
Entity Info:
	ID               : 0x00000006 (6)
	Name             : rkisp_mainpath
	Function         : V4L2 I/O
	Pad 0x01000009   : 0: Sink
	  Link 0x0200000a: from remote pad 0x1000004 of entity 'rkisp-isp-subdev' (Unknown V4L2 Sub-Device): Data, Enabled
Priority: 2
Format Video Capture Multiplanar:
	Width/Height      : 640/480
	Pixel Format      : 'NM12' (Y/UV 4:2:0 (N-C))
	Field             : None
	Number of planes  : 2
	Flags             :
	Colorspace        : SMPTE 170M
	Transfer Function : Rec. 709
	YCbCr/HSV Encoding: ITU-R 601
	Quantization      : Full Range
	Plane 0           :
	   Bytes per Line : 640
	   Size Image     : 307200
	Plane 1           :
	   Bytes per Line : 640
	   Size Image     : 153600
Selection Video Capture: crop, Left 0, Top 0, Width 4224, Height 3136, Flags:
Selection Video Capture: crop_bounds, Left 0, Top 0, Width 4224, Height 3136, Flags:
Selection Video Output: crop, Left 0, Top 0, Width 4224, Height 3136, Flags:
Selection Video Output: crop_bounds, Left 0, Top 0, Width 4224, Height 3136, Flags:

Image Processing Controls

                     pixel_rate 0x009f0902 (int64)  : min=0 max=1000000000 step=1 default=1000000000 value=432000000 flags=read-only, volatile
$
$ # 查看支持的捕获格式
$ v4l2-ctl --list-formats-ext -d /dev/video11
ioctl: VIDIOC_ENUM_FMT
	Type: Video Capture Multiplanar

	[0]: 'UYVY' (UYVY 4:2:2)
		Size: Stepwise 32x32 - 4224x3136 with step 8/8
	[1]: 'NV16' (Y/UV 4:2:2)
		Size: Stepwise 32x32 - 4224x3136 with step 8/8
	[2]: 'NV61' (Y/VU 4:2:2)
		Size: Stepwise 32x32 - 4224x3136 with step 8/8
	[3]: 'NV21' (Y/VU 4:2:0)
		Size: Stepwise 32x32 - 4224x3136 with step 8/8
	[4]: 'NV12' (Y/UV 4:2:0)
		Size: Stepwise 32x32 - 4224x3136 with step 8/8
	[5]: 'NM21' (Y/VU 4:2:0 (N-C))
		Size: Stepwise 32x32 - 4224x3136 with step 8/8
	[6]: 'NM12' (Y/UV 4:2:0 (N-C))
		Size: Stepwise 32x32 - 4224x3136 with step 8/8
$
$ # 抓图
$ W=640; H=480; v4l2-ctl -d /dev/video11 \
    --stream-mmap=4 --stream-skip=60 --stream-count=1 \
    --set-fmt-video=width=${W},height=${H},pixelformat=NV12 \
    --stream-to=/tmp/${W}x${H}.nv12
$
$ # 显示保存后的YUV图像
$ W=640; H=480; mplayer /tmp/${W}x${H}.nv12 \
    -loop 0 -demuxer rawvideo \
    -rawvideo w=${W}:h=${H}:size=$((${W}*${H}*3/2)):format=NV12
$
$ # 抓视频流
$ W=640; H=480; gst-launch-1.0 v4l2src device=/dev/video11 io-mode=4 \
    ! video/x-raw,format=NV12,width=${W},height=${H},framerate=30/1 \
    ! xvimagesink -e
````

## 2、设备树

* `Documentation/devicetree/bindings/media/video-interfaces.txt`（或`video-interfaces.yaml`）：
    * `视频数据流水线`（`Video Data Pipelines`）由外部设备和内部设备组成：
        * `外`部设备：例如摄像头（重点是传感器部件，即`sensor`）之类，其控制总线可以是`I2C`、
        `SPI`、`UART`等。
        * `内`部设备：包括视频`直接内存访问`（即`DMA`）引擎、图像信号处理器（即`ISP`）等。
    * 内外设备均可通过设备树节点来描述：
        * `外`部设备：挂载在其控制总线（例如`I2C`）之下，作为该总线其中一个子节点。
        * `内`部设备：与`片上系统`（`SoC`）的其他内部模块的配置内容类似。
    * 所有视频设备的数据接口均通过`port`子节点来描述。若有多个端口，
    则每个`port`节点需显式写上地址，还可以在它们之上再套一个`ports`父节点（该节点可选，
    且仍是设备节点的子节点），而且设备节点或`ports`节点需要设置`#address-cells`、
    `#size-cells`和`reg`属性。格式举例：
        ````
        device {
            ...
            ports {
                #address-cells = <1>;
                #size-cells = <0>;

                port@0 {
                    ...
                    endpoint@0 { ... };
                    endpoint@1 { ... };
                };

                port@1 { ... };
            };
        };
        ````
    * 端口之下还有`端点`（`endpoint`）的概念，一个`endpoint`节点表示参与数据传输的逻辑实体：
        * 一个`port`节点可包含一个或多个`endpoint`节点。若有多个`endpoint`节点，
        则该`port`节点需要设置`#address-cells`、`#size-cells`和`reg`属性。
        * 两个相连的`endpoint`节点之间均通过属性名为`remote-endpoint`的句柄来指明对方。
        * 两个设备之间交换数据所需的配置信息记录在`endpoint`（而非`port`）节点里相应的属性项。
        * 在设备硬件支持的前提下，允许一个`port`节点下同时激活多个`endpoint`子节点，
        一个例子是把一个`16`位输入端口拆分成两条`ITU-R BT.656`规范的`8`位总线。
        * 常见属性项：
            * `slave-mode`：是否工作在`从`模式。若不指明，则默认为`false`，即`主`模式。
            当处于`从`模式时，由`主`设备（即`data sink`，`数据槽`）负责提供水平同步、
            垂直同步信号给`从`设备（即`data source`，`数据源`）。
            * `bus-type`：总线类型，其中`4`表示`MIPI CSI-2 D-PHY`，其余枚举值详见原文档。
            * `bus-width`：启用的数据总线宽度，仅适用于并行总线（即`DVP`），最大值为`64`。
            * `data-shift`：数据线偏移位移，需搭配`bus-width`使用，
            例如“`bus-width=<8>; data-shift=<2>;`”表示启用的数据线是`[9:2]`。
            * `hsync-active`、`vsync-active`、`data-active`、`data-enable-active`：
            水平同步信号、垂直同步信号、数据极性、数据线开关极性在激活状态下对应的电平，
            `0`为`低`电平，`1`为`高`电平。
            * `pclk-sample`：指定在像素时钟信号的上升沿（值为`1`时）或下降沿（值为`0`时）
            进行数据采样。
            * `data-lanes`：数据通道列表（仅适用于串行总线，即**非DVP**），数据类型是`uint32`数组，
            每个数组元素`值`（最小值为`0`或`1`，最大值为`8`）表示一个`物理`通道号，
            元素`位置`则表示`逻辑`通道号。若硬件（例如`D-PHY`）不支持通道重排序，
            则数组元素值应以`0`（无时钟通道）或`1`（时钟通道作为通道`0`）开始且单调递增，
            驱动代码往往也仅关心数组大小以确定通道数量，而不使用具体的元素值。
            * `clock-lanes`：时钟通道，究竟是整数还是数组暂且存疑，因其名称与释义有矛盾，
            实际中也很少使用，或直接设成“`clock-lanes = <0>;`”。
            * `clock-noncontinuous`：布尔类型，是否允许`MIPI CSI-2`主控使用非连续时钟信号。
            * `link-frequencies`：链路频率列表，数据类型是`uint64`数组，但对于`MIPI CSI-2`而言，
            实际上仅用一个数字来表示链路总线的实际频率（不是每条通道每个时钟的位速率）。
            * `lane-polarities`：通道极性列表（仅适用于串行总线，即**非DVP**），
            必须同时包括时钟通道和所有数据通道。每个元素取值为`0`（正常极性）或`1`（相反极性）。
            若不指定该属性，则驱动代码应默认为正常极性。
            此外，部分芯片型号的驱动直接采取硬编码的方式，不使用设备树配置。

* `Documentation/devicetree/bindings/media/rockchip-mipi-dphy.txt`（根据具体的`SoC`和开发板而定，
此处仅采用瑞芯微作为例子）：
    * 必需属性：`compatible`、`clocks`、`clock-names`
    * 可选属性：`reg`、`rockchip,grf`（对于`MIPI TX1RX1 D-PHY`是必需）
    * 额外要求：必须包含`2`个`端口`（`ports`），一个连摄像头`传感器`（`sensor`），
    另一个连`图像信号处理器`（`ISP`，`image signal processor`）。
    每个端口节点内需要包含**至少`1`个**`端点`（`endpoint`）子节点，
    以便描述各个视频设备的具体配置。通用的作用说明和格式要求详见前述的`video-interfaces.txt`，
    此处只需重点关注`remote-endpoint`、`data-lanes`、`bus-type`等属性项。
    另外，每个端口可包含多个端点（以便连接多个传感器），但某一时间点只能激活一个端点。

## 3、源代码

### 3.1 `V4L2`架构

可分为`4`个部分：

* 字符设备驱动：即`/dev/videoX`（`X` = 0, 1, 2, ...），围绕其有以下设计：
    * 向**上**为`虚拟文件系统`提供了统一的接口，应用程序可通过虚拟文件系统访问视频设备。
    * 向**下**为视频设备提供接口，同时管理所有视频设备：<---- 描述可能不太准确
        * **主**设备：`SoC`主控，负责**图像数据的传输**。
        * **从**设备：`Sensor`，一般为`I2C`接口，可**控制从设备采集图像的行为**，
        如图像的大小、图像的帧速率等。

* `V4L2`驱动核心：作用是提供一个标准框架和统一的接口函数供其他模块使用，
其源码位于`drivers/media/v4l2-core`目录。可重点关注以下源文件（以`6.1`为例，
之前和之后的版本可能会变化）：
    * `v4l2-async.c`：异步注册接口
    * `v4l2-dev.c`：字符设备接口
    * `v4l2-device.c`：`V4L2`子系统的入口，**主**设备，一个主设备可包含多个`子`设备
    * `v4l2-event.c`：事件管理
    * `v4l2-fwnode.c`：设备树节点，**重点关注操作`remote-endpoint`句柄的设备树函数**
    * `v4l2-ioctl.c`：`ioctl`的什么？？
    * `v4l2-subdev.c`：**子**设备，又叫从设备
    * `videobuf-core.c`：缓冲区管理

* 平台`V4L2`设备驱动：即板卡厂商写的`platform`驱动，例如注册`video_device`和`v4l2_device`。
以瑞芯微为例，其源码主要位于`drivers/media/platform/rockchip/{cif,isp,...}`、
`drivers/phy/rockchip`目录。

* 摄像头驱动：约等于`Sensor`驱动，主要是实现上下电流程、视频图像裁剪、
数据流开关、设备控制接口（即`ioctl`）、注册`v4l2_subdev`等。

### 3.2 `V4L2`重要数据结构

结构体大致可分为`4`类，摘要如下（为方便排版，部分成员顺序有调整）：

````
video_device：视频设备，会生成`/dev/videoX`节点，video_register_device()
    |-- ...
    |-- cdev：字符设备
    |-- v4l2_device：主设备（主要作用是管理子设备），v4l2_device_{register,unregister,disconnect}()
    |       |-- ...
    |       |-- struct list_head subdevs：链表项的数据类型为struct v4l2_subdev：
    |       |       v4l2_subdev：子设备，v4l2_subdev_init()、v4l2_device_{register,unregister}_subdev()
    |       |           |-- ...
    |       |           |-- const struct v4l2_subdev_ops *ops：v4l2_subdev_call()
    |       |           |-- void *dev_priv：子设备自己的私有数据，通常是一个具体设备指针，例如i2c_client，
    |       |           |       v4l2_set_subdevdata()。与此相反，i2c_client也可以保存子设备指针，
    |       |           |       使用i2c_{get,set}_clientdata()
    |       |           |-- void *host_priv：主设备的私有数据，v4l2_{get,set}_subdev_hostdata()
    |       |           `-- ...
    |       |-- ...
    |       |-- void (*notify)(struct v4l2_subdev *sd, unsigned int notification, void *arg)：
    |       |       v4l2_subdev_notify()
    |       `-- ...
    |-- ...
    |-- queue：见下面的vb2_queue结构体
    `-- ...

vb2_queue：数据流操作（上面则是控制流）的入口和核心
    |-- struct vb2_mem_ops *mem_ops：alloc、put、{get,detach}_dmabuf、get_userptr、mmap
    |-- struct vb2_buf_ops *buf_ops：verify_planes_array、fill_{user,vb2}_buffer、copy_timestamp
    |-- struct vb2_ops *ops：queue_setup、wait_{prepare,finish}、buf_{init,prepare,queue}、{start,stop}_streaming
    |-- struct vb2_buffer bufs[]：待确认：缓冲区既可以是用户态分配，也可以是内核分配？？
    |           |-- struct vb2_queue *vb2_queue
    |           |-- struct vb2_plane planes[]
    |           `-- ...
    |-- struct list_head queued_list：入队且待处理的缓冲项
    `-- struct list_head done_list：处理完毕变闲置的缓冲项

media_device：entity、pad、link等，待补充

controls：v4l2_ctrl_handler、v4l2_ctrl_ops等，待补充
````

### 3.3 工作流程

* `ioctl`接口。一般至少需要实现以下接口：
    * `VIDIOC_REQBUFS`：分配缓冲区
    * `VIDIOC_QUERYCAP`：查询驱动能力
    * `VIDIOC_{ENUM,S,G,TRY}_FMT`：枚举全部/设置/获取当前/验证视频捕获格式
    * `VIDIOC_CROPCAP`：查询裁剪能力
    * `VIDIOC_{S,G}_CROP`：设置/获取裁剪范围
    * `VIDIOC_{QBUF,DQBUF}`：将数据项入队/出队
    * `VIDIOC_STREAM{ON,OFF}`：启动/停止视频流

* `MIPI` `D-PHY`时序：
    * 传输模式：
        * `LP`：`Low Power`，低功耗模式，用于传输**控制信号**，速率范围为`(0, 10]Mbps`。
        * `HS`：`High Speed`，高速模式，用于传输**图像数据**，速率范围为`[80, 1000]Mbps`
        （注：速率上限在刷新，早已不止`1000Mbps`，视具体设备而定）。
    * 时钟通道：通过`LP11 -> LP01 -> LP00`的顺序进入`HS`模式，
    而且还会根据不同的时钟信号模式来决定在无图像数据的空闲时期是否切换到`LP`模式：
        * 连续时钟：不会切换，所以只有一次上述进入`HS`模式的状态序列。
        * 非连续时钟：会切换。
    * 数据通道：有`3`种操作模式：
        * `Esc`：`Escape`，`逃逸`模式，用于XXXX，状态时序为：
            * 进入：`LP11 -> LP10 -> LP00 -> LP01 -> LP00`
            * 退出：`LP10 -> LP11`
        * `HS`：前文已有介绍，又叫`Burst`（`迸发`）模式，用于XXXX，状态时序为：
            * 进入：`LP11 -> LP01 -> LP00 -> SoT(0001_1101)`
            * 退出：`EoT -> LP11`
        * `Ctrl`：`Control`，`控制`模式，用于XXXX，状态时序为：
            * 进入：`LP11 -> LP10 -> LP00 -> LP10 -> LP00`
            * 退出：`LP00 -> LP10 -> LP11`

* `PPI`接口描述：待补充

小提示：可在某个函数内（例如`vb2_core_dqbuf()`）调用`stack_dump()`来打印调用链。

### 3.4 计算公式

一帧的曝光行数也叫积分时间（`Integration Time`），单位：行。

未完待续……

## 4、图像格式

### 4.1 `RGB`格式

待补充……

### 4.2 `Bayer`格式

待补充……

### 4.3 `YUV`格式

* 名称释义：本质上是一种**色差**`分量`（`Component`）格式，`色差`是指`颜色值`与`亮度`之间的差值，
此格式有`3`个`分量`：
    * `Y`：`亮度`（`Luminance`或`Luma`），也叫`灰度值`或`灰阶值`。若只使用`Y`分量则得到灰度图，
    如此既方便兼容黑白电视，又极大减少数据量。
    * `U`、`V`：合称为`色度`（`Chrominance`或`Chroma`），用于描述`色彩`和`饱和度`（`Saturation`，
    饱和度越高则图像越鲜艳明显，越低则越灰暗柔和），
    但注意实际的格式定义**并非**`U`对应`色彩`且`V`对应`饱和度`（或反过来），
    而是根据数据压缩的需要、三元一次方程有确定解的最低要求以及人眼对三原色的不同敏感度，
    从原来的**蓝**（`Blue`）、**红**（`Red`）信号去掉亮度信息后分别得到**Cb**（或记作`Pb`）、
    **Cr**（或记作`Pr`）。
    * **注**：`Y`、`U`、`V`类似于数学上的`X`、`Y`、`Z`坐标轴记法，本身并无特殊之处，
    不过在实际场景中，还可代换成`YCbCr`（用于`隔`行扫描，此技术出现较早，
    所以直接用了`Component`的首字母`C`，而不是`Interlace`的首字母）或`YPbPr`（用于`逐`行扫描，
    `P`表示`Progressive`）。

* 理论基础：人眼对`亮度`较敏感，对`色度`较不敏感，因而将色度信息削减一部分（约减一半），
仍对实际的视觉体验影响不大。

* 与`RGB`格式的互转计算：详见后面章节。

* 采样格式：
    * 格式统一记为YUV`a:b:c`：`a`、`b`、`c`均指在`水平`方向上的`相对`采样率：
        * `a`：`Y`分量的个数。`a` === `4`，即`Y`分量一定是完全采样。
        * `b`：`U`分量的个数。`b`可取`0`（尽管在命名上未有体现）、`1`、`2`、`4`，
        且`a` >= `b` == `c`。
        * `c`：`V`分量的个数。`c`可取`0`、`1`、`2`、`4`，且`a` >= `b` == `c`。
    * YUV`4:4:4`：完全采样，一个像素占`3`字节。以`4x4`像素大小的图片为例（下同），
    示意图如下：
        ````
        YUV YUV | YUV YUV
        YUV YUV | YUV YUV
        -------   -------
        YUV YUV | YUV YUV
        YUV YUV | YUV YUV
        ````
    * YUV`4:2:2`：水平方向按`Y:UV`=`2:1`采样，任意一行之中`U`、`V`相间且与`Y`配对填满该行，
    垂直方向完全采样，所以每`2`个像素共享`1`份`UV`分量，最终平均一个像素占`2`字节：
        ````
        YU  YV  | YU  YV
        YU  YV  | YU  YV
        -------   -------
        YU  YV  | YU  YV
        YU  YV  | YU  YV
        ````
    * YUV`4:2:0`：更准确但冗长的命名应为`YUV420YUV402`，水平、垂直方向均是按`Y:UV`=`2:1`采样，
    任意一行之中只有相间的`U`或`V`（即对于`U`或`V`，要么是`2`比例，要么是`0`）且只占半行，
    所以每`4`个像素共享`1`份`UV`分量，最终平均一个像素占`1.5`字节：
        ````
        MPEG-1标准：
        YU  Y   | YU  Y
        Y   YV  | Y   YV
        -------   -------
        YU  Y   | YU  Y
        Y   YV  | Y   YV

        MPEG-2标准：
        YU  Y   | YU  Y
        YV  Y   | YV  Y
        -------   -------
        YU  Y   | YU  Y
        YV  Y   | YV  Y
        ````
    * YUV`4:1:1`：水平方向按`Y:UV`=`4:1`采样，任意一行之中`U`和`V`相间且累计只占半行，
    垂直方向完全采样，所以（水平方向上）每`4`个像素共享`1`份`UV`分量，
    最终平均一个像素占`1.5`字节：
        ````
        YU  Y   YV  Y
        ---------------
        YU  Y   YV  Y
        ---------------
        YU  Y   YV  Y
        ---------------
        YU  Y   YV  Y
        ````
    * 以上格式的像素还原示意图可看[这篇文章](https://developer.aliyun.com/article/1638599)
    （若失效则看此[备份文档](references/camera/手机广告常见的10bit是什么？YUV444、YUV422、YUV420、YUV411是什么？-阿里云开发者社区.pdf)）。

* 储存格式，可分为：
    * `紧缩`格式（`Packed Format`）：按**像素**次序存放，每个像素包含`Y`、`U`、`V`值，
    类似`RGB`格式。采用此格式的有：
        * YUV`4:4:4` Packed：
            ````
            YUV YUV YUV YUV
            YUV YUV YUV YUV
            YUV YUV YUV YUV
            YUV YUV YUV YUV
            ````
        * YUV`4:2:2`：
            * `YUYV`、`YVYU`、`UYVY`、`VYUY`，从名称就能直观知道存放方式，仅以`YUYV`举例：
                ````
                YU YV YU YV
                YU YV YU YV
                YU YV YU YV
                YU YV YU YV
                ````
    * `半平面`格式（`Semi-Planar Format`）：`Y`分量存放在一个**矩阵**（即平面），
    `U`和`V`交错存放在另一个平面，所以共有`2`个平面。采用此格式的有：
        * YUV`4:4:4`：
            * `NV24`：
                ````
                Y Y Y Y
                Y Y Y Y
                Y Y Y Y
                Y Y Y Y
                -------
                U V U V U V U V
                U V U V U V U V
                U V U V U V U V
                U V U V U V U V
                ````
            * `NV42`：与`NV24`类似，只是`U`、`V`分量顺序相反。
        * YUV`4:2:2`：
            * `NV16`：
                ````
                Y Y Y Y
                Y Y Y Y
                Y Y Y Y
                Y Y Y Y
                -------
                U V U V
                U V U V
                U V U V
                U V U V
                ````
            * `NV61`：与`NV16`类似，只是`U`、`V`分量顺序相反。
        * YUV`4:2:0`（又叫`YUV420SP`）：
            * `NV12`：
                ````
                Y Y Y Y
                Y Y Y Y
                Y Y Y Y
                Y Y Y Y
                -------
                U V U V
                U V U V
                ````
            * `NM12`：是以**非连续**（`Non-Contiguous`）内存空间来储存图像数据的`NV12`格式，
            是一种软件数据结构表现方式，而非一种图像新格式，多见于支持`多平面接口`
            （`Multi-Planar API`）的驱动和应用代码。
            * `NV21`（`Android`系统默认格式）：与`NV12`类似，只是`U`、`V`分量顺序相反。
            * `NM21`：与`NM12`类似，是`NV21`的一种软件数据结构表现形式。
    * `平面`格式（`Planar Format`）：将`Y`、`U`、`V`分别存放在不同的平面，
    所以共有`3`个平面。采用此格式的有：
        * YUV`4:4:4`：
            * `YU24`（又叫`I444`）：
                ````
                Y Y Y Y
                Y Y Y Y
                Y Y Y Y
                Y Y Y Y
                -------
                U U U U
                U U U U
                U U U U
                U U U U
                -------
                V V V V
                V V V V
                V V V V
                V V V V
                ````
            * `YV24`：与`YU24`类似，只是`U`、`V`平面顺序相反。
        * YUV`4:2:2`：
            * `YU16`（又叫`I422`）：
                ````
                Y Y Y Y
                Y Y Y Y
                Y Y Y Y
                Y Y Y Y
                -------
                U U
                U U
                U U
                U U
                -------
                V V
                V V
                V V
                V V
                ````
            * `YV16`：与`YU16`类似，只是`U`、`V`平面顺序相反。
        * YUV`4:2:0`（又叫`YUV420P`）：
            * `YU12`（又叫`I420`）：
                ````
                Y Y Y Y
                Y Y Y Y
                Y Y Y Y
                Y Y Y Y
                -------
                U U
                U U
                -------
                V V
                V V
                ````
            * `YV12`：与`YU12`类似，只是`U`、`V`平面顺序相反。

### 4.4 `Bayer`转`RGB`或`YUV`

待补充……

### 4.5 `RGB`与`YUV`互转（具体等式可能不准确，仅用于解释原理）

* `RGB`转`YUV`（`SDTV`标准，即`Standard Definition TeleVision`）：
    * `Y = 0.299 * R + 0.587 * G + 0.114 * B`
    * `Cb = 0.564 * (B - Y) = -0.169 * R - 0.331 * G + 0.500 * B`
    * `Cr = 0.713 * (R - Y) = 0.500 * R - 0.419 * G - 0.081 * B`

* `RGB`转`YUV`（`HDTV`标准，即`High Definition TeleVision`）：
    * `Y = 0.2126 * R + 0.7152 * G + 0.0722 * B`
    * `Pb = 0.5389 * (B - Y) = -0.1146 * R - 0.3854 * G + 0.5000 * B`
    * `Pr = 0.6350 * (R - Y) = 0.5000 * R - 0.4542 * G - 0.0458 * B`

* 代码实现：通常不推荐手写算法，因为效率可能不高，更推荐直接使用`OpenCV`或其他封装好的接口，
例如`NV12`转`BGR`可以这样写（以`640x480`大小的图像为例）：
    ````
    const size_t IMG_WIDTH = 640;
    const size_t IMG_HEIGHT = 480;
    const size_t IMG_SIZE = IMG_WIDTH * IMG_HEIGHT * 3 / 2;
    unsigned char nv12_buf[IMG_SIZE];
    cv::Mat nv12(IMG_HEIGHT * 3 / 2, IMG_WIDTH, CV_8UC1, nv12_buf);
    unsigned char bgr_buf[IMG_WIDTH * IMG_HEIGHT * 3];
    cv::Mat bgr(IMG_HEIGHT, IMG_WIDTH, CV_8UC3, bgr_buf);

    cv::cvtColor(nv12, bgr, cv::COLOR_YUV2BGR_NV12);
    ````

## X、参考材料

* `Understanding MIPI Interface`：[原始网页](https://zhuanlan.zhihu.com/p/100476927)及其[备份文档](references/camera/Understanding_MIPI_Interface.pdf)


