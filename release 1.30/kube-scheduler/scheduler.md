### runCommand

#### Setup
```go
// 
// Setup creates a completed config and a scheduler based on the command args and options
func Setup(ctx context.Context, opts *options.Options, outOfTreeRegistryOptions ...Option) (*schedulerserverconfig.CompletedConfig, *scheduler.Scheduler, error) {
	// 生成默认的kubeSchedulerConfiguration
    if cfg, err := latest.Default(); err != nil {
		return nil, nil, err
	} else {
		opts.ComponentConfig = cfg
	}

    // 配置参数校验，如必填字段、值必须为正数、监听端口、证书、http请求头、webhook重试次数等，如果校验不通过，组装后返回所有的错误
	if errs := opts.Validate(); len(errs) > 0 {
		return nil, nil, utilerrors.NewAggregate(errs)
	}

    // 根据前面的配置文件和默认生成的参数等组装schedulerApp的配置实例，生成kubeConfig并通过kubeConfig创建client，并根据client初始化Informer
	c, err := opts.Config(ctx)
	if err != nil {
		return nil, nil, err
	}

    // 填充信息，这里主要是生成高权限bearerToken
	// Get the completed config
	cc := c.Complete()

    // 这一段往上面追溯没看到传参，主要用于外部集成资源，本文中没用到
	outOfTreeRegistry := make(runtime.Registry)
	for _, option := range outOfTreeRegistryOptions {
		if err := option(outOfTreeRegistry); err != nil {
			return nil, nil, err
		}
	}

	recorderFactory := getRecorderFactory(&cc)
	completedProfiles := make([]kubeschedulerconfig.KubeSchedulerProfile, 0)
    // 创建scheduler实例，这里面初始化了各类缓存，如 nodeInfo 和 scheduler 本身的缓存。 以及各类内置插件，用于调度时打分。
	// Create the scheduler.
	sched, err := scheduler.New(ctx,
		cc.Client,
		cc.InformerFactory,
		cc.DynInformerFactory,
		recorderFactory,
		scheduler.WithComponentConfigVersion(cc.ComponentConfig.TypeMeta.APIVersion),
		scheduler.WithKubeConfig(cc.KubeConfig),
		scheduler.WithProfiles(cc.ComponentConfig.Profiles...),
		scheduler.WithPercentageOfNodesToScore(cc.ComponentConfig.PercentageOfNodesToScore),
		scheduler.WithFrameworkOutOfTreeRegistry(outOfTreeRegistry),
		scheduler.WithPodMaxBackoffSeconds(cc.ComponentConfig.PodMaxBackoffSeconds),
		scheduler.WithPodInitialBackoffSeconds(cc.ComponentConfig.PodInitialBackoffSeconds),
		scheduler.WithPodMaxInUnschedulablePodsDuration(cc.PodMaxInUnschedulablePodsDuration),
		scheduler.WithExtenders(cc.ComponentConfig.Extenders...),
		scheduler.WithParallelism(cc.ComponentConfig.Parallelism),
		scheduler.WithBuildFrameworkCapturer(func(profile kubeschedulerconfig.KubeSchedulerProfile) {
			// Profiles are processed during Framework instantiation to set default plugins and configurations. Capturing them for logging
			completedProfiles = append(completedProfiles, profile)
		}),
	)
	if err != nil {
		return nil, nil, err
	}
    // 将完整的配置文件和组件的配置文件写入指定路径
	if err := options.LogOrWriteConfig(klog.FromContext(ctx), opts.WriteConfigTo, &cc.ComponentConfig, completedProfiles); err != nil {
		return nil, nil, err
	}

	return &cc, sched, nil
}

```
#### run
```go
// cmd/kube-scheduler/app/server.go:150
// 根据上面setup生成的配置文件和scheduler实例运行scheduler，只有在上下文结束时或出现错误时返回
// Run executes the scheduler based on the given configuration. It only returns on error or when context is done.
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
	logger := klog.FromContext(ctx)

	// To help debugging, immediately log version
	logger.Info("Starting Kubernetes Scheduler", "version", version.Get())

	logger.Info("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

    // 组件配置文件handler注册，可以通过/configz 查看配置与修改配置
	// Configz registration.
	if cz, err := configz.New("componentconfig"); err == nil {
		cz.Set(cc.ComponentConfig)
	} else {
		return fmt.Errorf("unable to register configz: %s", err)
	}

	// Start events processing pipeline.
	cc.EventBroadcaster.StartRecordingToSink(ctx.Done())
	defer cc.EventBroadcaster.Shutdown()

	// Setup healthz checks.
	var checks []healthz.HealthChecker
	if cc.ComponentConfig.LeaderElection.LeaderElect {
		checks = append(checks, cc.LeaderElection.WatchDog)
	}

    // leader选举
	waitingForLeader := make(chan struct{})
	isLeader := func() bool {
		select {
		case _, ok := <-waitingForLeader:
			// if channel is closed, we are leading
			return !ok
		default:
			// channel is open, we are waiting for a leader
			return false
		}
	}

    // 暴露指标
	// Start up the healthz server.
	if cc.SecureServing != nil {
		handler := buildHandlerChain(newHealthzAndMetricsHandler(&cc.ComponentConfig, cc.InformerFactory, isLeader, checks...), cc.Authentication.Authenticator, cc.Authorization.Authorizer)
		// TODO: handle stoppedCh and listenerStoppedCh returned by c.SecureServing.Serve
		if _, _, err := cc.SecureServing.Serve(handler, 0, ctx.Done()); err != nil {
			// fail early for secure handlers, removing the old error loop from above
			return fmt.Errorf("failed to start secure server: %v", err)
		}
	}

	startInformersAndWaitForSync := func(ctx context.Context) {
		// 启动所有 informer。InformerFactory 是一个用于创建和管理 informer 的工厂对象。informer 是 kubernetes 中用来监听资源变化的组件。
		// Start all informers.
		cc.InformerFactory.Start(ctx.Done())
		// DynInformerFactory can be nil in tests.
		if cc.DynInformerFactory != nil {
			// DynInformerFactory 是动态 informer 的工厂，通常用于处理自定义资源（Custom Resource）。
			cc.DynInformerFactory.Start(ctx.Done())
		}
		// WaitForCacheSync 方法会阻塞当前执行，直到所有由 InformerFactory 管理的 informer 的缓存都与 Kubernetes API 服务器的数据同步完成。这确保了在同步完成之前，不会继续后续操作。
		// Wait for all caches to sync before scheduling.
		cc.InformerFactory.WaitForCacheSync(ctx.Done())
		// DynInformerFactory can be nil in tests.
		if cc.DynInformerFactory != nil {
			cc.DynInformerFactory.WaitForCacheSync(ctx.Done())
		}
		// 确保在开始调度或处理 Kubernetes 资源事件之前，所有相关的 informer 和处理程序都已经启动并且缓存已同步。这里的处理程序指负责处理 informer 中资源事件的组件。
		// Wait for all handlers to sync (all items in the initial list delivered) before scheduling.
		if err := sched.WaitForHandlersSync(ctx); err != nil {
			logger.Error(err, "waiting for handlers to sync")
		}

		logger.V(3).Info("Handlers synced")
	}
	// 如果选举没有开启，直接启动 informer 并等待缓存同步
	if !cc.ComponentConfig.DelayCacheUntilActive || cc.LeaderElection == nil {
		startInformersAndWaitForSync(ctx)
	}
	// If leader election is enabled, runCommand via LeaderElector until done and exit.
	if cc.LeaderElection != nil {
		cc.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
			// 如果是leader，则跟上面的启动流程一样，注册各类内置的controller，然后启动
			OnStartedLeading: func(ctx context.Context) {
				close(waitingForLeader)
				if cc.ComponentConfig.DelayCacheUntilActive {
					logger.Info("Starting informers and waiting for sync...")
					startInformersAndWaitForSync(ctx)
					logger.Info("Sync completed")
				}
				sched.Run(ctx)
			},
			// 如果不是leader，则退出
			OnStoppedLeading: func() {
				select {
				case <-ctx.Done():
					// We were asked to terminate. Exit 0.
					logger.Info("Requested to terminate, exiting")
					os.Exit(0)
				default:
					// We lost the lock.
					logger.Error(nil, "Leaderelection lost")
					klog.FlushAndExit(klog.ExitFlushTimeout, 1)
				}
			},
		}
		// 启动选举
		leaderElector, err := leaderelection.NewLeaderElector(*cc.LeaderElection)
		if err != nil {
			return fmt.Errorf("couldn't create leader elector: %v", err)
		}
		// 这个后面再看
		leaderElector.Run(ctx)

		return fmt.Errorf("lost lease")
	}

	// Leader election is disabled, so runCommand inline until done.
	close(waitingForLeader)
	// 如果没有leader选举，则直接运行
	sched.Run(ctx)
	return fmt.Errorf("finished without leader elect")
}
```
### Scheduler.Run
```go
// 该方法监听资源变化并开始调度工作，阻塞执行，只有在上下文结束时或出现错误时返回
// Run begins watching and scheduling. It starts scheduling and blocked until the context is done.
func (sched *Scheduler) Run(ctx context.Context) {
	logger := klog.FromContext(ctx)
	// 看下面SchedulingQueue.Run
	sched.SchedulingQueue.Run(logger)

	// 看下面ScheduleOne
	// We need to start scheduleOne loop in a dedicated goroutine,
	// because scheduleOne function hangs on getting the next item
	// from the SchedulingQueue.
	// If there are no new pods to schedule, it will be hanging there
	// and if done in this goroutine it will be blocking closing
	// SchedulingQueue, in effect causing a deadlock on shutdown.
	go wait.UntilWithContext(ctx, sched.ScheduleOne, 0)

	<-ctx.Done()
	sched.SchedulingQueue.Close()

	// If the plugins satisfy the io.Closer interface, they are closed.
	err := sched.Profiles.Close()
	if err != nil {
		logger.Error(err, "Failed to close plugins")
	}
}
```
#### SchedulingQueue.Run
这里的 SchedulingQueue 是一个接口类型，往上找找初始化步骤，在 scheduler.New() 中会生成这个类型的实例。
```go
// New returns a Scheduler
func New(ctx context.Context,
	client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	opts ...Option) (*Scheduler, error) {
	...
	podQueue := internalqueue.NewSchedulingQueue(
		profiles[options.profiles[0].SchedulerName].QueueSortFunc(),
		informerFactory,
		internalqueue.WithPodInitialBackoffDuration(time.Duration(options.podInitialBackoffSeconds)*time.Second),
		internalqueue.WithPodMaxBackoffDuration(time.Duration(options.podMaxBackoffSeconds)*time.Second),
		internalqueue.WithPodLister(podLister),
		internalqueue.WithPodMaxInUnschedulablePodsDuration(options.podMaxInUnschedulablePodsDuration),
		internalqueue.WithPreEnqueuePluginMap(preEnqueuePluginMap),
		internalqueue.WithQueueingHintMapPerProfile(queueingHintsPerProfile),
		internalqueue.WithPluginMetricsSamplePercent(pluginMetricsSamplePercent),
		internalqueue.WithMetricsRecorder(*metricsRecorder),
	)
	...
	sched := &Scheduler{
		Cache:                    schedulerCache,
		client:                   client,
		nodeInfoSnapshot:         snapshot,
		percentageOfNodesToScore: options.percentageOfNodesToScore,
		Extenders:                extenders,
		StopEverything:           stopEverything,
		SchedulingQueue:          podQueue,
		Profiles:                 profiles,
		logger:                   logger,
	}	
	...
}
```

