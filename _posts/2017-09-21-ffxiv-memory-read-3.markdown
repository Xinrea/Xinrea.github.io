---
layout: post
title: "🍅FinalFantasyXIV 内存数据的读取（3）"
date:   2017-09-21 22:27:00 +0800
tags: Windows
cover: '/assets/pic9212218.png'
---

对于变量的定位，在此推荐使用`Cheat Engine`，算是一个远古神器吧，许多游戏辅助工具都需要借助这个工具。下面是`Cheat Engine`的主界面截图：

![Cheat Engine 主界面]({{ site.url }}/assets/pic9212218.png)

由于这个工具出现有了一定的年头了，详细的使用方法不会在本文中进行详细介绍，读者可自行查阅资料进行使用。

以技能`猛者`为例，要找到其cd时间变量的位置，需要先使用技能，然后根据剩余时间进行`精确扫描`和`比···小`两种扫描方式，可定位到如下图所示的五个地址，双击加入到列表中，右键任意地址，选择`找出是什么改写了这个地址`，可以看到这个地址的计算方法，即其指针为`RDX+4*RAX`，可以手动一层一层分析指针，也可右键地址，选择`扫描当前地址指针`，可自动寻找到指向变量地址的指针。

重复打开游戏，进行变量地址的指针扫描，会使得列表中的指针数量越来越少，最后会发现游戏中有几个指针是固定指向该地址的，而且表中给出了各处的偏移量，不同指针的区别可能与角色的其他状态有关，需要读者自行验证，最终找到了通用的指针偏移表便是成功。

![扫描变量地址]({{ site.url }}/assets/pic9212312.png)

最终给出一路上所查阅到的相关资料：

[bobopeng的csdn博客](http://blog.csdn.net/junbopengpeng/article/details/40786291)

[stackoverflow相关提问](https://stackoverflow.com/questions/37792725/how-do-i-get-the-base-address-of-another-process-aslr)

[wiki](https://en.wikipedia.org/wiki/Virtual_address_space)