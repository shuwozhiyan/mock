﻿# Nginx

##Nginx简介
- Nginx 是俄罗斯人编写的十分轻量级的 HTTP 服务器
- Nginx 以事件驱动的方式编写，所以有非常好的性能，同时也是一个非常高效的反向代理、负载平衡。
- 拥有匹敌 Lighttpd 的性能，同时还没有 Lighttpd 的内存泄漏问题，而且 Lighttpd 的 mod_proxy 也有一些问题并且很久没有更新。

##Nginx特点
- 处理静态文件，索引文件以及自动索引；打开文件描述符缓冲．
- 无缓存的反向代理加速，简单的负载均衡和容错．
- FastCGI，简单的负载均衡和容错．
- 模块化的结构。包括 gzipping, byte ranges, chunked responses,以及 SSI-filter 等 filter。如果由 FastCGI 或其它代理服务器处理单页中存在的多个 SSI，则这项处理可以并行运行，而不需要相互等待。
- 支持 SSL 和 TLSSNI．

##Nginx架构
1. 概要
    - Nginx 在启动后，在 unix 系统中会以 daemon 的方式在后台运行，后台进程包含一个 master 进程和多个 worker 进程。
    - master 进程主要用来管理 worker 进程，包含：接收来自外界的信号，向各 worker 进程发送信号，监控 worker 进程的运行状态，当 worker 进程退出后(异常情况下)，会自动重新启动新的 worker 进程。
    -  worker 进程处理基本网络事件。
    -  一个请求，只可能在一个 worker 进程中处理，一个 worker 进程，不可能处理其它进程的请求。
    -  worker 进程的个数一般设置为机器cpu核数。
    -  每个 worker 里面只有一个主线程，Nginx 采用了异步非阻塞的方式来处理请求解决高并发。
    -  所有 worker 进程在注册 listenfd 读事件前抢 accept_mutex，抢到互斥锁的那个进程注册 listenfd 读事件，在读事件里调用 accept 接受该连接。当一个 worker 进程在 accept 这个连接之后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接，这样保证了一个请求，完全由 worker 进程来处理，且只在一个 worker 进程中处理。

    #####这样做的好处：
    - 对于每个 worker 进程来说，独立的进程省掉了锁带来的开销
    - 一个进程退出后，其它进程还在工作，服务不会中断，master 进程则很快启动新的 worker 进程。
    - worker 进程的异常退出，会导致当前 worker 上的所有请求失败，不过不会影响到所有worker请求，所以降低了风险。
2. 负载均衡
- Nginx提供的负载均衡策略有两种：
- - 内置策略：轮询，加权轮询，Ip hash
- - 扩展策略
- Ip hash算法大概原理是：根据请求所属的客户端IP计算得到一个数值，然后把请求发往该数值对应的后端，这样同一个客户端的请求，都会发往同一台后端，除非该后端不可用了，因此ip_hash能够达到保持会话的效果。
3. web缓存
- Nginx可以对不同的文件做不同的缓存处理，配置灵活，并且支持FastCGI_Cache，主要用于对FastCGI的动态程序进行缓存。
4. Nginx配置文件结构
- 全局块：配置影响nginx全局的指令。一般有运行nginx服务器的用户组，nginx进程pid存放路径，日志存放路径，配置文件引入，允许生成worker process数等。
- events块：配置影响nginx服务器或与用户的网络连接。有每个进程的最大连接数，选取哪种事件驱动模型处理连接请求，是否允许同时接受多个网路连接，开启多个网络连接序列化等。
- http块：可以嵌套多个server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。如文件引入，mime-type定义，日志自定义，是否使用sendfile传输文件，连接超时时间，单连接请求数等。
- server块：配置虚拟主机的相关参数，一个http中可以有多个server。
- location块：配置请求的路由，以及各种页面的处理情况。
- 示例如下
```
########### 每个指令必须有分号结束。#################
#user administrator administrators;  #配置用户或者组，默认为nobody nobody。
#worker_processes 2;  #允许生成的进程数，默认为1
#pid /nginx/pid/nginx.pid;   #指定nginx进程运行文件存放地址
error_log log/error.log debug;  #制定日志路径，级别。这个设置可以放入全局块，http块，server块，级别以此为：debug|info|notice|warn|error|crit|alert|emerg
events {
    accept_mutex on;   #设置网路连接序列化，防止惊群现象发生，默认为on
    multi_accept on;  #设置一个进程是否同时接受多个网络连接，默认为off
    #use epoll;      #事件驱动模型，select|poll|kqueue|epoll|resig|/dev/poll|eventport
    worker_connections  1024;    #最大连接数，默认为512
}
http {
    include       mime.types;   #文件扩展名与文件类型映射表
    default_type  application/octet-stream; #默认文件类型，默认为text/plain
    #access_log off; #取消服务日志    
    log_format myFormat '$remote_addr–$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for'; #自定义格式
    access_log log/access.log myFormat;  #combined为日志格式的默认值
    sendfile on;   #允许sendfile方式传输文件，默认为off，可以在http块，server块，location块。
    sendfile_max_chunk 100k;  #每个进程每次调用传输数量不能大于设定的值，默认为0，即不设上限。
    keepalive_timeout 65;  #连接超时时间，默认为75s，可以在http，server，location块。

    upstream mysvr {   
      server 127.0.0.1:7878;
      server 192.168.10.121:3333 backup;  #热备
    }
    error_page 404 https://www.baidu.com; #错误页
    server {
        keepalive_requests 120; #单连接请求上限次数。
        listen       4545;   #监听端口
        server_name  127.0.0.1;   #监听地址       
        location  ~*^.+$ {       #请求的url过滤，正则匹配，~为区分大小写，~*为不区分大小写。
           #root path;  #根目录
           #index vv.txt;  #设置默认页
           proxy_pass  http://mysvr;  #请求转向mysvr 定义的服务器列表
           deny 127.0.0.1;  #拒绝的ip
           allow 172.18.5.54; #允许的ip           
        } 
    }
}
```