#### PriorityQueue
优先级队列，队列头是优先级最高的待调度 Pod。该结构体中的 activeQ 和 podBackoffQ 两个队列都是堆结构。
- activeQ 用于存储需要被调度的 Pod
- podBackoffQ 用于存储那些之前因为某些原因无法调度被放置到 unschedulablePods 中的 pod。当这些 pod 的补偿周期结束时，它们会被移动到 activeQ 中
- unschedulablePods 用于存储那些已经尝试调度但因条件不满足而决定不调度的 pod。
```go
type PriorityQueue struct {
	*nominator

	stop  chan struct{}
	clock clock.Clock

	// pod initial backoff duration.
	podInitialBackoffDuration time.Duration
	// pod maximum backoff duration.
	podMaxBackoffDuration time.Duration
	// the maximum time a pod can stay in the unschedulablePods.
	podMaxInUnschedulablePodsDuration time.Duration

	cond sync.Cond

	// inFlightPods holds the UID of all pods which have been popped out for which Done
	// hasn't been called yet - in other words, all pods that are currently being
	// processed (being scheduled, in permit, or in the binding cycle).
	//
	// The values in the map are the entry of each pod in the inFlightEvents list.
	// The value of that entry is the *v1.Pod at the time that scheduling of that
	// pod started, which can be useful for logging or debugging.
	inFlightPods map[types.UID]*list.Element

	// inFlightEvents holds the events received by the scheduling queue
	// (entry value is clusterEvent) together with in-flight pods (entry
	// value is *v1.Pod). Entries get added at the end while the mutex is
	// locked, so they get serialized.
	//
	// The pod entries are added in Pop and used to track which events
	// occurred after the pod scheduling attempt for that pod started.
	// They get removed when the scheduling attempt is done, at which
	// point all events that occurred in the meantime are processed.
	//
	// After removal of a pod, events at the start of the list are no
	// longer needed because all of the other in-flight pods started
	// later. Those events can be removed.
	inFlightEvents *list.List

	// activeQ is heap structure that scheduler actively looks at to find pods to
	// schedule. Head of heap is the highest priority pod.
	activeQ *heap.Heap
	// podBackoffQ is a heap ordered by backoff expiry. Pods which have completed backoff
	// are popped from this heap before the scheduler looks at activeQ
	podBackoffQ *heap.Heap
	// unschedulablePods holds pods that have been tried and determined unschedulable.
	unschedulablePods *UnschedulablePods
	// schedulingCycle represents sequence number of scheduling cycle and is incremented
	// when a pod is popped.
	schedulingCycle int64
	// moveRequestCycle caches the sequence number of scheduling cycle when we
	// received a move request. Unschedulable pods in and before this scheduling
	// cycle will be put back to activeQueue if we were trying to schedule them
	// when we received move request.
	// TODO: this will be removed after SchedulingQueueHint goes to stable and the feature gate is removed.
	moveRequestCycle int64

	// preEnqueuePluginMap is keyed with profile name, valued with registered preEnqueue plugins.
	preEnqueuePluginMap map[string][]framework.PreEnqueuePlugin
	// queueingHintMap is keyed with profile name, valued with registered queueing hint functions.
	queueingHintMap QueueingHintMapPerProfile

	// closed indicates that the queue is closed.
	// It is mainly used to let Pop() exit its control loop while waiting for an item.
	closed bool

	nsLister listersv1.NamespaceLister

	metricsRecorder metrics.MetricAsyncRecorder
	// pluginMetricsSamplePercent is the percentage of plugin metrics to be sampled.
	pluginMetricsSamplePercent int

	// isSchedulingQueueHintEnabled indicates whether the feature gate for the scheduling queue is enabled.
	isSchedulingQueueHintEnabled bool
}

// NewSchedulingQueue initializes a priority queue as a new scheduling queue.
func NewSchedulingQueue(
	lessFn framework.LessFunc,
	informerFactory informers.SharedInformerFactory,
	opts ...Option) SchedulingQueue {
	return NewPriorityQueue(lessFn, informerFactory, opts...)
}

// NewPriorityQueue creates a PriorityQueue object.
func NewPriorityQueue(
	lessFn framework.LessFunc,
	informerFactory informers.SharedInformerFactory,
	opts ...Option,
) *PriorityQueue {
	options := defaultPriorityQueueOptions
	if options.podLister == nil {
		options.podLister = informerFactory.Core().V1().Pods().Lister()
	}
	for _, opt := range opts {
		opt(&options)
	}

	// 优先级比较方法
	comp := func(podInfo1, podInfo2 interface{}) bool {
		pInfo1 := podInfo1.(*framework.QueuedPodInfo)
		pInfo2 := podInfo2.(*framework.QueuedPodInfo)
		return lessFn(pInfo1, pInfo2)
	}

	pq := &PriorityQueue{
		nominator:                         newPodNominator(options.podLister),
		clock:                             options.clock,
		stop:                              make(chan struct{}),
		podInitialBackoffDuration:         options.podInitialBackoffDuration,
		podMaxBackoffDuration:             options.podMaxBackoffDuration,
		podMaxInUnschedulablePodsDuration: options.podMaxInUnschedulablePodsDuration,
		activeQ:                           heap.NewWithRecorder(podInfoKeyFunc, comp, metrics.NewActivePodsRecorder()),
		unschedulablePods:                 newUnschedulablePods(metrics.NewUnschedulablePodsRecorder(), metrics.NewGatedPodsRecorder()),
		inFlightPods:                      make(map[types.UID]*list.Element),
		inFlightEvents:                    list.New(),
		preEnqueuePluginMap:               options.preEnqueuePluginMap,
		queueingHintMap:                   options.queueingHintMap,
		metricsRecorder:                   options.metricsRecorder,
		pluginMetricsSamplePercent:        options.pluginMetricsSamplePercent,
		moveRequestCycle:                  -1,
		isSchedulingQueueHintEnabled:      utilfeature.DefaultFeatureGate.Enabled(features.SchedulerQueueingHints),
	}
	pq.cond.L = &pq.lock
	pq.podBackoffQ = heap.NewWithRecorder(podInfoKeyFunc, pq.podsCompareBackoffCompleted, metrics.NewBackoffPodsRecorder())
	pq.nsLister = informerFactory.Core().V1().Namespaces().Lister()

	return pq
}
```
#### Run
定期将backoff 或 unschedulable 队列中的pod移动到active队列
```go
// Run starts the goroutine to pump from podBackoffQ to activeQ
func (p *PriorityQueue) Run(logger klog.Logger) {
	go wait.Until(func() {
		// 将 backoff 队列中的 pod 移动到 activeQ
		p.flushBackoffQCompleted(logger)
	}, 1.0*time.Second, p.stop)
	go wait.Until(func() {
		// 将 unschedulablePods 队列中的时长超过 podMaxInUnschedulablePodsDuration 的 pod 移动到 activeQ 或者 podBackoffQ
		p.flushUnschedulablePodsLeftover(logger)
	}, 30*time.Second, p.stop)
}

// flushBackoffQCompleted Moves all pods from backoffQ which have completed backoff in to activeQ
func (p *PriorityQueue) flushBackoffQCompleted(logger klog.Logger) {
	p.lock.Lock()
	defer p.lock.Unlock()
	activated := false
	for {
		// 获取 backoff 队列中的第一个 pod
		rawPodInfo := p.podBackoffQ.Peek()
		if rawPodInfo == nil {
			break
		}
		pInfo := rawPodInfo.(*framework.QueuedPodInfo)
		pod := pInfo.Pod
		// 检查该 pod 在 backoff 队列中是否已经存在足够长时间，如果没有，直接跳过，等待下一秒再判断
		if p.isPodBackingoff(pInfo) {
			break
		}
		// 堆中弹出第一个 pod
		_, err := p.podBackoffQ.Pop()
		if err != nil {
			logger.Error(err, "Unable to pop pod from backoff queue despite backoff completion", "pod", klog.KObj(pod))
			break
		}
		// 加入到 active 队列，会尝试进行预调度插件检查，如果所有检查都通过，则将该 pod 移动到 active 队列，否则，移到 unschedulablePods 队列
		if added, _ := p.addToActiveQ(logger, pInfo); added {
			logger.V(5).Info("Pod moved to an internal scheduling queue", "pod", klog.KObj(pod), "event", BackoffComplete, "queue", activeQ)
			metrics.SchedulerQueueIncomingPods.WithLabelValues("active", BackoffComplete).Inc()
			activated = true
		}
	}

	if activated {
		// 如果有 pod 被移动到 active 队列，且之前 active 队列为空，则唤醒队列
		p.cond.Broadcast()
	}
}

func (p *PriorityQueue) flushUnschedulablePodsLeftover(logger klog.Logger) {
	p.lock.Lock()
	defer p.lock.Unlock()

	var podsToMove []*framework.QueuedPodInfo
	currentTime := p.clock.Now()
	// 检查时间戳是否超过了 podMaxInUnschedulablePodsDuration
	for _, pInfo := range p.unschedulablePods.podInfoMap {
		lastScheduleTime := pInfo.Timestamp
		if currentTime.Sub(lastScheduleTime) > p.podMaxInUnschedulablePodsDuration {
			podsToMove = append(podsToMove, pInfo)
		}
	}

	if len(podsToMove) > 0 {
		p.movePodsToActiveOrBackoffQueue(logger, podsToMove, UnschedulableTimeout, nil, nil)
	}
}

// 将 unschedulablePods 队列中的时长超过 podMaxInUnschedulablePodsDuration 的 pod 移动到 activeQ 或者 podBackoffQ
func (p *PriorityQueue) movePodsToActiveOrBackoffQueue(logger klog.Logger, podInfoList []*framework.QueuedPodInfo, event framework.ClusterEvent, oldObj, newObj interface{}) {
	// 如果没有调度插件对传入的集群事件进行拦截，直接返回，这里的事件是调度超时事件
	if !p.isEventOfInterest(logger, event) {
		// No plugin is interested in this event.
		return
	}

	activated := false
	for _, pInfo := range podInfoList {
		// 判断pod是否需要被调度
		schedulingHint := p.isPodWorthRequeuing(logger, pInfo, event, oldObj, newObj)
		// 如果不需要被调度，则直接跳过
		if schedulingHint == queueSkip {
			// QueueingHintFn determined that this Pod isn't worth putting to activeQ or backoffQ by this event.
			logger.V(5).Info("Event is not making pod schedulable", "pod", klog.KObj(pInfo.Pod), "event", event.Label)
			continue
		}
		// 从 unschedulablePods 队列中移除该pod
		p.unschedulablePods.delete(pInfo.Pod, pInfo.Gated)
		// 根据调度提示，将该pod移动到对应的队列
		queue := p.requeuePodViaQueueingHint(logger, pInfo, schedulingHint, event.Label)
		logger.V(4).Info("Pod moved to an internal scheduling queue", "pod", klog.KObj(pInfo.Pod), "event", event.Label, "queue", queue, "hint", schedulingHint)
		if queue == activeQ {
			activated = true
		}
	}

	p.moveRequestCycle = p.schedulingCycle

	if p.isSchedulingQueueHintEnabled && len(p.inFlightPods) != 0 {
		logger.V(5).Info("Event received while pods are in flight", "event", event.Label, "numPods", len(p.inFlightPods))
		// AddUnschedulableIfNotPresent might get called for in-flight Pods later, and in
		// AddUnschedulableIfNotPresent we need to know whether events were
		// observed while scheduling them.
		p.inFlightEvents.PushBack(&clusterEvent{
			event:  event,
			oldObj: oldObj,
			newObj: newObj,
		})
	}

	if activated {
		p.cond.Broadcast()
	}
}

func (p *PriorityQueue) isPodWorthRequeuing(logger klog.Logger, pInfo *framework.QueuedPodInfo, event framework.ClusterEvent, oldObj, newObj interface{}) queueingStrategy {
	// 如果这个集合为空，表示没有专门处理失败或等待中的插件，因此这个Pod值得重新排队，策略为 queueAfterBackoff
	rejectorPlugins := pInfo.UnschedulablePlugins.Union(pInfo.PendingPlugins)
	if rejectorPlugins.Len() == 0 {
		logger.V(6).Info("Worth requeuing because no failed plugins", "pod", klog.KObj(pInfo.Pod))
		return queueAfterBackoff
	}

	// 如果事件的Resource == WildCard 并且事件的ActionType == All，这里传入的event是UnschedulableTimeout，满足条件，直接返回 queueAfterBackoff
	if event.IsWildCard() {
		// If the wildcard event is special one as someone wants to force all Pods to move to activeQ/backoffQ.
		// We return queueAfterBackoff in this case, while resetting all blocked plugins.
		logger.V(6).Info("Worth requeuing because the event is wildcard", "pod", klog.KObj(pInfo.Pod))
		return queueAfterBackoff
	}

	// 从 queueingHintMap 中查找与Pod的调度器名字（SchedulerName）相关的提示函数。如果没有找到对应的提示映射，记录错误并返回 queueAfterBackoff 策略。
	hintMap, ok := p.queueingHintMap[pInfo.Pod.Spec.SchedulerName]
	if !ok {
		// shouldn't reach here unless bug.
		logger.Error(nil, "No QueueingHintMap is registered for this profile", "profile", pInfo.Pod.Spec.SchedulerName, "pod", klog.KObj(pInfo.Pod))
		return queueAfterBackoff
	}

	// 这里一般的调度逻辑都走完了，只有特殊的调度逻辑需要特殊处理
	pod := pInfo.Pod
	queueStrategy := queueSkip
	for eventToMatch, hintfns := range hintMap {
		if !eventToMatch.Match(event) {
			continue
		}
		// 遍历 hintMap，匹配当前事件 event，检查是否存在匹配的提示函数。
		for _, hintfn := range hintfns {
			if !rejectorPlugins.Has(hintfn.PluginName) {
				// skip if it's not hintfn from rejectorPlugins.
				continue
			}
			// 如果匹配成功，调用提示函数 QueueingHintFn，根据返回值和错误进行不同的策略决定。
			hint, err := hintfn.QueueingHintFn(logger, pod, oldObj, newObj)
			if err != nil {
				// If the QueueingHintFn returned an error, we should treat the event as Queue so that we can prevent
				// the Pod from being stuck in the unschedulable pod pool.
				oldObjMeta, newObjMeta, asErr := util.As[klog.KMetadata](oldObj, newObj)
				if asErr != nil {
					logger.Error(err, "QueueingHintFn returns error", "event", event, "plugin", hintfn.PluginName, "pod", klog.KObj(pod))
				} else {
					logger.Error(err, "QueueingHintFn returns error", "event", event, "plugin", hintfn.PluginName, "pod", klog.KObj(pod), "oldObj", klog.KObj(oldObjMeta), "newObj", klog.KObj(newObjMeta))
				}
				hint = framework.Queue
			}
			if hint == framework.QueueSkip {
				continue
			}
			// 如果返回 queueImmediately，表示这个Pod应立即重新排队，这是最高优先级的策略。
			if pInfo.PendingPlugins.Has(hintfn.PluginName) {
				// interprets Queue from the Pending plugin as queueImmediately.
				// We can return immediately because queueImmediately is the highest priority.
				return queueImmediately
			}

			// interprets Queue from the unschedulable plugin as queueAfterBackoff.
			if pInfo.PendingPlugins.Len() == 0 {
				// We can return immediately because no Pending plugins, which only can make queueImmediately, registered in this Pod,
				// and queueAfterBackoff is the second highest priority.
				return queueAfterBackoff
			}

			// We can't return immediately because there are some Pending plugins registered in this Pod.
			// We need to check if those plugins return Queue or not and if they do, we return queueImmediately.
			queueStrategy = queueAfterBackoff
		}
	}
	// 最终，返回决定的策略。可能是 queueImmediately、queueAfterBackoff 或 queueSkip。
	return queueStrategy
}

// 根据调度提示，将该pod移动到对应的队列
func (p *PriorityQueue) requeuePodViaQueueingHint(logger klog.Logger, pInfo *framework.QueuedPodInfo, strategy queueingStrategy, event string) string {
	// 如果策略是 queueSkip，则在 unschedulablePods 队列中添加或更新该pod
	if strategy == queueSkip {
		p.unschedulablePods.addOrUpdate(pInfo)
		metrics.SchedulerQueueIncomingPods.WithLabelValues("unschedulable", event).Inc()
		return unschedulablePods
	}

	pod := pInfo.Pod
	// 如果策略是 queueAfterBackoff 并且 pod 处于 backingoff 状态，则将该pod放入 backoff 队列
	if strategy == queueAfterBackoff && p.isPodBackingoff(pInfo) {
		if err := p.podBackoffQ.Add(pInfo); err != nil {
			logger.Error(err, "Error adding pod to the backoff queue, queue this Pod to unschedulable pod pool", "pod", klog.KObj(pod))
			// 如果加入 backoff 队列失败，则将该pod放入 unschedulablePods 队列
			p.unschedulablePods.addOrUpdate(pInfo)
			return unschedulablePods
		}

		metrics.SchedulerQueueIncomingPods.WithLabelValues("backoff", event).Inc()
		return backoffQ
	}

	// 走到这段逻辑的pod 要么是 queueImmediately，要么是 pod 不处于 backingoff 状态，直接加入 activeQ
	// Reach here if schedulingHint is QueueImmediately, or schedulingHint is Queue but the pod is not backing off.
	added, err := p.addToActiveQ(logger, pInfo)
	if err != nil {
		logger.Error(err, "Error adding pod to the active queue, queue this Pod to unschedulable pod pool", "pod", klog.KObj(pod))
	}
	if added {
		metrics.SchedulerQueueIncomingPods.WithLabelValues("active", event).Inc()
		return activeQ
	}
	if pInfo.Gated {
		// In case the pod is gated, the Pod is pushed back to unschedulable Pods pool in addToActiveQ.
		return unschedulablePods
	}
	// 如果加入 active 队列失败，则将该pod放入 unschedulablePods 队列
	p.unschedulablePods.addOrUpdate(pInfo)
	metrics.SchedulerQueueIncomingPods.WithLabelValues("unschedulable", ScheduleAttemptFailure).Inc()
	return unschedulablePods
}
```
### ScheduleOne
```go
func (sched *Scheduler) ScheduleOne(ctx context.Context) {
	logger := klog.FromContext(ctx)
	// 获取下一个待调度的Pod，这个pod是上面优先级队列中activeQ 中的第一个pod
	podInfo, err := sched.NextPod(logger)
	if err != nil {
		logger.Error(err, "Error while retrieving next pod from scheduling queue")
		return
	}
	// 如果当前没有需要调度的pod，直接返回
	// pod could be nil when schedulerQueue is closed
	if podInfo == nil || podInfo.Pod == nil {
		return
	}

	pod := podInfo.Pod
	// TODO(knelasevero): Remove duplicated keys from log entry calls
	// When contextualized logging hits GA
	// https://github.com/kubernetes/kubernetes/issues/111672
	logger = klog.LoggerWithValues(logger, "pod", klog.KObj(pod))
	ctx = klog.NewContext(ctx, logger)
	logger.V(4).Info("About to try and schedule pod", "pod", klog.KObj(pod))

	// 获取pod对应的调度插件
	fwk, err := sched.frameworkForPod(pod)
	if err != nil {
		// This shouldn't happen, because we only accept for scheduling the pods
		// which specify a scheduler name that matches one of the profiles.
		logger.Error(err, "Error occurred")
		return
	}
	// 判断 pod 是否跳过调度
	// 当pod被删除，即pod的deletionTime不为空时，跳过调度
	// 当pod之前就assumed时，跳过调度，除非更新过
	if sched.skipPodSchedule(ctx, fwk, pod) {
		return
	}

	logger.V(3).Info("Attempting to schedule pod", "pod", klog.KObj(pod))

	// Synchronously attempt to find a fit for the pod.
	start := time.Now()
	state := framework.NewCycleState()
	state.SetRecordPluginMetrics(rand.Intn(100) < pluginMetricsSamplePercent)

	// Initialize an empty podsToActivate struct, which will be filled up by plugins or stay empty.
	podsToActivate := framework.NewPodsToActivate()
	state.Write(framework.PodsToActivateKey, podsToActivate)

	schedulingCycleCtx, cancel := context.WithCancel(ctx)
	defer cancel()
	// 尝试调度单个pod，具体看下面schedulingCycle
	// 如果调度成功，会返回调度结果，其中包含推荐节点信息，pod绑定节点信息等
	scheduleResult, assumedPodInfo, status := sched.schedulingCycle(schedulingCycleCtx, state, fwk, podInfo, start, podsToActivate)
	// 如果调度失败，返回
	if !status.IsSuccess() {
		sched.FailureHandler(schedulingCycleCtx, fwk, assumedPodInfo, status, scheduleResult.nominatingInfo, start)
		return
	}

	// bind the pod to its host asynchronously (we can do this b/c of the assumption step above).
	go func() {
		bindingCycleCtx, cancel := context.WithCancel(ctx)
		defer cancel()

		metrics.Goroutines.WithLabelValues(metrics.Binding).Inc()
		defer metrics.Goroutines.WithLabelValues(metrics.Binding).Dec()

		// pod 真正绑定到节点，看下面bindingCycle
		status := sched.bindingCycle(bindingCycleCtx, state, fwk, scheduleResult, assumedPodInfo, start, podsToActivate)
		if !status.IsSuccess() {
			sched.handleBindingCycleError(bindingCycleCtx, state, fwk, assumedPodInfo, start, scheduleResult, status)
			return
		}
		// 从优先级队列中删除该pod，仅做二次保证，通常情况下该操作由队列自身的循环函数处理
		// Usually, DonePod is called inside the scheduling queue,
		// but in this case, we need to call it here because this Pod won't go back to the scheduling queue.
		sched.SchedulingQueue.Done(assumedPodInfo.Pod.UID)
	}()
}
```
#### schedulingCycle
调度单个pod
```go
func (sched *Scheduler) schedulingCycle(
	ctx context.Context,
	state *framework.CycleState,
	fwk framework.Framework,
	podInfo *framework.QueuedPodInfo,
	start time.Time,
	podsToActivate *framework.PodsToActivate,
) (ScheduleResult, *framework.QueuedPodInfo, *framework.Status) {
	logger := klog.FromContext(ctx)
	pod := podInfo.Pod
	// 看下文SchedulePod
	// 这里已经通过预选和打分选择到了最合适的一个节点，并且还返回了参与调度的节点数量和通过预选和优选插件的节点数量
	scheduleResult, err := sched.SchedulePod(ctx, fwk, state, pod)
	if err != nil {
		defer func() {
			metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInSeconds(start))
		}() 
		// 如果错误是没有可用节点，则直接返回
		if err == ErrNoNodesAvailable {
			status := framework.NewStatus(framework.UnschedulableAndUnresolvable).WithError(err)
			return ScheduleResult{nominatingInfo: clearNominatedNode}, podInfo, status
		}
		// 如果没有节点匹配，也直接返回
		fitError, ok := err.(*framework.FitError)
		if !ok {
			logger.Error(err, "Error selecting node for pod", "pod", klog.KObj(pod))
			return ScheduleResult{nominatingInfo: clearNominatedNode}, podInfo, framework.AsStatus(err)
		}

		// SchedulePod() may have failed because the pod would not fit on any host, so we try to
		// preempt, with the expectation that the next time the pod is tried for scheduling it
		// will fit due to the preemption. It is also possible that a different pod will schedule
		// into the resources that were preempted, but this is harmless.

		if !fwk.HasPostFilterPlugins() {
			logger.V(3).Info("No PostFilter plugins are registered, so no preemption will be performed")
			return ScheduleResult{}, podInfo, framework.NewStatus(framework.Unschedulable).WithError(err)
		}

		// 如果本次调度失败，但是有PostFilter插件，那么就尝试运行插件，下次调度的时候优先使用这个提名节点
		// Run PostFilter plugins to attempt to make the pod schedulable in a future scheduling cycle.
		result, status := fwk.RunPostFilterPlugins(ctx, state, pod, fitError.Diagnosis.NodeToStatusMap)
		msg := status.Message()
		fitError.Diagnosis.PostFilterMsg = msg
		if status.Code() == framework.Error {
			logger.Error(nil, "Status after running PostFilter plugins for pod", "pod", klog.KObj(pod), "status", msg)
		} else {
			logger.V(5).Info("Status after running PostFilter plugins for pod", "pod", klog.KObj(pod), "status", msg)
		}

		var nominatingInfo *framework.NominatingInfo
		if result != nil {
			nominatingInfo = result.NominatingInfo
		}
		return ScheduleResult{nominatingInfo: nominatingInfo}, podInfo, framework.NewStatus(framework.Unschedulable).WithError(err)
	}

	metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInSeconds(start))
	// 更新缓存，假定pod已经在推荐节点上运行
	// Tell the cache to assume that a pod now is running on a given node, even though it hasn't been bound yet.
	// This allows us to keep scheduling without waiting on binding to occur.
	assumedPodInfo := podInfo.DeepCopy()
	assumedPod := assumedPodInfo.Pod
	// 更新assumedPod中的NodeName
	// assume modifies `assumedPod` by setting NodeName=scheduleResult.SuggestedHost
	err = sched.assume(logger, assumedPod, scheduleResult.SuggestedHost)
	if err != nil {
		// This is most probably result of a BUG in retrying logic.
		// We report an error here so that pod scheduling can be retried.
		// This relies on the fact that Error will check if the pod has been bound
		// to a node and if so will not add it back to the unscheduled pods queue
		// (otherwise this would cause an infinite loop).
		return ScheduleResult{nominatingInfo: clearNominatedNode}, assumedPodInfo, framework.AsStatus(err)
	}

	// 当资源已经预留给 Pod 时，会通知插件
	// Run the Reserve method of reserve plugins.
	if sts := fwk.RunReservePluginsReserve(ctx, state, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
		// 如果插件验证不通过，则回收之前保留的资源
		// trigger un-reserve to clean up state associated with the reserved Pod
		fwk.RunReservePluginsUnreserve(ctx, state, assumedPod, scheduleResult.SuggestedHost)
		if forgetErr := sched.Cache.ForgetPod(logger, assumedPod); forgetErr != nil {
			logger.Error(forgetErr, "Scheduler cache ForgetPod failed")
		}

		if sts.IsRejected() {
			fitErr := &framework.FitError{
				NumAllNodes: 1,
				Pod:         pod,
				Diagnosis: framework.Diagnosis{
					NodeToStatusMap: framework.NodeToStatusMap{scheduleResult.SuggestedHost: sts},
				},
			}
			fitErr.Diagnosis.AddPluginStatus(sts)
			return ScheduleResult{nominatingInfo: clearNominatedNode}, assumedPodInfo, framework.NewStatus(sts.Code()).WithError(fitErr)
		}
		return ScheduleResult{nominatingInfo: clearNominatedNode}, assumedPodInfo, sts
	}

	// 准入插件
	// Run "permit" plugins.
	runPermitStatus := fwk.RunPermitPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost)
	if !runPermitStatus.IsWait() && !runPermitStatus.IsSuccess() {
		// 如果准入插件验证不通过，则回收之前保留的资源
		// trigger un-reserve to clean up state associated with the reserved Pod
		fwk.RunReservePluginsUnreserve(ctx, state, assumedPod, scheduleResult.SuggestedHost)
		if forgetErr := sched.Cache.ForgetPod(logger, assumedPod); forgetErr != nil {
			logger.Error(forgetErr, "Scheduler cache ForgetPod failed")
		}

		if runPermitStatus.IsRejected() {
			fitErr := &framework.FitError{
				NumAllNodes: 1,
				Pod:         pod,
				Diagnosis: framework.Diagnosis{
					NodeToStatusMap: framework.NodeToStatusMap{scheduleResult.SuggestedHost: runPermitStatus},
				},
			}
			fitErr.Diagnosis.AddPluginStatus(runPermitStatus)
			return ScheduleResult{nominatingInfo: clearNominatedNode}, assumedPodInfo, framework.NewStatus(runPermitStatus.Code()).WithError(fitErr)
		}

		return ScheduleResult{nominatingInfo: clearNominatedNode}, assumedPodInfo, runPermitStatus
	}

	// At the end of a successful scheduling cycle, pop and move up Pods if needed.
	if len(podsToActivate.Map) != 0 {
		sched.SchedulingQueue.Activate(logger, podsToActivate.Map)
		// Clear the entries after activation.
		podsToActivate.Map = make(map[string]*v1.Pod)
	}

	return scheduleResult, assumedPodInfo, nil
}
```
#### SchedulePod
```go
// 尝试调度单个pod到node列表中的一个可用节点上。如果调度成功，返回调度结果，否则返回错误
func (sched *Scheduler) schedulePod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
	trace := utiltrace.New("Scheduling", utiltrace.Field{Key: "namespace", Value: pod.Namespace}, utiltrace.Field{Key: "name", Value: pod.Name})
	defer trace.LogIfLong(100 * time.Millisecond)
	// 更新节点上的pod调度信息
	if err := sched.Cache.UpdateSnapshot(klog.FromContext(ctx), sched.nodeInfoSnapshot); err != nil {
		return result, err
	}
	trace.Step("Snapshotting scheduler cache and node infos done")

	// 如果缓存中的node数量为0，则返回错误
	if sched.nodeInfoSnapshot.NumNodes() == 0 {
		return result, ErrNoNodesAvailable
	}

	// 寻找满足pod调度条件的node列表，看下面findNodesThatFitPod
	feasibleNodes, diagnosis, err := sched.findNodesThatFitPod(ctx, fwk, state, pod)
	if err != nil {
		return result, err
	}
	trace.Step("Computing predicates done")

	// 如果可用node为0，则直接返回错误
	if len(feasibleNodes) == 0 {
		return result, &framework.FitError{
			Pod:         pod,
			NumAllNodes: sched.nodeInfoSnapshot.NumNodes(),
			Diagnosis:   diagnosis,
		}
	}
	// 如果可用node为1，就直接使用这个node
	// When only one node after predicate, just use it.
	if len(feasibleNodes) == 1 {
		return ScheduleResult{
			SuggestedHost:  feasibleNodes[0].Node().Name,
			EvaluatedNodes: 1 + len(diagnosis.NodeToStatusMap),
			FeasibleNodes:  1,
		}, nil
	}
	// 运行打分插件对通过预选的节点进行打分，对于一个节点来说可能有多个打分插件，会计算加权分数来获取一个优先级列表
	priorityList, err := prioritizeNodes(ctx, sched.Extenders, fwk, state, pod, feasibleNodes)
	if err != nil {
		return result, err
	}
	// 根据上面的打分情况获取最优的node，和分数较高的几个node集合，实际上就是对上面的优先级队列进行排序并选择最前面的那一个节点
	host, _, err := selectHost(priorityList, numberOfHighestScoredNodesToReport)
	trace.Step("Prioritizing done")

	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(feasibleNodes) + len(diagnosis.NodeToStatusMap),
		FeasibleNodes:  len(feasibleNodes),
	}, err
}
```
#### findNodesThatFitPod
```go
func (sched *Scheduler) findNodesThatFitPod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) ([]*framework.NodeInfo, framework.Diagnosis, error) {
	logger := klog.FromContext(ctx)
	// 获取所有节点列表
	allNodes, err := sched.nodeInfoSnapshot.NodeInfos().List()
	if err != nil {
		return nil, framework.Diagnosis{
			NodeToStatusMap: make(framework.NodeToStatusMap),
		}, err
	}
	// 初始化diagnosis，用于记录节点调度情况
	diagnosis := framework.Diagnosis{
		NodeToStatusMap: make(framework.NodeToStatusMap, len(allNodes)),
	}
	// 预选阶段，运行预选插件，返回预选结果和状态码，如果预选插件返回的结果不是Success或skip，code将被置为non-success
	// 当返回skip状态时，status中的结果和其他字段将被忽略，并且这个调度周期内的其他预选插件也会被跳过。如果返回non-success，调度将被终止
	// 插件太多，不展开看
	// Run "prefilter" plugins.
	preRes, s := fwk.RunPreFilterPlugins(ctx, state, pod)
	if !s.IsSuccess() {
		if !s.IsRejected() {
			return nil, diagnosis, s.AsError()
		}
		// All nodes in NodeToStatusMap will have the same status so that they can be handled in the preemption.
		// Some non trivial refactoring is needed to avoid this copy.
		for _, n := range allNodes {
			diagnosis.NodeToStatusMap[n.Node().Name] = s
		}

		// Record the messages from PreFilter in Diagnosis.PreFilterMsg.
		msg := s.Message()
		diagnosis.PreFilterMsg = msg
		logger.V(5).Info("Status after running PreFilter plugins for pod", "pod", klog.KObj(pod), "status", msg)
		// 记录插件状态，pod创建后一直处于pending状态就来自于这里，一般是需要的依赖没有准备好，如节点不可用
		diagnosis.AddPluginStatus(s)
		return nil, diagnosis, nil
	}

	// 提名节点有可能在上一次调度周期中被隐式设置，所以需要优先尝试这些节点是否满足要求
	// "NominatedNodeName" can potentially be set in a previous scheduling cycle as a result of preemption.
	// This node is likely the only candidate that will fit the pod, and hence we try it first before iterating over all nodes.
	if len(pod.Status.NominatedNodeName) > 0 {
		feasibleNodes, err := sched.evaluateNominatedNode(ctx, pod, fwk, state, diagnosis)
		if err != nil {
			logger.Error(err, "Evaluation failed on nominated node", "pod", klog.KObj(pod), "node", pod.Status.NominatedNodeName)
		}
		// Nominated node passes all the filters, scheduler is good to assign this node to the pod.
		if len(feasibleNodes) != 0 {
			return feasibleNodes, diagnosis, nil
		}
	}

	// 假设pod可以被调度到所有节点上，但是如果预选结果中并不是所有节点都满足要求，则筛选出那些节点，并且记录不符合预选要求的节点落选原因
	nodes := allNodes
	if !preRes.AllNodes() {
		nodes = make([]*framework.NodeInfo, 0, len(preRes.NodeNames))
		for _, n := range allNodes {
			if !preRes.NodeNames.Has(n.Node().Name) {
				// We consider Nodes that are filtered out by PreFilterResult as rejected via UnschedulableAndUnresolvable.
				// We have to record them in NodeToStatusMap so that they won't be considered as candidates in the preemption.
				diagnosis.NodeToStatusMap[n.Node().Name] = framework.NewStatus(framework.UnschedulableAndUnresolvable, "node is filtered out by the prefilter result")
				continue
			}
			nodes = append(nodes, n)
		}
	}
	// 在选择节点时并不会对所有节点都打分，会按比例进行分配，比如需要三个节点参与打分，那就不会再去获取其余节点的信息，如果集群中节点数量较多，配置合理的指以提高调度器调度性能
	// 用于过滤不能运行 Pod 的节点。 过滤器的调用顺序是可配置的。 如果没有一个节点通过所有过滤器的筛选，Pod 将会被标记为不可调度。
	feasibleNodes, err := sched.findNodesThatPassFilters(ctx, fwk, state, pod, &diagnosis, nodes)
	// 记录当前参与调度的节点数，以平均各节点被调度到的几率
	// always try to update the sched.nextStartNodeIndex regardless of whether an error has occurred
	// this is helpful to make sure that all the nodes have a chance to be searched
	processedNodes := len(feasibleNodes) + len(diagnosis.NodeToStatusMap)
	sched.nextStartNodeIndex = (sched.nextStartNodeIndex + processedNodes) % len(nodes)
	if err != nil {
		return nil, diagnosis, err
	}
	// 跟上面一样，运行额外插件，过滤掉不符合要求的节点
	feasibleNodesAfterExtender, err := findNodesThatPassExtenders(ctx, sched.Extenders, pod, feasibleNodes, diagnosis.NodeToStatusMap)
	if err != nil {
		return nil, diagnosis, err
	}
	// 如果这两组插件匹配的节点数量不同，则在diagnosis的不可调度插件上集合中的所有元素中都新增插件名
	if len(feasibleNodesAfterExtender) != len(feasibleNodes) {
		// Extenders filtered out some nodes.
		//
		// Extender doesn't support any kind of requeueing feature like EnqueueExtensions in the scheduling framework.
		// When Extenders reject some Nodes and the pod ends up being unschedulable,
		// we put framework.ExtenderName to pInfo.UnschedulablePlugins.
		// This Pod will be requeued from unschedulable pod pool to activeQ/backoffQ
		// by any kind of cluster events.
		// https://github.com/kubernetes/kubernetes/issues/122019
		if diagnosis.UnschedulablePlugins == nil {
			diagnosis.UnschedulablePlugins = sets.New[string]()
		}
		diagnosis.UnschedulablePlugins.Insert(framework.ExtenderName)
	}

	return feasibleNodesAfterExtender, diagnosis, nil
}
```
#### bindingCycle
尝试将pod绑定到节点上
```go
func (sched *Scheduler) bindingCycle(
	ctx context.Context,
	state *framework.CycleState,
	fwk framework.Framework,
	scheduleResult ScheduleResult,
	assumedPodInfo *framework.QueuedPodInfo,
	start time.Time,
	podsToActivate *framework.PodsToActivate) *framework.Status {
	logger := klog.FromContext(ctx)

	assumedPod := assumedPodInfo.Pod

	// 之前schedulingCycle中运行的准入插件，读取运行结果，并判断是否进行后续步骤
	// Run "permit" plugins.
	if status := fwk.WaitOnPermit(ctx, assumedPod); !status.IsSuccess() {
		if status.IsRejected() {
			fitErr := &framework.FitError{
				NumAllNodes: 1,
				Pod:         assumedPodInfo.Pod,
				Diagnosis: framework.Diagnosis{
					NodeToStatusMap:      framework.NodeToStatusMap{scheduleResult.SuggestedHost: status},
					UnschedulablePlugins: sets.New(status.Plugin()),
				},
			}
			return framework.NewStatus(status.Code()).WithError(fitErr)
		}
		return status
	}
	// 预绑定
	// Run "prebind" plugins.
	if status := fwk.RunPreBindPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost); !status.IsSuccess() {
		return status
	}

	// 绑定，调用client-go更新pod状态
	// Run "bind" plugins.
	if status := sched.bind(ctx, fwk, assumedPod, scheduleResult.SuggestedHost, state); !status.IsSuccess() {
		return status
	}

	// Calculating nodeResourceString can be heavy. Avoid it if klog verbosity is below 2.
	logger.V(2).Info("Successfully bound pod to node", "pod", klog.KObj(assumedPod), "node", scheduleResult.SuggestedHost, "evaluatedNodes", scheduleResult.EvaluatedNodes, "feasibleNodes", scheduleResult.FeasibleNodes)
	metrics.PodScheduled(fwk.ProfileName(), metrics.SinceInSeconds(start))
	metrics.PodSchedulingAttempts.Observe(float64(assumedPodInfo.Attempts))
	if assumedPodInfo.InitialAttemptTimestamp != nil {
		metrics.PodSchedulingDuration.WithLabelValues(getAttemptsLabel(assumedPodInfo)).Observe(metrics.SinceInSeconds(*assumedPodInfo.InitialAttemptTimestamp))
		metrics.PodSchedulingSLIDuration.WithLabelValues(getAttemptsLabel(assumedPodInfo)).Observe(metrics.SinceInSeconds(*assumedPodInfo.InitialAttemptTimestamp))
	}
	// Run "postbind" plugins.
	fwk.RunPostBindPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost)

	// At the end of a successful binding cycle, move up Pods if needed.
	if len(podsToActivate.Map) != 0 {
		sched.SchedulingQueue.Activate(logger, podsToActivate.Map)
		// Unlike the logic in schedulingCycle(), we don't bother deleting the entries
		// as `podsToActivate.Map` is no longer consumed.
	}

	return nil
}
```