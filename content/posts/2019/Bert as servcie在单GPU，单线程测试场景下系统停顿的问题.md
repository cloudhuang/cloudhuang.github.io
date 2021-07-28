---
title: "解决Bert as servcie在单GPU，单线程测试场景下无响应的问题"
date: 2019-05-15
categories: ["Bert"]
tags: ["Bert", "Bert as Service"]
---

## 问题描述：
在一个单GPU（RTX 2080 Ti）的服务器上进行[bert as servcie](https://github.com/hanxiao/bert-as-service)的压力测试，当使用单线程跑的时候，在JMeter连续请求1分钟左右，bert as servcie会停顿，没有响应。
但是当多个线程执行，或者在多核CPU上则无法重现。

## 原因：

```
I:PROXY:[htt:enc: 48]:new request from 192.168.0.83
I:VENTILATOR:[__i:_ru:194]:new encode request   req id: 17813   size: 1 client: b'e38043ad-153b-427a-b78d-d148c2032c11'
I:SINK:[__i:_ru:352]:job register       size: 1 job id: b'e38043ad-153b-427a-b78d-d148c2032c11#17813'
I:WORKER-0:[__i:gen:553]:new job        socket: 0       size: 1 client: b'e38043ad-153b-427a-b78d-d148c2032c11#17813'
I:WORKER-0:[__i:_ru:528]:job done       size: (1, 768)  client: b'e38043ad-153b-427a-b78d-d148c2032c11#17813'
I:SINK:[__i:_ru:332]:collect b'EMBEDDINGS' b'e38043ad-153b-427a-b78d-d148c2032c11#17813' (E:1/T:0/A:1)
I:SINK:[__i:_ru:341]:send back  size: 1 job id: b'e38043ad-153b-427a-b78d-d148c2032c11#17813'
192.168.0.83 - - [15/May/2019 03:19:37] "POST /encode HTTP/1.1" 200 -
I:PROXY:[htt:enc: 48]:new request from 192.168.0.83
I:VENTILATOR:[__i:_ru:194]:new encode request   req id: 17814   size: 1 client: b'e38043ad-153b-427a-b78d-d148c2032c11'
I:SINK:[__i:_ru:352]:job register       size: 1 job id: b'e38043ad-153b-427a-b78d-d148c2032c11#17814'
I:WORKER-0:[__i:gen:553]:new job        socket: 0       size: 1 client: b'e38043ad-153b-427a-b78d-d148c2032c11#17814'
I:VENTILATOR:[__i:_ru:178]:new config request   req id: 4160    client: b'fafe3701-d689-4f36-ac79-f5042ae30d2d'
I:WORKER-0:[__i:_ru:528]:job done       size: (1, 768)  client: b'e38043ad-153b-427a-b78d-d148c2032c11#17814'
I:SINK:[__i:_ru:332]:collect b'EMBEDDINGS' b'e38043ad-153b-427a-b78d-d148c2032c11#17814' (E:1/T:0/A:1)
I:SINK:[__i:_ru:341]:send back  size: 1 job id: b'e38043ad-153b-427a-b78d-d148c2032c11#17814'
192.168.0.83 - - [15/May/2019 03:19:37] "POST /encode HTTP/1.1" 200 -
I:PROXY:[htt:enc: 48]:new request from 192.168.0.83
I:VENTILATOR:[__i:_ru:194]:new encode request   req id: 17815   size: 1 client: b'e38043ad-153b-427a-b78d-d148c2032c11'
I:WORKER-0:[__i:gen:553]:new job        socket: 0       size: 1 client: b'e38043ad-153b-427a-b78d-d148c2032c11#17815'
I:WORKER-0:[__i:_ru:528]:job done       size: (1, 768)  client: b'e38043ad-153b-427a-b78d-d148c2032c11#17815'
I:SINK:[__i:_ru:355]:send config        client b'fafe3701-d689-4f36-ac79-f5042ae30d2d'
I:SINK:[__i:_ru:332]:collect b'EMBEDDINGS' b'e38043ad-153b-427a-b78d-d148c2032c11#17815' (E:0/T:0/A:0)
I:SINK:[__i:_ru:352]:job register       size: 1 job id: b'e38043ad-153b-427a-b78d-d148c2032c11#17815'
192.168.0.28 - - [15/May/2019 03:19:37] "GET /status/server?_=1557800692813 HTTP/1.1" 200 -
I:PROXY:[htt:enc: 48]:new request from 192.168.3.200
```

上面是bert as servcie的log，通过分析log，发现一个特殊的现象，每次停止响应，最后均发现了`send config        client b'fafe3701-d689-4f36-ac79-f5042ae30d2d'`这样一条log。

bert-as-service使用了zeromq的sink方式来进行socket的通信，当一个新的请求过来，bert-as-servcie源码中会建立一个`ServerCmd.new_job`的sink process,这个时候系统是正常运行的。
然后，系统会发出`ServerCmd.show_config`信号，来检查系统的状态。
由于是单GPU，这个时候，WORKER只有一个，检查系统状态的这个事件就中断了上面的`ServerCmd.new_job`的sink process了。


## 解决方法：
找准了问题之后，解决方案则比较容易了：
1. 多卡，采用多个GPU
2. 单卡的情况下，关闭系统的状态监测事件

但是在真实环境中，这个场景并不会出现，即不大会存在一个线程连续的请求1分多钟，哪怕出现，当发起新的请求后，系统也马上就可以恢复。