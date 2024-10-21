### pod autoscaler
顾名思义，pod autoscaler 是 Kubernetes HPA 的功能，根据设定的指标对 pod 进行自动扩缩，使其资源利用率始终保持于设定值范围内。
HPA 中只包含水平扩缩，即更改 pod 数量以分配请求从而降低单个 pod 负载。源码中使用 HorizontalPodAutoscalerController 定义。
#### HorizontalController 
老规矩，先来看看 pod autoscaler controller 的结构体定义，从代码注释上来看 HorizontalController 负责同步系统中的 HPA 对象和 HPA 对象所控制的 deployment/replication controllers 的状态。
```go
type HorizontalController struct {
    // 接口类型，用于操作各类资源对象，方法签名为 Scales(namespace)，可知是在一个特定的 namespace 中对资源对象进行扩缩
	scaleNamespacer scaleclient.ScalesGetter
    // 同样是针对某个特定 namespace 返回 HorizontalPodAutoscalerInterface，使用这个接口操作 HPA 对象
	hpaNamespacer   autoscalingclient.HorizontalPodAutoscalersGetter
	mapper          apimeta.RESTMapper

    // 用于计算 HPA 所需副本数量
	replicaCalc   *ReplicaCalculator
	eventRecorder record.EventRecorder

    // 缩容时间窗口，即负载减小后等待多久才会进行缩容
	downscaleStabilisationWindow time.Duration
    
	monitor monitor.Monitor

    // HPA informer
	// hpaLister is able to list/get HPAs from the shared cache from the informer passed in to
	// NewHorizontalController.
	hpaLister       autoscalinglisters.HorizontalPodAutoscalerLister
	hpaListerSynced cache.InformerSynced

    // pod informer
	// podLister is able to list/get Pods from the shared cache from the informer passed in to
	// NewHorizontalController.
	podLister       corelisters.PodLister
	podListerSynced cache.InformerSynced

    // 需要被同步的 controller 队列 
	// Controllers that need to be synced
	queue workqueue.RateLimitingInterface

    // 
	// Latest unstabilized recommendations for each autoscaler.
	recommendations     map[string][]timestampedRecommendation
	recommendationsLock sync.Mutex

	// Latest autoscaler events
	scaleUpEvents       map[string][]timestampedScaleEvent
	scaleUpEventsLock   sync.RWMutex
	scaleDownEvents     map[string][]timestampedScaleEvent
	scaleDownEventsLock sync.RWMutex

    // HPA 的双向哈希表，通过一个对象查找与之相关联的对象，既能从一个“对象 A”找到与之关联的“对象 B”，也能从“对象 B”找到所有关联的“对象 A”。
	// Storage of HPAs and their selectors.
	hpaSelectors    *selectors.BiMultimap
	hpaSelectorsMux sync.Mutex
}
```
#### 调用路径
初始化 kube controller 时注册已知的 controller 实现，即 newHorizontalPodAutoscalerControllerDescriptor，直接调用 Run 方法即为整个初始化流程。
```go
// cmd/kube-controller-manager/app/controllermanager.go
func NewControllerDescriptors() map[string]*ControllerDescriptor {
    ...
    register(newHorizontalPodAutoscalerControllerDescriptor())
    ...
}

func startHPAControllerWithMetricsClient(ctx context.Context, controllerContext ControllerContext, metricsClient metrics.MetricsClient) (controller.Interface, bool, error) {
    ...
	go podautoscaler.NewHorizontalController(
		ctx,
		hpaClient.CoreV1(),
		scaleClient,
		hpaClient.AutoscalingV2(),
		controllerContext.RESTMapper,
		metricsClient,
		controllerContext.InformerFactory.Autoscaling().V2().HorizontalPodAutoscalers(),
		controllerContext.InformerFactory.Core().V1().Pods(),
		controllerContext.ComponentConfig.HPAController.HorizontalPodAutoscalerSyncPeriod.Duration,
		controllerContext.ComponentConfig.HPAController.HorizontalPodAutoscalerDownscaleStabilizationWindow.Duration,
		controllerContext.ComponentConfig.HPAController.HorizontalPodAutoscalerTolerance,
		controllerContext.ComponentConfig.HPAController.HorizontalPodAutoscalerCPUInitializationPeriod.Duration,
		controllerContext.ComponentConfig.HPAController.HorizontalPodAutoscalerInitialReadinessDelay.Duration,
	).Run(ctx, int(controllerContext.ComponentConfig.HPAController.ConcurrentHorizontalPodAutoscalerSyncs))
	return nil, true, nil
}

```

