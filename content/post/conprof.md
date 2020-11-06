---
title: "Conprof: Like Prometheus, but for profiles"
date: 2020-11-06T15:24:27-05:00
draft: false 
description: ""
tags:
- "Conprof"
categories: 
- "Observability"
---

Conprof 是一个对应用程序进行持续 profilling 的系统，它基于https://research.google/pubs/pub36575/。Conprof 目前已经在我们组生产环境的集群上稳定运行了几个月了，并且能够在 debug 线上集群问题方面提供非常大的帮助。 据我所知，Conprof 最早亮相于 KubeCon EU 2019 的 Keynote 演讲上。Conprof 的主要作者 Frederic 和 Grafana 的 VP Tom 分享了 Observability 的未来趋势，当时他们预测了三点，很有趣的是，这三点在 2020 年底来看，基本已经成为现实：

- More correlation between the three pillars (monitoring, logging and tracing)
- New signals and new analysis
- Rise of index free log aggregation



先聊聊第三点。Index free log aggregation 的服务，主要指的就是 Grafana Loki。以往我们在日志的解决方案上通常采用 EFK 来收集、索引以及查询日志，但是这真的有必要吗？很多时候我们并不需要建立那么多的索引，我们只需要类似 grep 那样的功能来对日志进行查询。就这有了 Loki，它并不像 ES 一样需要你自己管理索引，而是采用 Prometheus 一样的 labels 机制来索引日志。这一点在 Kubernetes 的场景下尤为重要，因为这也就意味着 每个 Pod 的监控指标和日志被相同的 labels 元数据信息给串联了起来。



再来说第一点，在日前结束的 ObservabilityCon 上， Grafana 演示了一个 demo 来展示 他们如何打通 metrics，logs，traces。首先在查询 Loki 的日志，可以找到每个请求的 trace ID。通过 trace ID 可以唯一识别一个 trace， 从而跳转到 Trace 的 UI 进行查询。



在 metrics 与 trace 的结合上，主要是采用 exemplars 的机制在 metrics 中加带上额外的 label pairs。Prometheus 在收集 metrics 的时候也会收集这些数据并存储下来，而在查询的时候通过一个单独的 API 来暴露 exemplars 信息。借用这种机制，我们可以把 trace ID 作为一个 label pair 加入 exemplar 中，从而可以从 Prometheus 中查询到 trace 的信息。



好的，感觉扯的有点远了。最后讲讲今天的主题，第二点中的 new signals，指的就是 profiles。pprof 目前已经在 Go 的生态中被广泛运用，而在其他语言上也有 pprof 的实现，例如 pprof-rs。Profiles 可以帮助我们分析，观察程序的运行性能，包括 CPU，内存（heap），goroutine 等等。从CPU profiles的函数调用耗时中，我们可以很直观的定位到程序当前的瓶颈。



profiles 与 metrics 类似，都是通过 HTTP 端点进行暴露。那么如果像 Prometheus 一样，每隔一段时间定期去抓取程序的 profiles 并存储在 TSDB 中，后续出现问题了再去查询那个时间段的 profiles，就能够很方便地定位到问题了，这就是 Conprof。

Conprof 本质上就是与 Prometheus 一样的系统，可以通过一个二进制文件直接运行。不过为了更好的可扩展性，它也支持微服务模式来更好的支持横向扩展。它将内部的组件分成了三个部分，并通过 gRPC 进行通信。主要的组件包括：

1. Sampler
2. Store
3. API

Sampler 是数据的写入部分，它直接使用了 Prometheus 的服务发现模块来发现 targets，定期对他们进行 Profiling，并将结果的 protobuf 压缩后，通过gRPC API存储到 TSDB 中。

Store 就是一个 gRPC 封装起来的 TSDB。Conprof 的 TSDB 是一个 fork 版本的 Prometheus TSDB。它与上游的 TSDB 基本一样，唯一的区别就是 Conprof 里面 sample 的值是 []byte, 而 Prometheus sample 的值是 float64，这也导致了 Prometheus 中的数据可以应用 Gorilla paper 中的方式进行压缩存储，而 Conprof 不行，所以在内存和磁盘使用量上目前还是会高于 Prometheus。



最后是 API 也就是 Query 组件。它对外暴露了类似于 Prometheus 的 API，包括 `query`, `label_names`, `label_values`, `series` 等等，它也包括一个非常基本的 web UI 来查询 profiles。



不过目前来说这个 UI 来处于非常初级的阶段，如果想要将 Conprof 与 Grafana 更好的集成，可以看看这个为 Grafana 实现的 Conprof 数据源，用来在 Grafana上通过 annotations 来展示 profiles https://github.com/yeya24/conprof-datasource 。



未来？









