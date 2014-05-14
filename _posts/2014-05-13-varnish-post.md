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

### director
director 是一组后端服务器的集合，作用是让varnish从后端服务器中选择可用的的服务器，将请求打到健康的服务器上。varnish提供了多种director的机制，每一种机制
都用不同的算法来实现如何寻找可用的后端服务器，下面是一个random director的配置：
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

###四种director机制：

>The family of random directors 三种方式都是通过随机来实现，可以调节权重：

	1. The random director：根据一个随机数做种子从后端服务器中选择
	2. The client director：根据client身份从后端服务器中选择
	3. The hash director：根据url hash值从后端服务器选择

>The round-robin director：根据请求的顺序分配后端服务器，第一个请求给第一个后端服务器，第二个请求给第二个后端服务器。
如果一个后端服务器不可用或者varnish链接它失败，那么这个不可用的服务器会直接被跳过。在放弃之前，会尝试连接所有后端服务器

{% highlight python linenos %}
...
backend api_server8 {
    .host = "10.105.28.144";
    .port = "80";
    .connect_timeout = 3s;
    .first_byte_timeout = 3s;
    .between_bytes_timeout = 3s;
    .probe = healthcheck;
}


director default round-robin {
    { .backend = api_server1; }
    { .backend = api_server2; }
    { .backend = api_server3; }
    { .backend = api_server4; }
    { .backend = api_server5; }
    { .backend = api_server6; }
    { .backend = api_server7; }
    { .backend = api_server8; }
}

{% endhighlight %}

>DNS director 

{% highlight python linenos %}
director directorname dns {
        .list = {
                .host_header = "www.example.com";
                .port = "80";
                .connect_timeout = 0.4s;
                "192.168.15.0"/24;
                "192.168.16.128"/25;
        }
        .ttl = 5m;
        .suffix = "internal.example.net";
}
{% endhighlight %}


上例中指定了384个后端服务器，所有的都是使用的80端口，连接超时时间是0.4s。所有在.list语句里的的选项必须定义在ip列表之前。.list方法不支持IPv6。它不是一个白名单，它只是一个在varnish内部创建的实际的后端服务器列表。定义的子网越大开销越大 .ttl定义DNS查询的缓存时间。
.suffix表示把其后面的值，追加到客户端提供的Host Header中。此director不支持健康检测。支持DNS轮询负载均衡。如果主机名解析后对应多个后端服务器，那么这director将以轮询的方式分派流量。

>The fallback director: 选择第一个可用的后端服务器。它是以服务器定义的顺序依次进行选择的。如果第一个服务器可用，它不会再去尝试连接第二个服务器，除非第一个服务器不可用了

{% highlight python linenos %}
 director b3 fallback {
    { .backend = www1; }
    { .backend = www2; } // will only be used if www1 is unhealthy.
    { .backend = www3; } // will only be used if both www1 and www2
                       // are unhealthy.
 }
 {% endhighlight %}

###Backend probes

后端服务器可以通过req.backend.healthy变量返回的探测状态来判断是否可用。

>探测器可以跟的参数有：
  
  * .url 表示指定一个发送到后端服务器的URL请求，默认值是“/”。
  * .request 表示指定一个完整的多字符串的HTTP请求，每个字符串后面都会自动的插入\r\n 回车换行符。它的优先级比.url高。
  * .window 表示我们要检查确定后端服务器的可用性，所要进行探测的次数。默认是8次。
  * .threshold 表示在选项.window中，要成功的探测多少次我们才认为后端服务器可用。默认是3次。
  * .initial 表示当varnish启动的时候，要探测多少次后则认为服务器可用。默认值和选项.threshold的值的相同。
  * .expected_response 表示你期望后端所返回的HTTP响应代码。默认是200
  * .interval 表示每隔多少秒探测一次。默认是每5秒探测一次。
  * .timeout 表示每次探测所使用的时间，如果超过设定的时间，就表示此次才探测超时，也就表示此次探测失败。默认是2秒。
  

{% highlight python linenos %}

