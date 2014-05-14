---
layout: post
title: tornado websocket
description: "websocket tornado 代码 nginx配置"
modified: 2014-05-11
tags: [varnish,web]
comments: true
share: true  
---

###websocket代码：

{% highlight python linenos %}

class SessionPushSocket(websocket.WebSocketHandler):
    
    def open(self, m=None):
        #timeout ten mins
        self.time_out = 10 * 60
        self.time_wait = 1
        self.loop_limit = self.time_out / self.time_wait
        self.loop_index = 0
        logging.debug("WebSocket opened:%s" % m)


    def on_message(self, message):
        logging.debug(u"You said: " + message)
        self.register_loop(message)
 

    def register_loop(self, guid):
        if self.loop_index < self.loop_limit:
            self.loop_index += 1
            ioloop.IOLoop.instance().add_timeout(
                time.time() + self.time_wait,
                lambda: self.get_data(guid=guid, callback=self.push)
            )
        else:
            #release time out connection
            try:
                self.close()
            except Exception, e:
                logging.debug(e)


    def get_data(self, key, callback):
        if self.ws_connection and self.ws_connection.stream.closed():
            logging.debug("%s:close" % guid)
            return

        #get data from store
        data = get_from_store(key)
        logging.debug("data:%s" % phone)
        if data:
            callback(data)
        else:
            #next loop
            self.register_loop(guid)

    def push(self, message):
        self.write_message(message)

    def on_close(self):
        logging.debug("WebSocket closed")

{% endhighlight %}

###nginx配置

nginx 需要新版本 1.4 以上
{% highlight python linenos %}

http {
    ...
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        ...

        location /chat/ {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
        }
    }
}
{% endhighlight %}
