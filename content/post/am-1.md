---
title: "Alertmanager的告警流程分析（一）"
date: 2019-02-24T15:05:50+08:00
draft: false
description: ""
tags:
- "Alertmanager"
- "Golang"
categories: 
- "Observability"
---
这篇文章会从一些源代码的角度来分析alertmanager告警的这个流程，基于版本0.16.1。由于本人水平有限，分析可能有一些不到位的地方，欢迎大家指出。由于涉及的阶段比较多，我打算分2篇写完。先上一张官方的架构图。

![Alertmanager架构](https://github.com/prometheus/alertmanager/raw/master/doc/arch.svg?sanitize=true)

alertmanager的架构非常清晰：图中左上角的API层包括从prometheus接收到告警以及silence的增删改查。接收到alert和silence之后，各自保存在一个provider对象中存储起来。后续，真正的告警流程由Dispatcher对象来进行处理。Dispatcher对象面会对告警进行分组和聚合，然后每一个告警组（Group）会依次执行Notification Pipeline，这就是基本的告警流程。
## Dispatcher
Dispatcher对象是告警流程的执行者，在main中调用NewDispatcher初始化

```
disp = dispatch.NewDispatcher(alerts, dispatch.NewRoute(conf.Route, nil), pipeline, marker, timeoutFunc, logger)
```

下面是Dispatcher结构体的主要定义

```
type Dispatcher struct {
	route  *Route
	alerts provider.Alerts
	stage  notify.Stage

	marker  types.Marker
	timeout func(time.Duration) time.Duration

	aggrGroups map[*Route]map[model.Fingerprint]*aggrGroup
	mtx        sync.RWMutex

	done   chan struct{}
	ctx    context.Context
	cancel func()

	logger log.Logger
}
```

结构体中，首先是route。route是路由信息，在alertmanager的配置中配置，当收到告警后，匹配路由（其实也就是匹配label），然后执行指定的操作。接下来是alerts。alerts是alertmanager中待发送的告警信息store，prometheus发出的告警就是进入这个store保存下来，然后在Dispatcher中被消费。stage是一个接口，主要实现了exec方法，即流水线的执行动作。在Dispatcher中使用的stage结构是pipelines，pipelines里面包含了多个阶段，也就是架构图右侧的notification pipelines。其次是marker接口，marker接口实现了以下方法，相当于它是一个告警的处理者的抽象，对告警状态进行更新。后面是aggrGroups，在alertmanager中有对告警根据labels进行分组的概念，这就是保存每个组每条告警的map缓存。

```
type Marker interface {
	SetActive(alert model.Fingerprint)
	SetInhibited(alert model.Fingerprint, ids ...string)
	SetSilenced(alert model.Fingerprint, ids ...string)

	Count(...AlertState) int

	Status(model.Fingerprint) AlertStatus
	Delete(model.Fingerprint)

	Unprocessed(model.Fingerprint) bool
	Active(model.Fingerprint) bool
	Silenced(model.Fingerprint) ([]string, bool)
	Inhibited(model.Fingerprint) ([]string, bool)
}
```

下面看下Dispatcher的 run 方法

```
func (d *Dispatcher) run(it provider.AlertIterator) {
	cleanup := time.NewTicker(30 * time.Second)
	defer cleanup.Stop()

	defer it.Close()

	for {
		select {
		case alert, ok := <-it.Next():
			if !ok {
				// Iterator exhausted for some reason.
				if err := it.Err(); err != nil {
					level.Error(d.logger).Log("msg", "Error on alert update", "err", err)
				}
				return
			}

			level.Debug(d.logger).Log("msg", "Received alert", "alert", alert)

			// Log errors but keep trying.
			if err := it.Err(); err != nil {
				level.Error(d.logger).Log("msg", "Error on alert update", "err", err)
				continue
			}

			for _, r := range d.route.Match(alert.Labels) {
				d.processAlert(alert, r)
			}

		case <-cleanup.C:
			d.mtx.Lock()

			for _, groups := range d.aggrGroups {
				for _, ag := range groups {
					if ag.empty() {
						ag.stop()
						delete(groups, ag.fingerprint())
					}
				}
			}

			d.mtx.Unlock()

		case <-d.ctx.Done():
			return
		}
	}
}
```

it对象是一个alerts的迭代器，它的Next()方法是一个channel，从里面消费到alert。整个run方法的核心逻辑很简单，从it.Next()这个channel里面消费到alert之后，查看是否与所配置的路由匹配，如果匹配的话，进入alert的处理方法processAlert。

```
func (d *Dispatcher) processAlert(alert *types.Alert, route *Route) {
	groupLabels := getGroupLabels(alert, route)

	fp := groupLabels.Fingerprint()

	d.mtx.Lock()
	defer d.mtx.Unlock()

	group, ok := d.aggrGroups[route]
	if !ok {
		group = map[model.Fingerprint]*aggrGroup{}
		d.aggrGroups[route] = group
	}

	// If the group does not exist, create it.
	ag, ok := group[fp]
	if !ok {
		ag = newAggrGroup(d.ctx, groupLabels, route, d.timeout, d.logger)
		group[fp] = ag

		go ag.run(func(ctx context.Context, alerts ...*types.Alert) bool {
			_, _, err := d.stage.Exec(ctx, d.logger, alerts...)
			if err != nil {
				level.Error(d.logger).Log("msg", "Notify for alerts failed", "num_alerts", len(alerts), "err", err)
			}
			return err == nil
		})
	}

	ag.insert(alert)
}
```

这个方法的逻辑也是非常的清楚。在这里就是先拿出groupLabel，查看这个labels的hash是否存在，不存在则新建一个aggrGroup。接下来，执行stage的Exec方法。至此，Dispatcher的处理流程结束，已经进入了notification pipelines的阶段。

## pipelines

先看看pipelines的创建，pipelines就是RoutingStage对象，RoutingStage是一个map，它的key是每一个receiver的名字，value是一个multistage对象。在每个multistage中，可以看到都会包括这么几个阶段，GossipSettleStage、InhibitStage、SilenceStage，后面的阶段由createStage函数创建，createStage返回一个FanoutStage，FanoutStage中为每一个receiver都创建了WaitStage、DedupStage、RetryStage、SetNotifierStage，可以 发现这都和上面的架构图是一一对应的。

```
func BuildPipeline(
	confs []*config.Receiver,
	tmpl *template.Template,
	wait func() time.Duration,
	muter types.Muter,
	silences *silence.Silences,
	notificationLog NotificationL执行时都会跑一个Goroutine。og,
	marker types.Marker,
	peer *cluster.Peer,
	logger log.Logger,
) RoutingStage {
	rs := RoutingStage{}

	ms := NewGossipSettleStage(peer)
	is := NewInhibitStage(muter)
	ss := NewSilenceStage(silences, marker)

	for _, rc := range confs {
		rs[rc.Name] = MultiStage{ms, is, ss, createStage(rc, tmpl, wait, notificationLog, logger)}
	}
	return rs
}

func createStage(rc *config.Receiver, tmpl *template.Template, wait func() time.Duration, notificationLog NotificationLog, logger log.Logger) Stage {
	var fs FanoutStage
	for _, i := range BuildReceiverIntegrations(rc, tmpl, logger) {
		recv := &nflogpb.Receiver{
			GroupName:   rc.Name,
			Integration: i.name,
			Idx:         uint32(i.idx),
		}
		var s MultiStage
		s = append(s, NewWaitStage(wait))
		s = append(s, NewDedupStage(i, notificationLog, recv))
		s = append(s, NewRetryStage(i, rc.Name))
		s = append(s, NewSetNotifiesStage(notificationLog, recv))

		fs = append(fs, s)
	}
	return fs
}
```

所有的小阶段都被包装在MultiStage和FanoutStage中，先来看看它们的exec方法是怎么实现的。在MultiStage中，exec很简单，遍历自身的Stage，依次执行每个阶段各自的exec方法，根据之前所说pipelines中的MultiStage中，保存了4个阶段，GossipSettle、Inhibit、Silence、FanoutStage，这四个阶段会依次调用exec方法。后面看看FanoutStage，它的类型其实与MultiStage相同，不同的是它的每个阶段都是通过一个Goroutine来执行。其实也比较好理解，GossipSettle、Inhibit、Silence这几个阶段属于发送前的准备工作，开启Gossip，禁止某些警报以及沉默某些警报，后面的几个阶段才是真正发送警报的流程。包括了Wait、Dedup、Retry以及SetNotify。

```
type MultiStage []Stage

func (ms MultiStage) Exec(ctx context.Context, l log.Logger, alerts ...*types.Alert) (context.Context, []*types.Alert, error) {
	var err error
	for _, s := range ms {
		if len(alerts) == 0 {
			return ctx, nil, nil
		}

		ctx, alerts, err = s.Exec(ctx, l, alerts...)
		if err != nil {
			return ctx, nil, err
		}
	}
	return ctx, alerts, nil
}

type FanoutStage []Stage

func (fs FanoutStage) Exec(ctx context.Context, l log.Logger, alerts ...*types.Alert) (context.Context, []*types.Alert, error) {
	var (
		wg sync.WaitGroup
		me types.MultiError
	)
	wg.Add(len(fs))

	for _, s := range fs {
		go func(s Stage) {
			if _, _, err := s.Exec(ctx, l, alerts...); err != nil {
				me.Add(err)
				level.Error(l).Log("msg", "Error on notify", "err", err)
			}
			wg.Done()
		}(s)
	}
	wg.Wait()

	if me.Len() > 0 {
		return ctx, alerts, &me
	}
	return ctx, alerts, nil
}
```

## GossipSettleStage
GossipSettle是第一个阶段，很简单，这个阶段的主要作用就是执行  n.peer.WaitReady方法，这个方法与alertmanager的高可用有关，对于每一个peer有一个readyChannel，当某一个peer处于unReady状态，就一直阻塞，ready之后就进入下一个阶段

```
type GossipSettleStage struct {
	peer *cluster.Peer
}

func NewGossipSettleStage(p *cluster.Peer) *GossipSettleStage {
	return &GossipSettleStage{peer: p}
}

func (n *GossipSettleStage) Exec(ctx context.Context, l log.Logger, alerts ...*types.Alert) (context.Context, []*types.Alert, error) {
	if n.peer != nil {
		n.peer.WaitReady()
	}
	return ctx, alerts, nil
}
```

## InhibitStage
下一个阶段是InhibitStage，顾名思义，对告警执行抑制。抑制在alertmanager中的配置文件中配置，满足某些label的条件则抑制该告警。下面的实现中是对alerts进行遍历，如果不满足抑制的匹配规则，则加入filtered ，进入下一个阶段；而匹配上的相当于被丢弃，不被处理

```
// InhibitStage filters alerts through an inhibition muter.
type InhibitStage struct {
	muter types.Muter
}

func NewInhibitStage(m types.Muter) *InhibitStage {
	return &InhibitStage{muter: m}
}

func (n *InhibitStage) Exec(ctx context.Context, l log.Logger, alerts ...*types.Alert) (context.Context, []*types.Alert, error) {
	var filtered []*types.Alert
	for _, a := range alerts {
		if !n.muter.Mutes(a.Labels) {
			filtered = append(filtered, a)
		}
	}

	return ctx, filtered, nil
}
```

### SilenceStage
Silence阶段是判断是否需要对告警执行静默处理。silence这个东西比较复杂，与inhibit不同，inhibit是在alertmanager中的配置文件进行配置，在3个AM组成的集群中，是允许不同的AM有不同的inhibit配置。而silence则是通过AM的API进行创建删除。在高可用的情况下，总不能某个AM的某条告警被silence掉了，而另外两个AM由于本身没有创建silence，就把告警发送出去了，这就无法起到高可用的作用。所以silence信息一样会通过gossip的消息在AM集群之间进行同步。由于gossip是一个最终一致性协议，所以肯定在中间会出现数据一致性的问题，让我们看看在发送silence的时候是怎么处理的。

```
// SilenceStage filters alerts through a silence muter.
type SilenceStage struct {
	silences *silence.Silences
	marker   types.Marker
}

func NewSilenceStage(s *silence.Silences, mk types.Marker) *SilenceStage {
	return &SilenceStage{
		silences: s,
		marker:   mk,
	}
}

func (n *SilenceStage) Exec(ctx context.Context, l log.Logger, alerts ...*types.Alert) (context.Context, []*types.Alert, error) {
	var filtered []*types.Alert
	for _, a := range alerts {
		sils, err := n.silences.Query(
			silence.QState(types.SilenceStateActive),
			silence.QMatches(a.Labels),
		)
		if err != nil {
			level.Error(l).Log("msg", "Querying silences failed", "err", err)
		}

		if len(sils) == 0 {
			filtered = append(filtered, a)
			n.marker.SetSilenced(a.Labels.Fingerprint())
		} else {
			ids := make([]string, len(sils))
			for i, s := range sils {
				ids[i] = s.Id
			}
			n.marker.SetSilenced(a.Labels.Fingerprint(), ids...)
		}
	}

	return ctx, filtered, nil
}
```

首先遍历所有的alerts，根据每条告警，查询silences的store中是否有目前状态是active（即没有过期的），且labels与该告警匹配的silence，如果有，那么通过marker的SetSilenced方法将指定labels的告警设置为silenced状态。方法的关键就在于silences的查询过程，那就看看silences.Query这个方法

```
func (s *Silences) Query(params ...QueryParam) ([]*pb.Silence, error) {
	start := time.Now()
	s.metrics.queriesTotal.Inc()

	sils, err := func() ([]*pb.Silence, error) {
		q := &query{}
		for _, p := range params {
			if err := p(q); err != nil {
				return nil, err
			}
		}
		return s.query(q, s.now())
	}()
	if err != nil {
		s.metrics.queryErrorsTotal.Inc()
	}
	s.metrics.queryDuration.Observe(time.Since(start).Seconds())
	return sils, err
}

func (s *Silences) query(q *query, now time.Time) ([]*pb.Silence, error) {
	// If we have an ID constraint, all silences are our base set.
	// This and the use of post-filter functions is the
	// the trivial solution for now.
	var res []*pb.Silence

	s.mtx.Lock()
	defer s.mtx.Unlock()

	if q.ids != nil {
		for _, id := range q.ids {
			if s, ok := s.st[id]; ok {
				res = append(res, s.Silence)
			}
		}
	} else {
		for _, sil := range s.st {
			res = append(res, sil.Silence)
		}
	}

	var resf []*pb.Silence
	for _, sil := range res {
		remove := false
		for _, f := range q.filters {
			ok, err := f(sil, s, now)
			if err != nil {
				return nil, err
			}
			if !ok {
				remove = true
				break
			}
		}
		if !remove {
			resf = append(resf, cloneSilence(sil))
		}
	}

	return resf, nil
}
```

这两个方法比较长，silences.Query这个方法实际调用的是query方法，所以我们直接看query方法。query方法中，首先看参数的query里面有没有带上id限制，如果有指定id，就从s.st这个保存了当前所有silence的map找出指定id的silence。如果没有指定，则将map里面的所有silence加入res这个切片中，在这个方法中肯定是没有指定id，所以会先得到所有的silence。之后对res这个切片进行遍历，执行filter进行过滤。所谓的过滤即是比较之前调用Query时候指定的两个参数：状态为active，且能匹配上警报的标签。通过filter过滤，如果不满足，则remove标志为true，并break退出；如果一直都满足，则将这个silence对象加入到resf这个切片中，返回回去。可以看出，在上述silence的查询过程中，是有可能出现多个AM实例中silence状态不同的情况的，可能会出现数据一致性问题。

```
sils, err := n.silences.Query(
			silence.QState(types.SilenceStateActive),
			silence.QMatches(a.Labels),
		)
```
