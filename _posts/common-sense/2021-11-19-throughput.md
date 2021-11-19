---
layout: post
title: 关于吞吐量与速度的常识
cover-img: ["/assets/img/穿越土星环.jpg"]
tags: [common-sense, throughput]
lang: zh
---

# 常见接口协议速度
---

HDMI
* 2.1 标准: 4GBps

SATA
*

PCIE
*

USB
* 2.0
* 3.0


# 例子
---
- 单个像素点占用空间
  - 黑白二值图像, 一个像素只需要1bit
  - 256种状态的灰度图像, 需要8bit
  - 由于屏幕是由RGB组成, 因此 8*3=24bit

以4K清晰度, FPS 120Hz, 游戏为例, 需要传输的带宽是怎样的?
  - 4096*2160*24*120 = 25,480,396,800 bps = 2.97 GBps

