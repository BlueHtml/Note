---
layout: post
title:  "Redis热点Key的发现与解决之道-阿里云"
categories: 技术
tags: redis 阿里云
excerpt: 本文是观看 Redis热点Key的发现与解决之道-阿里云redis直播 之后的个人总结。
---

* content
{:toc}

> 本文是观看[Redis热点Key的发现与解决之道-阿里云redis直播](https://www.bilibili.com/video/BV1pf4y1E7Tp?p=3)之后的个人总结，视频已附在文末。

## 热点问题的成因

- 用户消费的数据远远大于生产的数据
- 热卖商品、热点新闻、热点评论
- 请求分片集中，单Server性能极限

1和2，初步想一想感觉没啥问题，因为只是只读请求，不锁不就行了吗。

其实不是这样的，还得考虑其他问题。例如：
1. 流量/带宽超限了，会有问题
2. 请求过多，后台服务处理不过来。包括：api接口、缓存服务、数据库服务...它们能处理的请求都是有上限的，不是无限的。
3. 更多请看下面的[为何要重视热点key问题](#为何要重视热点key问题)

## 为何要重视热点key问题

- 流量集中，达到物理网卡上限
- 请求过多，缓存分片服务被打垮
- 缓存击穿，请求穿透，引起雪崩

## 热点key的解决方法

热点，说明命令非常多，积累起来的总耗时也就很大了。

### 读写分离

![redis-热点key发现解决-读写分离](https://img.guoqianfan.com/note/2021/08/redis-热点key发现解决-读写分离.png)

架构：
- SLB：负载均衡。转发分配流量到proxy
- proxy：区分`读`和`写`，并把`写`转发到不同的readonly节点上
- 多个redis：存储数据。每个redis的数据都是相同的。
    - master：只写
    - slave：备份-高可用
    - readonly：只读。可以链式扩展

其他：
- 增加只读节点
- 实现方式：链式升级版（正规链式查看[Redis读写分离介绍-阿里云](/2021/09/22/Redis读写分离介绍-阿里云)）：
    - master：只写
    - slave：备份-高可用
    - readonly：只读。可以链式扩展

优点：

- 无限扩展。

    因为是链式的，想扩展了，在readonly尾部追加一个新的即可。

- master同步压力小

    master只负责与它直连的2个：slave和readonly。每个readonly也都只负责与它直连的一个readonly即可。

缺点：

- 复制链越长，同步延迟越大。存在数据不一致性。

    每一节的复制都是需要时间的，越到后面，所需时间越长。
- 成本高。

    每一个readonly都是一个完整的redis，都是master数据的完整copy，成本要高些。

### proxy缓存

![redis-热点key发现解决-proxy缓存](https://img.guoqianfan.com/note/2021/08/redis-热点key发现解决-proxy缓存.png)

架构：
- SLB：负载均衡。转发分配流量到proxy
- proxy：
    - 缓存热点数据
    - key分片。proxy的分片规则都是一样的，才能保证key进入不同的proxy后都会被分片到同一个redis里。
    - 可增加节点（水平扩展）
- 多个redis：用于分片存储数据。每个redis的数据都是不同的。

原理：
1. redis定时计算热点数据集合
2. key来到redis里读取数据时，如果是热点，则在返回value的同时，带上热点标记
3. proxy识别热点标记，把数据缓存起来
4. proxy接收到key读取，该key已被自己缓存，则直接取出value进行返回，不再访问redis。
5. proxy接收到key写入，该key已被自己缓存，直接失效掉，并访问redis进行写入。其他的proxy则需要等待自动过期才行。因为每个proxy都是独立的。

优点：

- 无限扩展。

    每个proxy都是独立的，增加proxy即可。

- 成本较低。

    [读写分离模式](#读写分离)的每个readonly节点都存储的是完整的数据，成本高。而proxy只存储真正的热点，数量较少，成本低。

缺点：

- 数据不一致性。

    每个proxy都是独立的。热点更新后，其他的proxy只能等待自动过期后，并且还需要再收到读取请求后才会更新。
- 容量小。

    成本低，但是容量小，能存储的热点数据量有限。

redis有自己的热点计算规则，来维护热点数据集合。

proxy有自己的热点缓存失效规则(时间)，也会在key写入时主动过期。

#### redis计算热点

redis中的热点计算规则请观看[Redis 4.0解密-阿里云redis直播](https://www.bilibili.com/video/BV1pf4y1E7Tp?p=1)的`00:14:05-00:22:32`部分。

- 基于统计阈值的热点统计
- 基于统计周期的热点统计。1天访问1W次不算，1秒访问1W次才算。
- 基于版本号实现的无需重置初始值的统计方法。版本号作为热点集合的key，新的版本号进来时，发现没有，就重新统计热点，之前的就废弃掉了？？
- 对性能影响及其微小
- 内存占用及其微小。热点集合很小。

## 视频

<iframe src="//player.bilibili.com/player.html?bvid=BV1pf4y1E7Tp&page=3" width="100%" height="500px" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>