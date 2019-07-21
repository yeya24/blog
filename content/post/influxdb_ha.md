---
title: "influxdb的高可用方案"
date: 2019-03-02T22:32:12+08:00
draft: false 
description: ""
tags:
- "Database"
categories: 
- "Database"
---
由于目前influxdb本身的集群方案属于闭源状态，而本身的influxdb并不支持集群。在目前的社区中出现的方案主要有两种：

- influxdb-relay　这是官方提供的高可用方案，已经有２年没有更新了，目前有一个官方的fork还在活跃
- influx-proxy　饿了吗的开源的方案，大概是在influxdb-relay的基础上进行了修改

## influxdb-relay
这里主要讲一下目前活跃的relay的fork，github地址在[influxdb-relay](https://github.com/vente-privee/influxdb-relay)

[![kqtEdI.png](https://s2.ax1x.com/2019/03/02/kqtEdI.png)](https://imgchr.com/i/kqtEdI)

fork版本的relay与原版功能比较类似，对外是一个httpserver，收到写请求后能够支持通过http或者udp方式向influxdb写入数据，并且也可以支持prometheus的远程写入(http)。不过不足的是，relay并不支持influxdb的查询接口，也不支持prometheus的远程读。

当数据写入的请求到达relay这一层，relay会对自己配置的后端的influxdb发起写请求，不过这里并没有能够解决数据一致性的问题。如果写入influxdb时失败，则只会简单的返回一个错误信息，写入失败的数据就在这一个influxdb中丢失了。

## influx-proxy
influx-proxy 是饿了吗开源的方案。github地址在[influx-proxy](https://github.com/shell909090/influx-proxy)。

[![kqtAeA.png](https://s2.ax1x.com/2019/03/02/kqtAeA.png)](https://imgchr.com/i/kqtAeA)

上面是proxy给出的一个架构图。可以看出proxy组件同时支持外面的查询和写入请求。当proxy收到数据向influxdb写入只支持http的方式。在proxy和influxdb中间多了一个中间层，在这个中间层可以对写入数据进行分片。在配置文件中，可以针对不同的measurement（在prometheus中称为metric），配置它们发送到特定的后端influxdb。

proxy的架构中包含了redis。一开始以为redis是用来解决relay中没有解决的数据一致性的问题，当写入失败之后，将数据存入redis缓存，后面再写，后面发现proxy并没有这么使用redis。redis在其中的作用仅仅是存储proxy的配置信息，包括存储分片配置，influxdb后端的地址、使用的数据库、超时时间等等。proxy也提供了一个 reload 的热加载的接口，热加载时会去读取redis中的配置然后重新启动后端服务，redis只是用来干这件事情的。如果只是用于加载配置，不知道为什么不采用简单的文件方式而要使用redis。

proxy的写入的过程中，解决了之前relay的写入一致性问题。在proxy启动之后，读取配置中后端的influxdb，并为每一个后端生成一个数据文件和一个元数据文件。当通过http接口写入数据时，proxy会向符合分片配置的所有influxdb实例进行写入，如果向某个db写入成功，则返回；若失败，则将需要写入的数据写到该db对应的数据文件中。

有另外一个goroutine会去查看数据文件，发现数据文件中有数据，则会再对该influxdb尝试写入。如果此次写入成功，则删除刚刚写进去的这条数据。如果写入失败，则对数据文件进行回滚操作。具体的回滚操作需要用到元数据文件，在每次后台的goroutine发现数据文件有文件需要写入前，会将本次要写入的数据先写进元数据文件，之后尝试写influxdb，若失败，则将数据文件里的文件偏移(offset)回复到之前的位置(根据元数据文件里的信息判断长度)。

不过很奇怪的一点是，上面所说的这个后台goroutine只有在加载配置的时候才会启动一次，运行完就结束。加载配置只有在proxy启动或者调用reload接口时运行，而不是一直在后台运行，这一点真的让人很难理解。此外，采用文件来保存写入失败的数据的这一做法会消耗大量的磁盘io，性能受到比较大的影响，还需要承受文件系统损坏的风险。

