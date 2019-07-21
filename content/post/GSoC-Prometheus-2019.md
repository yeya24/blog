---
title: "GSoC 2019中的Prometheus相关项目介绍"
date: 2019-03-03T13:34:50+08:00
draft: false
description: ""
tags:
- "GSoC"
- "Prometheus"
categories: 
- "Observability"
---
中国时间3月1号凌晨，cncf 公布了2019年 GSoC 项目的 idea，具体的Github地址在 [cncf/soc](https://github.com/cncf/soc)。

简单看了一下里面的一些项目，在 k8s 的项目中，比较有意思的是 kube-batch 这个调度器与 pytorch-operator/mxnet-operator 的整合，这也是个与 kubeflow 紧密相关的项目。除此以外的 idea 很多与 k8s 的 CSI 以及 dashboard 相关。在 Prometheus 的 idea 中，感觉整体的有趣程度并不及去年，不过今年 cortex 也成为了 cncf 的沙箱项目，所以对 Prometheus 感兴趣的同学也可以选择去尝试完成 cortex 的项目。一直都对 Prometheus 比较感兴趣，这篇文章的主要内容就是来介绍一下19年 GSoC 里面与 Prometheus 相关的 idea 。

# Prometheus
Prometheus 是一个开源的监控报警系统。具体信息请参考官网：[https://prometheus.io/](https://prometheus.io/)

## Benchmarks for TSDB
- 导师: Krasi Georgiev
- 技能: CI, Golang, Kubernetes

TSDB 是 Prometheus 底下的时序数据库，被放在一个单独的 repo 中，链接在 [prometheus/tsdb](https://github.com/prometheus/tsdb)。简而言之就是为 TSDB 写一个 benchmark，与 Prombench 项目类似。

## Continue the work on Prombench
- 导师: Krasi Georgiev
- 技能: CI, Golang, Kubernetes, Grafana

Prombench 是18年的 GSoC 项目，包括了 Prometheus 的工作流中一整套的 benchmarks（不包括 TSDB）。 今年的工作是修复项目目前的一些可扩展性的问题，其实也就是帮忙修 bug 和做一些扩展。主要参考该 repo 下面的 [issues](https://github.com/prometheus/prombench/issues) ，优先修复高优先级的 issue，感觉就是去搬砖，没什么好说的。

## Persist Retroactive Rule Reevaluations
- 导师: Ganesh Vernekar (@codesome), Goutham Veeramachaneni (@gouthamve)
- 技能: Golang

在 Prometheus 目前的 recording rule 中，有一个比较大的问题就是，一定要这个 recording rule 执行之后，TSDB 里面才会出现这个 metric。比如在 Grafana 中根据这个 metric 画图，一开始的一段时间你是看不到数据的，因为数据很少（一次 evaluation 的过程生成一个 metric）。这个 idea 的目的就是为了解决拿不到数据的问题，去对旧的数据进行查询来得到这个指定的 metric，使在 Grafana 里面能看到数据。另外，生成的 metrics 要求在 TSDB 中持久化一段时间。（不知道我有没有理解错意思，如果有误请指正）


这是比较老的一个 issue 了，相对来说我也感觉不是很高的优先级。不知道去年没完成的一些 idea 今年为啥不搞了，比如在 Prometheus 中支持慢查询我觉得就蛮有意义的。

参考地址：[Persist Retroactive Rule Reevaluations](https://github.com/prometheus/prometheus/issues/11)

## Optimize queries using regex matchers for set lookups
- 导师: Goutham Veeramachaneni (@gouthamve)
- 技能: Golang

通过 Prometheus 查询时序数据，通过正则匹配 labels 时，通常会出现需要同时匹配多个值的情况，在 Grafana 中这种 case 尤为常见。而目前在Prometheus 的 TSDB 中，对于正则匹配的 matcher 它的处理过程是拿到需要查询的 label 的所有的 value，然后依次进行正则匹配来筛选，很有可能出现效率问题，而这个 idea 的目的就是对这种正则匹配进行优化。比如说```up{instance=~"foo|bar|baz"}```，如果需要优化可以将这种正则匹配转换成对 TSDB 的三次查询，变成 ```up{instance="foo"}```、```up{instance="bar"}```、```up{instance="baz"}```，之后再将查询得到的 timeseries 进行聚合。当然这也只是作者给出的一个示例，实际上想做成什么样子可以自己实现。

在这个 [issue](https://github.com/prometheus/prometheus/issues/2651) 中作者也给出了 2 个实现的主要思路：

- 通过 TSDB 自身来实现优化，我认为这也是比较合理的一种，针对 matcher 处理应该是 TSDB 本身的能力。
- 在 Prometheus 层面实现一个查询优化器，可以简单的类比于 MySQL 里面的 SQL Optimizer。用户进行 promql 查询之前，对用户输入的表达式进行优化。类似于这种 ```rate(requests{instance=~"foo|bar"}[1m])```，可以转化成 ```rate(requests{instance="foo"}[1m]) OR rate(requests{instance="bar"}[1m])```，将正则匹配中的或转化成了 promql 里的 or。

这个 idea 的话应该比较理想的实现方式是第一种，TSDB 的代码不多，涉及查询的那一部分的逻辑也比较清楚，不过这个小 feature 优先级不高，也不涉及 Prometheus 的核心能力，更多是对于 Grafana 查询 Prometheus 这个 case 下提供的一种优化。

## Package for bulk imports
- 导师: Ganesh Vernekar (@codesome)
- 技能: Golang

感觉是今年的 idea 里面最有用的一个 feature 了，不过感觉做起来也比较费力，目的主要是为 Prometheus 实现数据批量上传的 API。上传的数据包括 Prometheus 的数据以及非 Prometheus 的数据（如 OpenTSDB、Influxdb）。这个 idea 有很大的 usecase：

- 实现了批量上传之后，可以说更方便做 backup 了。监控的 metrics 数据可以另外做备份，需要时可以通过诸如 promtool 通过批量导入的 API 将数据载入 Prometheus，有点像 etcdctl 的 backup、restore。
- 更加方便做测试。比如你需要得到一些生产中的 metrics，然后根据它们来写一些报警规则或者测试 Grafana 的 dashboard。不过你只有你本地的 PC 的环境，没办法去连生产环境的 Prometheus，那么数据从哪里来？如果有批量导入数据可以很轻松的解决这个问题。
- 更加方便其他监控系统的用户转向 Promethues。如果你以前采用的是某个监控系统，比如 Influxdb、OpenTSDB，如果你想转而采用 Prometheus，那么对不起一切从零开始。如果批量上传的 API 能够对接其他系统的数据的话，那迁移其实就非常方便了。

这个 idea 工作量感觉还是蛮大的，目前的情况下在 TSDB 那边刚刚 merge 了一个 vertical querier 的实现，能够支持读取时序有覆盖的数据块里面的数据，有了这个 feature，稍微减少了一些工作量。值得一提的是 mentor 小哥 Ganesh 对于时序数据库非常的了解，为 Prometheus 和 TSDB 贡献了多个大 feature，如果有他做 mentor 的话应该还是能学到不少东西的。

参考 issue:  
[bulk input](https://github.com/prometheus/prometheus/issues/535)、
[vertical querier](https://github.com/prometheus/tsdb/pull/370)

# Linkerd2
Linkerd2 是一种 ServiceMesh，项目原生就提供了狂拽酷炫吊炸天的 Prometheus 和 Grafana 集成。

![linkerd-web](http://pnf62xzme.bkt.clouddn.com/linkerd-web.png)

![linkerd-Grafana](http://pnf62xzme.bkt.clouddn.com/linkerd-grafana.png)

## Alertmanager Integration
- 导师: Thomas Rampelberg (@grampelberg)
- 技能: Go, Prometheus, Grafana, Kubernetes

Linkerd 目前虽然已经集成了 Prometheus 和 Grafana，但目前并没有配置默认的告警规则，当然也没有使用 Alertmanager。这个 idea 主要是基于 Alertmanager 为 linkerd 实现开箱即用的告警能力。具体要求如下：

- Alertmanager 作为一个安装时的可选项。
- 为控制平面设置默认的告警规则。
- 使用者能够很轻松的配置告警的发送。提到的告警方式包括 slack 和 email，这两个都是 Alertmanager　原生支持的。
- 通过 linkerd 中的 ServiceProfile(Linkerd 中的一种 CRD) 来支持对某服务配置告警规则。

整体来说难度也不大，比较有趣。如果本身对 ServiceMesh 比较感兴趣的同学可以考虑参加。

# Cortex
Cortex 是基于 Prometheus 进行扩展的一个项目，地址在[cortexproject/cortex](https://github.com/cortexproject/cortex)。在之前两届的 PromCon 中都对这个项目有所介绍，目前也成为了 cncf 的沙箱项目。Cortex 的主要目的是实现支持多租户、可水平扩展的 Prometheus，另外它也提供 metrics 的长期存储。Cortex 一直以来都运行于 Grafana Cloud 和 Weave Cloud 中，为用户提供一种 Prometheus-as-a-Service 的服务。在Weave Cloud 中有一个叫做 Prometheus-book 的东东，听起来跟 Jupyter-Notebook 差不多，功能上确实类似。它给每个用户提供了一个 Prometheus 的界面，每个租户的 metrics 数据都是独立的，用户可以在 Prometheus-book 中查询自己的 metrics。既然是这样自然需要租户间数据隔离以及数据长期存储，提供这种能力的就是 Cortex。Cortex 项目目前贡献者不多，如果觉得 Prometheus 项目贡献的门槛比较高，竞争比较激烈（三哥们实在是太猛了），可以考虑参加这个项目。

另外由于对 Cortex 相关的代码实现还没怎么看完，所以下面部分内容是凭着官方的架构、文档以及一些 issue 里面的讨论写的，有错误请指正。

## Improve Ingester Handover
- 导师: Bryan Boreham (@bboreham)
- 技能: Go

![Architecture](https://github.com/cortexproject/cortex/blob/master/docs/architecture.png?raw=true)

在 Cortex 目前的架构中，大部分的组件都是无状态的，除了 Ingestor 是有状态的（里面需要保存一定时间内的 Prometheus 的 metrics）。Prometheus 通过 remote write API 向 Cortex 写数据，数据点通过 Distributor 分发到 Ingester，具体的分发方式使用的是 DHT（分布式哈希表）。当前的实现中，Distributor 里面保存的 IngesterPool 仅仅允许每12小时伸缩1个Ingestor，可扩展性方面问题比较大。至于为什么是12小时是因为 Ingester 会一直保存12个小时 Prometheus 的 metrics，如果出现相同的 series 就对它们进行压缩，12小时到了再刷到后端的对象存储中去，这样子做的话比较不好应对流量负载动态变化的情况。

总结一下这个 idea 就是对 Distributor 接收到数据分发 -> Ingester 处理 -> 一定时间刷到对象存储里面去，这一整个流程进行优化或者重构使这个架构变得易于扩展。值得一提的是，同样是 Grafana 的 loki 在这部分的架构似乎和 Cortex 一致，如果你完成了这边的 issue，也可以去给 loki 提 PR 了：）。

## Centralized Rate Limiting
- 导师: Bryan Boreham (@bboreham)
- 技能: Go

Cortex 是一个典型的微服务架构，每个组件各自都实现了限流。不过当需要用户对distributor/ingester 进行扩展时，就会受到限制（？这里没怎么看懂）。最好能够实现全局的限流能力，这样同样也能简化配置。参考链接在：[issues/1090](https://github.com/cortexproject/cortex/issues/1090)。在链接中还提到了 youtube 开源的 doorman，doorman 通过后端的一致性存储来实现全局的限流。相关的内容我不是很了解，感兴趣的同学可以自行参考。

## Use etcd in Cortex
- 导师: Bryan Boreham (@bboreham)
- 技能: Go

当前的 Cortex 中使用 consul 存储一些集群的状态信息，包括 ruler、ingester、querier 等组件都需要去查询 consul 中的信息。这个 idea 的主要任务是为存储状态提供可选的 etcd 支持，相对来说比较容易。另外在 Cortex 的 issue 中我也看到有人提议支持 tikv，如果时间充裕也可以一起实现了 ：）。


# 联系作者
由于本人水平有限，有错误欢迎大家指出。如果你对于今年的这些项目或是对于 Prometheus 或是 Kubernetes 感兴趣，或者是你觉得这篇文章对你有帮助，你都可以联系我。我的微信是 eWJfeGExNAo=，你也可以在 [Github](https://github.com/yeya24) 上找到我。

