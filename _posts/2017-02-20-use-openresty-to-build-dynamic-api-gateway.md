利用OpenResty搭建动态API网关
========

![apigateway.jpg](http://upload-images.jianshu.io/upload_images/96681-f0603041860c939e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 首先,什么是API网关?
> 如上图, API 网关作为全部客户端的单一入口点

##使用API网关的优势:
* 确保客户端无法察觉应用程序是如何被拆分为多项微服务的。
* 确保客户端不受服务实例的位置的影响。
* 为每套客户端提供最优API。
* 降低请求/往返次数。举例来说，API网关能够确保客户端在单次往返中就从多项服务中检索出数据。请求数量更少意味着运行负担更低且用户体验更好。API网关对于移动应用而言是必不可少的。
* 将从客户端调用多项服务的逻辑转换为从API网关处调用，从而简化整个客户端。

# 什么是OpenResty?
>  "OpenResty is not an Nginx fork. It is just a software bundle"
>  也就是说Openresty 是基于Nginx的一个软件包, 内部包含各种可用的库, 可参考链接 [awesome-resty](https://github.com/bungle/awesome-resty) 
>  OpenResty

# 如何做到动态呢?
>  搭配服务发现, 下面使用的是[Consul](https://www.consul.io/)

正常使用Nginx作为API网关, 需要如下配置, 当加后端实例时, reload一下Nginx, 使其生效
```
upstream test {
    server 192.168.0.1;
    server 192.168.0.2;
}
```

那么只要让upstream变成动态可编程就OK了, 当新增后端实例时, 无需reload, 自动生效

#### 可参考的代码
1. [又拍云的 slardar](https://github.com/upyun/slardar), ps: 吐槽一下这个名字, "斯拉达"... 这是Dota玩多了么
1. [微博的nginx-upsync-module](https://github.com/weibocom/nginx-upsync-module)

如下nginx-upsync-module 中, 可以设置每隔n秒, 同步Consul中的kv
```
http {
    upstream test {
        # fake server otherwise ngx_http_upstream will report error when startup
        server 127.0.0.1:11111;

        # all backend server will pull from consul when startup and will delete fake server
        upsync 127.0.0.1:8500/v1/kv/upstreams/test upsync_timeout=6m upsync_interval=500ms upsync_type=consul strong_dependency=off;
        upsync_dump_path /usr/local/nginx/conf/servers/servers_test.conf;
    }

    upstream bar {
        server 127.0.0.1:8090 weight=1 fail_timeout=10 max_fails=3;
    }

    server {
        listen 8080;

        location = /proxy_test {
            proxy_pass http://test;
        }

        location = /bar {
            proxy_pass http://bar;
        }

        location = /upstream_show {
            upstream_show;
        }

    }
}
```


最后, OpenResty的开发基于lua或者C, 使用lua可以相对轻松开发出高性能的服务, 如下链接包含的代码片段, 利用openresty实现合并多个后端get/post请求, 
[利用openresty实现合并多个后端API请求](https://gist.github.com/RavenZZ/667fb1a06de18f20bea946f276937847)