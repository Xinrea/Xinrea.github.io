---
layout: post
title:  "🌰linux下 fork()函数的使用"
date:   2017-09-17 20:43:00 +0800
tags: Linux
---

### fork()函数 ###

>使用需添加头文件 `unistd.h`

函数原型

{% highlight clike %}
pid_t fork(void);
{% endhighlight %}

>pid_t为宏定义的int类型，若需使用，需`include sys/types.h`文件

其返回值有三种情况：

- 子进程返回0
- 父进程返回子进程ID
- 出错返回-1

### 简单使用情景 ###

{% highlight clike %}
if(!fork()){
	statement1;
}
else{
	statement2;
}
{% endhighlight %}

由返回值的情况可以知道，父进程中执行`statement2`，子进程中执行`statement1`，两段代码可以“看作”在fork()之后，同时开始运行，但是实际顺序无法确定，请不要在竞争情景下使用。注意，fork()函数新建了进程来完成分支。

子进程在被创建时，会获得父进程数据空间、堆、栈等资源的副本。在CPU眼里，CPU每次任务只是读取上下文数据，进行处理，保存上下文数据，因而fork()产生的进程并非同时在进行，而是在两者之间不断的切换。

相比于多线程来说，切换进程时，CPU读取和保存上下文数据比切换线程时可能消耗资源更多，切换线程时由于仍在同一进程下，在很多方面具有优势。

使用fork()可以简单完成很多需要并行处理的操作，比如TCP连接中报文的发送和接收。

如果有任何问题，欢迎通过邮箱或者私信的方式与我联系。

