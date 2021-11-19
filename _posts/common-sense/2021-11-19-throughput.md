---
layout: post
title: 关于吞吐量与速度的常识
cover-img: ["/assets/img/穿越土星环.jpg"]
tags: [common-sense, throughput]
lang: zh
---

# 常见接口协议速度
---

## HDMI
* 2.1 标准: 48Gbps
* 2.0 标准: 18Gbps
![img_1.png](img_1.png)


## SATA
*

## PCIE
*

## USB
* 2.0
* 3.0

## 雷电



# 例子
---
- 单个像素点占用空间
  - 黑白二值图像, 一个像素只需要1bit
  - 256种状态的灰度图像, 需要8bit, 成为"真彩色", 称为色深(Color Depth), 色深的位数越高，越能细腻诠释色彩的色阶变化。 
  - 由于屏幕单个色块是由RGB组成, 因此 8*3=24bit
  - 因此色彩数量: 2^24=1670万色
![img.png](img.png)

选购显示设备时，会经常看见6bit、8bit、10bit等. 
当前(2021年)主流的是8bit

---
以4K清晰度, FPS 120Hz, 游戏为例, 需要传输的带宽是怎样的?
  - 4096*2160*24*120 = 25,480,396,800 bps =23.73 Gbps 即HDMI 2.0不够

