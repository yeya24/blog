---
title: "玩玩 PolarSignals"
date: 2021-06-16T17:43:54-07:00
draft: false 
description: "Introduction to Polar Signals"
tags:
- "Conprof"
categories: 
- "Observability"
---



最近，Continuous Profiling 这个可观测性主题越来越受到大家的关注，这一领域蛮多新的开源项目也出现了，比如 [Pyroscope](https://github.com/pyroscope-io/pyroscope)， 以及基于 eBPF 的 [pixie](https://github.com/pixie-labs/pixie) 等等。几个月前我也写了一篇文章简单介绍了 Conprof，一个基于 Prometheus 的 Continuous Profiling 工具。而 Conprof 背后的公司 [PolarSignals](https://www.polarsignals.com/)，也推出了他们的 SaaS 服务，用来做 profiles 的存储，可视化等等。其实主要功能还是基于 Conprof 本身的 API，但是在 UI 的易用性上比起开源的版本有了很大的提升。在这篇水文中，我会简单介绍一下 PolarSignals 有哪些功能。后面为了省略，我会将 PolarSignals 简写成 PS。由于 PS 背后还是基于开源的项目构建，如果你想开发自己内部的 Continuous Profiling 平台，可能是个不错的参考。



## 配置

![collect](/img/polar_signals/collect.png)



PS 作为一个 SaaS 服务，需要使用者先在自己的环境中配置好 Collector。Collector 使用 Prometheus 服务发现的格式，抓取 profiles，再将数据写入到远端的 PS 存储。



## 查询

![query](/img/polar_signals/ps-query.png)

查询界面上，首先左侧可以选择 Profile 的类型。实际上，类似于 Prometheus，Profile 的类型在这里就是 timeseries 的名字，用 `__name__` 这个特殊的 label 来表示，比如说下面两个 timeseries 是一样的。

```
{__name__="heap", app="test-app", namespace="default"}
heap{app="test-app", namespace="default"}
```

查询栏右侧可以选择要查询的时间范围，目前 PS 默认是保存 14 天的数据。

中间的输入可以来查询不同的标签，不过注意的是这里只能用 vector selector，也不支持 promql 的 functions。

![query-heap](/img/polar_signals/query-heap.png)





在这个例子中，我们查询 heap，并指定一下 job label。点击 Search button 之后，查询首先返回的是 heap 的 metrics。在这里，metrics 的值是从用户上传的 pprof 数据中得到的，并通过 Prometheus 远程写存储到 PS 后端的 [Thanos Receiver](https://thanos.io/tip/components/receive.md/) 集群中。这么做，相当于增加了一层 metrics 和 profiles 的 correlation。因为一个 metric 的 sample 和 profile 的 sample 有着相同的时间戳，并且它们所属的时序的 labels 也一样，只是值的类型不同，后面要查询具体的 Profile 可以直接通过标签 + 时间戳来做点查。

我们在排查问题时，从 metrics 入手通常更加直观。在 metrics 的图表上，如果看到有 heap 异常的地方，单击对应的时序点，即可跳转到 profile。

![correlation](/img/polar_signals/correlation.png)

在这里可以对比 Conprof 的查询 UI，仅仅展示单独时序的样本点，而没有其他信息，通常很难找到有价值的 profiles。

![conprof-ui](/img/conprof/conprof.png)





等得到了要看的 profile，后面进行具体的分析时，可以采用 PS 提供的 View UI，功能类似于 pprof 默认的 UI，但是没有那么全，比如就缺少了搜索过滤的功能。下面这张图展示的是 `Icicle Graph`。Umm，其实只是火焰图倒过来了 = =。

![pprof-ui](/img/polar_signals/pprof-ui.png)



## 进阶查询

相比于最简单的点查，PS 在 UI 这里还提供了 2 个功能，分别是 Profiles 的聚合和比较。



![merge](/img/polar_signals/merge.png)

聚合这边比较简单，对于所选的时间范围，所有满足所选择标签的时间序列，合并他们的 profiles。由于 profiling 每次的采样得到的 sample 不一定能覆盖到你想要看的函数，聚合多个 samples 可以更方便你的查询。你也可以理解为这是一种 downsampling，可惜目前还是在查询时进行 on-the-fly 的聚合，时延较高，而不是像 Thanos compactor 那样的离线降采样。



![compare](/img/polar_signals/compare.png)

比较的用途一般来说更广。通过在左右两侧的 metrics 视图中选择不同的时间点，对对应的 profiles 进行 diff 操作。比如线上的服务发了一个新的版本出问题了，通过比较新旧版本的 profiles，就能方便的定位到改动。



## 总结

这篇文章简单介绍了一下目前 PS 所提供的功能。由于还处在 private beta 阶段，目前功能较少。但是相比于 Conprof 的 UI，PS 在易用性方面还是很友好的，此外也提供了 metrics 这一侧的集成，方便查询。

再给 PolarSignals 打个小广告，除了它的 continuous profiling 服务，它们也提供了 profiles 分享的[服务](https://share.polarsignals.com/) 。使用者可以上传 profile，在它的 UI 上进行的分析，然后分享。比如下面这个 etcd 的 [PR](https://github.com/etcd-io/etcd/pull/12919) 就是很好的例子。

