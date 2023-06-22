---
layout: post
title: "解决mac与win双系统时间不同步问题"
subtitle: "转载"
author: "Slc"
#header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - hackintosh
  - 技术类
---
自从装了win和mac双系统之后，一直出现mac和win时间不同步的问题，查阅资料找到解决方法，避免以后忘记。

转自：https://blog.csdn.net/u013705509/article/details/50662897


很多同学都是 mac osx 和 windows 双系统,但是有个问题,进入mac osx再进windows 时间就不对了这个是因为 Windows 与 Mac缺省看待系统硬件时间的方式是不一样的： Windows把系统硬件时间当作本地时间(local time)，即操作系统中显示的时间跟BIOS中显示的时间是一样的。

Mac把硬件时间当作 UTC，操作系统中显示的时间是硬件时间经过换算得来的，比如说北京时间是GMT+8，则系统中显示时间是硬件时间+8这样，当PC中同时有多系统共存时，就出现了问题。

假如你的mac设置的时区都为北京时间东八区，当前系统时间为9：00AM。则此时硬件中存储的实际是UTC 时间1:00AM。这时你重启进入Windows后，你会发现windows系统中显示的时间是 1:00AM，比mac中慢了八个小时。

同理，你在Windows中更改或用网络同步了系统时间后，再到mac中去看，系统就会快了8小时。

解决这个问题的方法很简单： 修改 Windows 对硬件时间的对待方式，这样只在 Windows 上修改后就无需在mac上设置了。 让 Windows 把硬件时间当作 UTC

window7用户开始->运行->输入CMD

window8/10用户 WIN+x 选择管理员模式进入CMD

输入下面命令并回车 代码:

```
Reg add HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation /v RealTimeIsUniversal /t REG_DWORD /d 1
```

这样一来就不用在osx下安装时间补丁了.


————————————————


版权声明：本文为CSDN博主「洛阳如是」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。


原文链接：https://blog.csdn.net/u013705509/article/details/50662897


