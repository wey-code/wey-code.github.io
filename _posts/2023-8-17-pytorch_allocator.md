---
layout: post
title: "Pytorch的显存管理"
subtitle: "Pytorch allocator"
author: "Slc"
header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - Pytorch
---

# Pytorch的显存管理

Pytorch和设备强相关的库是C10，其包括了设备的显存管理等等。整体而言，Pytorch的显存管理为段式管理，但是这样容易带来内存碎片。具体展现出来的效果就是，明明剩余内存还够用，但是已经无法申请新的tensor，从而报OOM。如果不采用cache机制，时间又会耗费在频繁地申请和释放显存中。

整体而言，Pytorch的对显存采用隐式和显式两种方式同步进行管理，隐式管理指采用一个链表来

因此Pytorch的显存管理主要分为五个步骤：

1. 
