---
layout: post
title: "修复Win10下Synaptics触摸板双指触击无法打开右键菜单的问题"
subtitle: "通过修改注册表（转载）"
author: "Slc"
#header-img: "img/post-bg-dreamer.jpg"
mathjax: ture
header-mask: 0.4
tags:
  - 技术类
---
## 用于解决触控板右键失灵，或mac电脑触控板无右键问题

从Win8.1开始，Synaptics触摸板驱动的键值就不能正确设置，使得双指触击失效，无法打开右键菜单。

### 解决方法 
1.打开注册表；
2.搜索“2FingerTapAction”，
或直接定位到以下两个路径：
HKEY_LOCAL_MACHINE\SOFTWARE\Synaptics\SynTP\Win8
HKEY_CURRENT_USER\Software\Synaptics\SynTP\TouchPadSMB2c

3.把两个路径中“2FingerTapAction”的键值由0改为2，然后重启。


补充：在Win10下，没有以上两项。

解决方法
定位到以下路径：
HKEY_CURRENT_USER\Software\Synaptics\SynTP\TouchPadSMB2cTM2334
或
HKEY_CURRENT_USER\Software\Synaptics\SynTP\TouchPadSMB2cTM2336
手动新建“2FingerTapAction”项，类型为REG_DWORD，值为2。

 

引用地址：http://bbs.pcbeta.com/viewthread-1643794-1-1.html

转载于:https://www.cnblogs.com/live41/p/7577534.html

