---
layout: post
title: "mathtype使用"
subtitle: "通过修改注册表来激活"
author: "Slc"
#header-img: "img/post-bg-dreamer.jpg"
mathjax: ture
header-mask: 0.4
tags:
  - 技术类
---

试了N种Mathtype免费激活的方法，结果都不满意，下面这种方法可谓是拯救了我这个平民党（无需花钱，免费使用，哈哈哈哈）

话不多说，上方法：

Mathtype在电脑中安装后，一般都会有30天的免费试用期，试用期过后就会变成精简版，如诺想要永久试用就需要付费，但是我们将其在电脑中注册表删除，就可以再次获得30天的免费试用。该方法简单方便不需要破解。唯一的缺点就是每个月都需要去删除一下（土豪忽略）。

**1、cmd后输入regedit**

**2、找到HKEY_CURRENT_USER/software/install options**

**3、删除options6.9**

最后，重新启动Mathtype即可。

转载自：[https://zhuanlan.zhihu.com/p/329959853](https://zhuanlan.zhihu.com/p/329959853)