probe healthcheck {
   .url = "/status.cgi";
   .interval = 60s;
   .timeout = 0.3 s;
   .window = 8;
   .threshold = 3;
   .initial = 3;
   .expected_response = 200;
}

backend www {
  .host = "www.example.com";
  .port = "http";
  .probe = healthcheck;
}
{% endhighlight %}

#完整的vcl配置代码

{% highlight python linenos %}
/*-
 * Copyright (c) 2006 Verdens Gang AS
 * Copyright (c) 2006-2011 Varnish Software AS
 * All rights reserved.
 *
 * Author: Poul-Henning Kamp <phk@phk.freebsd.dk>
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 *
 * THIS SOFTWARE IS PROVIDED BY THE AUTHOR AND CONTRIBUTORS ``AS IS'' AND
 * ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
 * PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL AUTHOR OR CONTRIBUTORS BE
 * LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
 * CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
 * SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR 
 * BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
 * WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE
 * OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE,
 * EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
 *
 * The default VCL code.
 *
 * NB! You do NOT need to copy & paste all of these functions into your
 * own vcl code, if you do not provide a definition of one of these
 * functions, the compiler will automatically fall back to the default
 * code from this file.
 *
 * This code will be prefixed with a backend declaration built from the
 * -b argument.
 */

sub vcl_recv {
    if (req.restarts == 0) {
        if (req.http.x-forwarded-for) {
            set req.http.X-Forwarded-For =
                req.http.X-Forwarded-For + ", " + client.ip;
        } else {
            set req.http.X-Forwarded-For = client.ip;
        }
    }
    if (req.request != "GET" &&
      req.request != "HEAD" &&
      req.request != "PUT" &&
      req.request != "POST" &&
      req.request != "TRACE" &&
      req.request != "OPTIONS" &&
      req.request != "DELETE") {
        /* Non-RFC2616 or CONNECT which is weird. */
        return (pipe);
    }
    if (req.request != "GET" && req.request != "HEAD") {
        /* We only deal with GET and HEAD by default */
        return (pass);
    }
    if (req.http.Authorization || req.http.Cookie) {
        /* Not cacheable by default */
        return (pass);
    }
    return (lookup);
}

sub vcl_pipe {
    # Note that only the first request to the backend will have
    # X-Forwarded-For set.  If you use X-Forwarded-For and want to
    # have it set for all requests, make sure to have:
    # set bereq.http.connection = "close";
    # here.  It is not set by default as it might break some broken web
    # applications, like IIS with NTLM authentication.
    return (pipe);
}

sub vcl_pass {
    return (pass);
}

sub vcl_hash {
    hash_data(req.url);
    if (req.http.host) {
        hash_data(req.http.host);
    } else {
        hash_data(server.ip);
    }
    return (hash);
}

sub vcl_hit {
    return (deliver);
}

sub vcl_miss {
    return (fetch);
}

sub vcl_fetch {
    if (beresp.ttl <= 0s ||
        beresp.http.Set-Cookie ||
        beresp.http.Vary == "*") {
                /*
                 * Mark as "Hit-For-Pass" for the next 2 minutes
                 */
                set beresp.ttl = 120 s;
                return (hit_for_pass);
    }
    return (deliver);
}

sub vcl_deliver {
    return (deliver);
}

sub vcl_error {
    set obj.http.Content-Type = "text/html; charset=utf-8";
    set obj.http.Retry-After = "5";
    synthetic {"
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN"
 "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html>
  <head>
    <title>"} + obj.status + " " + obj.response + {"</title>
  </head>
  <body>
    <h1>Error "} + obj.status + " " + obj.response + {"</h1>
    <p>"} + obj.response + {"</p>
    <h3>Guru Meditation:</h3>
    <p>XID: "} + req.xid + {"</p>
    <hr>
    <p>Varnish cache server</p>
  </body>
</html>
"};
    return (deliver);
}

sub vcl_init {
        return (ok);
}

sub vcl_fini {
        return (ok);
}
{% endhighlight %}


