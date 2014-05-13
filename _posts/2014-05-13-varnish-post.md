---
layout: post
title: varnish
description: "varnish 学习记录"
modified: 2014-05-11
tags: [varnish,web]
comments: true
share: true  
---

varnish是一个http反向代理服务，可以用作http或web加速器，把文件或请求结果存到内存。从本质上说它可以认为是一个key/value存储系统，以url做key.

###(一) varnish 请求处理流程图
<figure>
	<img src="{{site.url}}/images/varnish_overview.jpg" alt="">
</figure>

###(二) VCL配置语言
&nbsp;&nbsp;&nbsp;&nbsp;vcl语言是专用于varnish配置的一门简单语言。当vcl被load的时候，会被转化为c语言一起编译为共享object，在varnish进程运行时被动态链接。

### 语法与c语言类似

>1. 以大括号界定代码块,以分号结束语句
2. 赋值 (=), 比较 (==, !=)  bool表达式 (!, && and \|\|) 等等都与C语言类似
3. 基本字符串: "..." 不支持换行
4. 长字符串: {"..."} 支持新行 和除了 NULL 以外的一些控制字符
5. 不同于c的是反斜杠在vcl中没有转义的意思
6. 字符串连接用'+'号
7. 常用关键字：
	1. set：赋值使用set关键字，比如 set req.backend = foo ，在varnish中没有用户自定义变量，
	    只能对 backend, request or document objects
	2. unset：删除变量值
    3. panic：给客户端发出警告信息
    4. rollback：回滚变量
    5. synthetic：在vcl_error中生成一个response body
    6. return：返回一个action return(action)

### action列表
	
>* deliver
* error
* fetch
* hash
* hit_for_pass
* lookup
* ok
* pass
* pipe
* restart

### backend
首先需要定义一个backend
{% highlight css linenos %}
backend www {
  .host = "www.example.com";
  .port = "80";
  .connect_timeout = 1s;
  .first_byte_timeout = 5s;
  .between_bytes_timeout = 2s;
}
{% endhighlight %}

### 健康检查 director
director 的作用是让varnish从后端服务器中选择可用的没有宕机的服务器，将请求打到健康的服务器上。varnish提供了多种检测的机制，每一种机制
都用不同的算法来实现如何寻找可用的后端服务器，下面是一个最基本的健康检查配置：
{% highlight python linenos %}
director b2 random {
  .retries = 5;
  {
    // We can refer to named backends
    .backend = b1;
    .weight  = 7;
  }
  {
    // Or define them inline
    .backend  = {
      .host = "fs2";
    }
  .weight         = 3;
  }
}
{% endhighlight %}