#### HorizontalController 初始化
初始化的时候传入一堆值，然后返回 HorizontalController 对象。
```go
func NewHorizontalController(
	ctx context.Context,
	evtNamespacer v1core.EventsGetter,
	scaleNamespacer scaleclient.ScalesGetter,
	hpaNamespacer autoscalingclient.HorizontalPodAutoscalersGetter,
	mapper apimeta.RESTMapper,
	metricsClient metricsclient.MetricsClient,
	hpaInformer autoscalinginformers.HorizontalPodAutoscalerInformer,
	podInformer coreinformers.PodInformer,
	resyncPeriod time.Duration,
	downscaleStabilisationWindow time.Duration,
	tolerance float64,
	cpuInitializationPeriod,
	delayOfInitialReadinessStatus time.Duration,
) *HorizontalController {
	broadcaster := record.NewBroadcaster(record.WithContext(ctx))
	broadcaster.StartStructuredLogging(3)
	broadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: evtNamespacer.Events("")})
	recorder := broadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "horizontal-pod-autoscaler"})

	hpaController := &HorizontalController{
		eventRecorder:                recorder,
		scaleNamespacer:              scaleNamespacer,
		hpaNamespacer:                hpaNamespacer,
		downscaleStabilisationWindow: downscaleStabilisationWindow,
		monitor:                      monitor.New(),
		queue:                        workqueue.NewNamedRateLimitingQueue(NewDefaultHPARateLimiter(resyncPeriod), "horizontalpodautoscaler"),
		mapper:                       mapper,
		recommendations:              map[string][]timestampedRecommendation{},
		recommendationsLock:          sync.Mutex{},
		scaleUpEvents:                map[string][]timestampedScaleEvent{},
		scaleUpEventsLock:            sync.RWMutex{},
		scaleDownEvents:              map[string][]timestampedScaleEvent{},
		scaleDownEventsLock:          sync.RWMutex{},
		hpaSelectors:                 selectors.NewBiMultimap(),
	}

	hpaInformer.Informer().AddEventHandlerWithResyncPeriod(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    hpaController.enqueueHPA,
			UpdateFunc: hpaController.updateHPA,
			DeleteFunc: hpaController.deleteHPA,
		},
		resyncPeriod,
	)
	hpaController.hpaLister = hpaInformer.Lister()
	hpaController.hpaListerSynced = hpaInformer.Informer().HasSynced

	hpaController.podLister = podInformer.Lister()
	hpaController.podListerSynced = podInformer.Informer().HasSynced

	replicaCalc := NewReplicaCalculator(
		metricsClient,
		hpaController.podLister,
		tolerance,
		cpuInitializationPeriod,
		delayOfInitialReadinessStatus,
	)
	hpaController.replicaCalc = replicaCalc

	monitor.Register()

	return hpaController
}
```
上面看到 HPA informer 在初始化后当 HPA 资源发生变化时添加了三个方法，对应 enqueueHPA、updateHPA、deleteHPA。逐个来看。
##### enqueueHPA
当有新增的 HPA 资源时会触发该操作，将 HPA 对象加入 queue 队列中进行处理，并在上面提到的双向哈希表中注册该对象。
```go
func (a *HorizontalController) enqueueHPA(obj interface{}) {
    // 将传入的对象转换为唯一的 key 值，通常是 namespace/name 格式
	key, err := controller.KeyFunc(obj)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get key for object %+v: %v", obj, err))
		return
	}

    // 加入限速队列，如果队列中已经存在该 HPA 的请求，则后续的请求会被丢弃。
	// Requests are always added to queue with resyncPeriod delay.  If there's already
	// request for the HPA in the queue then a new request is always dropped. Requests spend resync
	// interval in queue so HPAs are processed every resync interval.
	a.queue.AddRateLimited(key)

    // 在 HPA 选择器中注册。
	// Register HPA in the hpaSelectors map if it's not present yet. Attaching the Nothing selector
	// that does not select objects. The actual selector is going to be updated
	// when it's available during the autoscaler reconciliation.
	a.hpaSelectorsMux.Lock()
	defer a.hpaSelectorsMux.Unlock()
	if hpaKey := selectors.Parse(key); !a.hpaSelectors.SelectorExists(hpaKey) {
		a.hpaSelectors.PutSelector(hpaKey, labels.Nothing())
	}
}
```
##### 
同上 enqueueHPA
```go
func (a *HorizontalController) updateHPA(old, cur interface{}) {
	a.enqueueHPA(cur)
}
```
##### deleteHPA
```go
func (a *HorizontalController) deleteHPA(obj interface{}) {
    // 获取唯一 key 值
	key, err := controller.KeyFunc(obj)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get key for object %+v: %v", obj, err))
		return
	}
    // 队列中删除该 HPA 对象
	// TODO: could we leak if we fail to get the key?
	a.queue.Forget(key)

    // 删除 HPA 对应的选择器
	// Remove HPA and attached selector.
	a.hpaSelectorsMux.Lock()
	defer a.hpaSelectorsMux.Unlock()
	a.hpaSelectors.DeleteSelector(selectors.Parse(key))
}
```
#### Run
熟悉 kubernetes 代码的的都直到这一段代码中最主要的就是 a.worker 方法，这一段都是代码层层包装，看主逻辑就一直往里面跳
```go
// Run begins watching and syncing.
func (a *HorizontalController) Run(ctx context.Context, workers int) {
	defer utilruntime.HandleCrash()
	defer a.queue.ShutDown()

	logger := klog.FromContext(ctx)
	logger.Info("Starting HPA controller")
	defer logger.Info("Shutting down HPA controller")

	if !cache.WaitForNamedCacheSync("HPA", ctx.Done(), a.hpaListerSynced, a.podListerSynced) {
		return
	}

	for i := 0; i < workers; i++ {
		go wait.UntilWithContext(ctx, a.worker, time.Second)
	}

	<-ctx.Done()
}
// pkg/controller/podautoscaler/horizontal.go:254
func (a *HorizontalController) worker(ctx context.Context) {
	for a.processNextWorkItem(ctx) {
	}
	logger := klog.FromContext(ctx)
	logger.Info("Horizontal Pod Autoscaler controller worker shutting down")
}
```
关键的地方来了，这里的 processNextWorkItem 方法就是处理 queue 中的 HPA 请求。按照一般逻辑，从队列中获取资源，然后处理资源。
每个 HPA 的任务应该按照 resyncPeriod 定期重新加入队列。但由于任务的加入和移除存在竞争条件，有时在队列中移除任务的瞬间，新的任务还未加入。这会导致任务丢失，并可能导致 HPA 在 2 倍的 resyncPeriod 后才能被处理。
```go
func (a *HorizontalController) processNextWorkItem(ctx context.Context) bool {
	key, quit := a.queue.Get()
	if quit {
		return false
	}
	defer a.queue.Done(key)

	deleted, err := a.reconcileKey(ctx, key.(string))
	if err != nil {
		utilruntime.HandleError(err)
	}
	// Add request processing HPA to queue with resyncPeriod delay.
	// Requests are always added to queue with resyncPeriod delay. If there's already request
	// for the HPA in the queue then a new request is always dropped. Requests spend resyncPeriod
	// in queue so HPAs are processed every resyncPeriod.
	// Request is added here just in case last resync didn't insert request into the queue. This
	// happens quite often because there is race condition between adding request after resyncPeriod
	// and removing them from queue. Request can be added by resync before previous request is
	// removed from queue. If we didn't add request here then in this case one request would be dropped
	// and HPA would process after 2 x resyncPeriod.
	if !deleted {
		a.queue.AddRateLimited(key)
	}

	return true
}
```
##### reconcileKey
```go
func (a *HorizontalController) reconcileKey(ctx context.Context, key string) (deleted bool, err error) {
    // 从传入的 key 解析 namespace 和 name
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return true, err
	}

	logger := klog.FromContext(ctx)
    // 从本地 hpa informer 缓存中获取指定 name 的 hpa 对象
	hpa, err := a.hpaLister.HorizontalPodAutoscalers(namespace).Get(name)
    // 如果没找到,则删除 controller 中相关资源,避免后续出现错误
	if k8serrors.IsNotFound(err) {
		logger.Info("Horizontal Pod Autoscaler has been deleted", "HPA", klog.KRef(namespace, name))

		a.recommendationsLock.Lock()
		delete(a.recommendations, key)
		a.recommendationsLock.Unlock()

		a.scaleUpEventsLock.Lock()
		delete(a.scaleUpEvents, key)
		a.scaleUpEventsLock.Unlock()

		a.scaleDownEventsLock.Lock()
		delete(a.scaleDownEvents, key)
		a.scaleDownEventsLock.Unlock()

		return true, nil
	}
	if err != nil {
		return false, err
	}
    // 如果找到了 hpa 对象，则调用 reconcileAutoscaler 方法
	return false, a.reconcileAutoscaler(ctx, hpa, key)
}
```

