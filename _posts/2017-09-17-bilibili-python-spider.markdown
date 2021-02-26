---
layout: post
title:  "🍌爬取bilibili视频封面"
date:   2017-09-17 20:43:00 +0800
tags: Python
---

### 目标

+ 能够爬取指定up主视频封面
+ 能够设定爬取封面数量

### 工具

python3 / chrome

### 思路

首先，直接查看加载完成后的网页源代码，例如： 
 
[up主Shymieeeee的稿件](http://space.bilibili.com/879702/#!/video)  

发现源代码中并未包含与视频封面图相关的信息，而是在body中包含着如下的信息

{% highlight html %}
<div id="browser-version-tip">
  <div class="wrapper">
    抱歉，您正在使用不支持的浏览器访问个人空间。推荐您<a href="//www.google.cn/chrome/browser/desktop/index.html">安装 Chrome 浏览器</a>以获得更好的体验 ヾ(o◕∀◕)ﾉ
    <a href="javascript:;" id="close-browser-tip"></a>
  </div>
</div>
<div id="space-body"></div>
{% endhighlight %}

在使用chrome审查元素时，能够看到网页中包含着相关信息：  

![审查工具查看结果]({{ site.url }}/assets/pic9172210.png)

可见整个网页是使用js动态生成的，要找到数据的来源，可以使用chrome工具中的Network，查看网页在加载过程中加载了哪些数据，选择查看XHR类型，可以找到对应的JSON数据。

分析数据，可以看出，在JSON数据中，["data"]["vlist"]中的各项list均为稿件信息，其中每个list中的["pic"]即为视频封面的地址。

### 设计

整个代码十分简洁，如下所示：

{% highlight python %}
import urllib.request
import json
upID = 879702
num = 100
data = urllib.request.urlopen('https://space.bilibili.com/ajax/member/getSubmitVideos?mid='+str(upID)+'&pagesize='+str(num)+'&tid=0&page=1&keyword=&order=pubdate')
jsondata = json.load(data)
x = 0
for item in jsondata['data']['vlist']:
    imgdata = urllib.request.urlopen('http:'+item['pic'])
    img = imgdata.read()
    file = open(str(upID)+'img'+str(x)+'.jpg','wb')
    file.write(img)
    x = x + 1
{% endhighlight %}

先使用urlib模块获取数据，使用json模块解析数据，然后查找到封面图的位置，进行下载保存，代码中upID用于设置up主ID，num为爬取的封面数量，测试可完美达成目标。

如有任何疑问，欢迎微博/bilibili私信，相关信息在about页面。