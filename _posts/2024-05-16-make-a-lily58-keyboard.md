---
layout: post
title: Lily58 键盘的制作
date: 2024-05-16 10:56 +0800
image:
  path: /2024/05/16/9uL2kfGRJVeiWKr.jpg
  alt: My Lily58 Keyboard
---
我总共是做了两套键盘，第一套是没有热插拔的青轴，第二套是支持热插拔的红轴。

之所这样做，是因为第一套我是抱着翻车的态度去做的，而且我手里正好有一个Cherry的青轴键盘，我把轴拆了下来，翻车的代价也不是很高。

最后的结果反倒是出人意料的顺利。所以我又做了第二套也就是我现在用的这个。 

顺利的完成离不开这两个重要的因素:

学校课程的学习，林老师幽默的教学方式和卓越的教学技巧让我们受益颇多。

嘉立创的下单助手，它足够专业和可靠能给你带来足够完成这件事的信心。

## 介绍
[Lily58](https://github.com/kata0510/Lily58)是一个开源的分体式键盘，它包括键盘PCB和一个堆叠式外壳（我感觉算是外壳）。提供的[构建指南](https://github.com/kata0510/Lily58/blob/master/Pro/Doc/buildguide_en.md)包括所需零件和焊接步骤都非常详细。

[QMK Firmware](https://github.com/qmk/qmk_firmware)是一个开源键盘固件。我是用这个固件驱动我的这个键盘，也可以使用[ZMK](https://github.com/zmkfirmware/zmk)。

他们的区别在[这里](https://zmk.dev/docs#features)


## 零件
### PCB和元器件
BOM在[构建指南](https://github.com/kata0510/Lily58/blob/master/Pro/Doc/buildguide_en.md)里都有，基本上PCB打样和元器件我都是在 __立创商城__ 完成的。我没有选择SMT，不会弄，焊接的也不多。凯华的热插拔的底座是从凯华官方店铺买的。
![第一套的一些元器件](https://s2.loli.net/2024/05/16/9Ga3MorAzs5Ht2e.jpg){: .w-75}
_第一套的一些元器件，第二套没拍照_

## 外壳
除了仓库里的case，也有其他开发者开发的[外壳](https://github.com/BoardSodie/Lily58-Acrylic-Case)。我是在拼多多上的一家店下单制作的，很方便。

## 构建环境
[构建环境](https://docs.qmk.fm/#/newbs_getting_started)

### Win
下载[QMK MSYS](https://msys.qmk.fm/)构建环境

### MacOS
```shell
brew install qmk/qmk/qmk
```
## Build
[Build First](https://docs.qmk.fm/#/newbs_building_firmware)
```shell
qmk compile -kb <keyboard> -km <keymap>
```
### 文档
[QMK Docs](https://docs.qmk.fm)

{% include embed/bilibili.html id='BV1TX4y1j7v6' %}