```go
func (a *HorizontalController) reconcileAutoscaler(ctx context.Context, hpaShared *autoscalingv2.HorizontalPodAutoscaler, key string) (retErr error) {
	// actionLabel is used to report which actions this reconciliation has taken.
	actionLabel := monitor.ActionLabelNone
	start := time.Now()
	defer func() {
		errorLabel := monitor.ErrorLabelNone
		if retErr != nil {
			// In case of error, set "internal" as default.
			errorLabel = monitor.ErrorLabelInternal
		}
		if errors.Is(retErr, errSpec) {
			errorLabel = monitor.ErrorLabelSpec
		}
        // 记录协调所花费的时间
		a.monitor.ObserveReconciliationResult(actionLabel, errorLabel, time.Since(start))
	}()
    // 创建 HPA 对象的深拷贝，以免修改 Informer 缓存中的共享对象。
	// make a copy so that we never mutate the shared informer cache (conversion can mutate the object)
	hpa := hpaShared.DeepCopy()
	hpaStatusOriginal := hpa.Status.DeepCopy()

	reference := fmt.Sprintf("%s/%s/%s", hpa.Spec.ScaleTargetRef.Kind, hpa.Namespace, hpa.Spec.ScaleTargetRef.Name)
    // 检查 HPA 目标资源的 API 版本,如果解析失败，则记录事件和错误，并设置 HPA 状态。
	targetGV, err := schema.ParseGroupVersion(hpa.Spec.ScaleTargetRef.APIVersion)
	if err != nil {
		a.eventRecorder.Event(hpa, v1.EventTypeWarning, "FailedGetScale", err.Error())
		setCondition(hpa, autoscalingv2.AbleToScale, v1.ConditionFalse, "FailedGetScale", "the HPA controller was unable to get the target's current scale: %v", err)
		if err := a.updateStatusIfNeeded(ctx, hpaStatusOriginal, hpa); err != nil {
			utilruntime.HandleError(err)
		}
		return fmt.Errorf("invalid API version in scale target reference: %v%w", err, errSpec)
	}

	targetGK := schema.GroupKind{
		Group: targetGV.Group,
		Kind:  hpa.Spec.ScaleTargetRef.Kind,
	}

	mappings, err := a.mapper.RESTMappings(targetGK)
	if err != nil {
		a.eventRecorder.Event(hpa, v1.EventTypeWarning, "FailedGetScale", err.Error())
		setCondition(hpa, autoscalingv2.AbleToScale, v1.ConditionFalse, "FailedGetScale", "the HPA controller was unable to get the target's current scale: %v", err)
		if err := a.updateStatusIfNeeded(ctx, hpaStatusOriginal, hpa); err != nil {
			utilruntime.HandleError(err)
		}
		return fmt.Errorf("unable to determine resource for scale target reference: %v", err)
	}

    // 获取目标资源（如 Deployment）的 当前副本数。
	scale, targetGR, err := a.scaleForResourceMappings(ctx, hpa.Namespace, hpa.Spec.ScaleTargetRef.Name, mappings)
	if err != nil {
		a.eventRecorder.Event(hpa, v1.EventTypeWarning, "FailedGetScale", err.Error())
		setCondition(hpa, autoscalingv2.AbleToScale, v1.ConditionFalse, "FailedGetScale", "the HPA controller was unable to get the target's current scale: %v", err)
		if err := a.updateStatusIfNeeded(ctx, hpaStatusOriginal, hpa); err != nil {
			utilruntime.HandleError(err)
		}
		return fmt.Errorf("failed to query scale subresource for %s: %v", reference, err)
	}
	setCondition(hpa, autoscalingv2.AbleToScale, v1.ConditionTrue, "SucceededGetScale", "the HPA controller was able to get the target's current scale")
	currentReplicas := scale.Spec.Replicas
	a.recordInitialRecommendation(currentReplicas, key)

	var (
		metricStatuses        []autoscalingv2.MetricStatus
		metricDesiredReplicas int32
		metricName            string
	)

	desiredReplicas := int32(0)
	rescaleReason := ""

	var minReplicas int32

	if hpa.Spec.MinReplicas != nil {
		minReplicas = *hpa.Spec.MinReplicas
	} else {
		// Default value
		minReplicas = 1
	}

	rescale := true
	logger := klog.FromContext(ctx)

	if currentReplicas == 0 && minReplicas != 0 {
		// Autoscaling is disabled for this resource
		desiredReplicas = 0
		rescale = false
		setCondition(hpa, autoscalingv2.ScalingActive, v1.ConditionFalse, "ScalingDisabled", "scaling is disabled since the replica count of the target is zero")
	} else if currentReplicas > hpa.Spec.MaxReplicas {
		rescaleReason = "Current number of replicas above Spec.MaxReplicas"
		desiredReplicas = hpa.Spec.MaxReplicas
	} else if currentReplicas < minReplicas {
		rescaleReason = "Current number of replicas below Spec.MinReplicas"
		desiredReplicas = minReplicas
	} else {
		var metricTimestamp time.Time
        // 计算所需的副本数.详细的看下面
		metricDesiredReplicas, metricName, metricStatuses, metricTimestamp, err = a.computeReplicasForMetrics(ctx, hpa, scale, hpa.Spec.Metrics)
		// computeReplicasForMetrics may return both non-zero metricDesiredReplicas and an error.
		// That means some metrics still work and HPA should perform scaling based on them.
		if err != nil && metricDesiredReplicas == -1 {
			a.setCurrentReplicasAndMetricsInStatus(hpa, currentReplicas, metricStatuses)
			if err := a.updateStatusIfNeeded(ctx, hpaStatusOriginal, hpa); err != nil {
				utilruntime.HandleError(err)
			}
			a.eventRecorder.Event(hpa, v1.EventTypeWarning, "FailedComputeMetricsReplicas", err.Error())
			return fmt.Errorf("failed to compute desired number of replicas based on listed metrics for %s: %v", reference, err)
		}
		if err != nil {
			// We proceed to scaling, but return this error from reconcileAutoscaler() finally.
			retErr = err
		}

		logger.V(4).Info("Proposing desired replicas",
			"desiredReplicas", metricDesiredReplicas,
			"metric", metricName,
			"timestamp", metricTimestamp,
			"scaleTarget", reference)

		rescaleMetric := ""
		if metricDesiredReplicas > desiredReplicas {
			desiredReplicas = metricDesiredReplicas
			rescaleMetric = metricName
		}
        // 决定是否扩缩
		if desiredReplicas > currentReplicas {
			rescaleReason = fmt.Sprintf("%s above target", rescaleMetric)
		}
		if desiredReplicas < currentReplicas {
			rescaleReason = "All metrics below target"
		}
		if hpa.Spec.Behavior == nil {
			desiredReplicas = a.normalizeDesiredReplicas(hpa, key, currentReplicas, desiredReplicas, minReplicas)
		} else {
			desiredReplicas = a.normalizeDesiredReplicasWithBehaviors(hpa, key, currentReplicas, desiredReplicas, minReplicas)
		}
		rescale = desiredReplicas != currentReplicas
	}
    // 执行扩缩
	if rescale {
		scale.Spec.Replicas = desiredReplicas
        // 更新目标资源的副本数,并记录事件
		_, err = a.scaleNamespacer.Scales(hpa.Namespace).Update(ctx, targetGR, scale, metav1.UpdateOptions{})
		if err != nil {
			a.eventRecorder.Eventf(hpa, v1.EventTypeWarning, "FailedRescale", "New size: %d; reason: %s; error: %v", desiredReplicas, rescaleReason, err.Error())
			setCondition(hpa, autoscalingv2.AbleToScale, v1.ConditionFalse, "FailedUpdateScale", "the HPA controller was unable to update the target scale: %v", err)
			a.setCurrentReplicasAndMetricsInStatus(hpa, currentReplicas, metricStatuses)
			if err := a.updateStatusIfNeeded(ctx, hpaStatusOriginal, hpa); err != nil {
				utilruntime.HandleError(err)
			}
			return fmt.Errorf("failed to rescale %s: %v", reference, err)
		}
		setCondition(hpa, autoscalingv2.AbleToScale, v1.ConditionTrue, "SucceededRescale", "the HPA controller was able to update the target scale to %d", desiredReplicas)
		a.eventRecorder.Eventf(hpa, v1.EventTypeNormal, "SuccessfulRescale", "New size: %d; reason: %s", desiredReplicas, rescaleReason)
		a.storeScaleEvent(hpa.Spec.Behavior, key, currentReplicas, desiredReplicas)
		logger.Info("Successfully rescaled",
			"HPA", klog.KObj(hpa),
			"currentReplicas", currentReplicas,
			"desiredReplicas", desiredReplicas,
			"reason", rescaleReason)

		if desiredReplicas > currentReplicas {
			actionLabel = monitor.ActionLabelScaleUp
		} else {
			actionLabel = monitor.ActionLabelScaleDown
		}
	} else {
		logger.V(4).Info("Decided not to scale",
			"scaleTarget", reference,
			"desiredReplicas", desiredReplicas,
			"lastScaleTime", hpa.Status.LastScaleTime)
		desiredReplicas = currentReplicas
	}

	a.setStatus(hpa, currentReplicas, desiredReplicas, metricStatuses, rescale)
    // 更新 HPA 状态
	err = a.updateStatusIfNeeded(ctx, hpaStatusOriginal, hpa)
	if err != nil {
		// we can overwrite retErr in this case because it's an internal error.
		return err
	}

	return retErr
}
```
##### computeReplicasForMetrics
<!-- TODO:后面再补 -->
```go
// computeReplicasForMetrics computes the desired number of replicas for the metric specifications listed in the HPA,
// returning the maximum of the computed replica counts, a description of the associated metric, and the statuses of
// all metrics computed.
// It may return both valid metricDesiredReplicas and an error,
// when some metrics still work and HPA should perform scaling based on them.
// If HPA cannot do anything due to error, it returns -1 in metricDesiredReplicas as a failure signal.
func (a *HorizontalController) computeReplicasForMetrics(ctx context.Context, hpa *autoscalingv2.HorizontalPodAutoscaler, scale *autoscalingv1.Scale,
	metricSpecs []autoscalingv2.MetricSpec) (replicas int32, metric string, statuses []autoscalingv2.MetricStatus, timestamp time.Time, err error) {

	selector, err := a.validateAndParseSelector(hpa, scale.Status.Selector)
	if err != nil {
		return -1, "", nil, time.Time{}, err
	}

	specReplicas := scale.Spec.Replicas
	statusReplicas := scale.Status.Replicas
	statuses = make([]autoscalingv2.MetricStatus, len(metricSpecs))

	invalidMetricsCount := 0
	var invalidMetricError error
	var invalidMetricCondition autoscalingv2.HorizontalPodAutoscalerCondition

	for i, metricSpec := range metricSpecs {
		replicaCountProposal, metricNameProposal, timestampProposal, condition, err := a.computeReplicasForMetric(ctx, hpa, metricSpec, specReplicas, statusReplicas, selector, &statuses[i])

		if err != nil {
			if invalidMetricsCount <= 0 {
				invalidMetricCondition = condition
				invalidMetricError = err
			}
			invalidMetricsCount++
			continue
		}
		if replicas == 0 || replicaCountProposal > replicas {
			timestamp = timestampProposal
			replicas = replicaCountProposal
			metric = metricNameProposal
		}
	}

	if invalidMetricError != nil {
		invalidMetricError = fmt.Errorf("invalid metrics (%v invalid out of %v), first error is: %v", invalidMetricsCount, len(metricSpecs), invalidMetricError)
	}

	// If all metrics are invalid or some are invalid and we would scale down,
	// return an error and set the condition of the hpa based on the first invalid metric.
	// Otherwise set the condition as scaling active as we're going to scale
	if invalidMetricsCount >= len(metricSpecs) || (invalidMetricsCount > 0 && replicas < specReplicas) {
		setCondition(hpa, invalidMetricCondition.Type, invalidMetricCondition.Status, invalidMetricCondition.Reason, invalidMetricCondition.Message)
		return -1, "", statuses, time.Time{}, invalidMetricError
	}
	setCondition(hpa, autoscalingv2.ScalingActive, v1.ConditionTrue, "ValidMetricFound", "the HPA was able to successfully calculate a replica count from %s", metric)

	return replicas, metric, statuses, timestamp, invalidMetricError
}
```