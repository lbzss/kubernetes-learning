### podgc controller
podgc controller 是一个简单的 gc 实现。如果处于 terminated 阶段的 pod 数量超过设定的阈值，该 controller 会按照 pod 的创建时间并按时间先后删除这些处于 terminated 状态的 pod，通常是处于 Failed 或 Succeed 状态的 pod。直到下一次数量超阈值进行下一次删除逻辑。注意：不会删除 non-terminated 状态的 pod。

#### 实现逻辑
脑补一下如果我们来实现这个简单的 controller 应该如何下手。有以下几个要点：
- 通过 informer 机制订阅 pod 状态变化
- 筛选出处于 terminated 状态的 pod
- 根据创建时间排序
- 根据设定的阈值进行删除操作

接下来看看源码是如何做的：

```go
// pkg/controller/podgc/gc_controller.go
type PodGCController struct {
	kubeClient clientset.Interface

    // informer 相关
	podLister        corelisters.PodLister
	podListerSynced  cache.InformerSynced
	nodeLister       corelisters.NodeLister
	nodeListerSynced cache.InformerSynced
    // 用于存放那些 pod 已经绑定但是 node informer 中不存在的 node
	nodeQueue workqueue.DelayingInterface

    // 阈值
	terminatedPodThreshold int
	gcCheckPeriod          time.Duration
	quarantineTime         time.Duration
}
```

##### PodGCController 初始化
比较简单，就是根据传参赋值，直接返回，没有做任何逻辑。在 metrics 中也注册了关于 pod 删除数量的 counter 指标。
```go
func NewPodGC(ctx context.Context, kubeClient clientset.Interface, podInformer coreinformers.PodInformer,
	nodeInformer coreinformers.NodeInformer, terminatedPodThreshold int) *PodGCController {
	return NewPodGCInternal(ctx, kubeClient, podInformer, nodeInformer, terminatedPodThreshold, gcCheckPeriod, quarantineTime)
}

// This function is only intended for integration tests
func NewPodGCInternal(ctx context.Context, kubeClient clientset.Interface, podInformer coreinformers.PodInformer,
	nodeInformer coreinformers.NodeInformer, terminatedPodThreshold int, gcCheckPeriod, quarantineTime time.Duration) *PodGCController {
	gcc := &PodGCController{
		kubeClient:             kubeClient,
		terminatedPodThreshold: terminatedPodThreshold,
		podLister:              podInformer.Lister(),
		podListerSynced:        podInformer.Informer().HasSynced,
		nodeLister:             nodeInformer.Lister(),
		nodeListerSynced:       nodeInformer.Informer().HasSynced,
		nodeQueue:              workqueue.NewNamedDelayingQueue("orphaned_pods_nodes"),
		gcCheckPeriod:          gcCheckPeriod,
		quarantineTime:         quarantineTime,
	}

	// Register prometheus metrics
	metrics.RegisterMetrics()
	return gcc
}
```

##### Run 方法
```go
func (gcc *PodGCController) Run(ctx context.Context) {
	logger := klog.FromContext(ctx)

	defer utilruntime.HandleCrash()

	logger.Info("Starting GC controller")
	defer gcc.nodeQueue.ShutDown()
	defer logger.Info("Shutting down GC controller")

    // informer 常规操作，等待相关信息已缓存到本地
	if !cache.WaitForNamedCacheSync("GC", ctx.Done(), gcc.podListerSynced, gcc.nodeListerSynced) {
		return
	}
    // 开启协程每隔一段时间进行 gc 操作
	go wait.UntilWithContext(ctx, gcc.gc, gcc.gcCheckPeriod)

	<-ctx.Done()
}
```
Run 方法非常简单，主要是 informer 的常规操作，接下来看看实际的逻辑 gc 方法。
从下面的代码可以看到也就是获取所有的 pod 和 node 列表，然后依次进行 gc，逐个来看。
```go
func (gcc *PodGCController) gc(ctx context.Context) {
    // informer 缓存中获取所有 pod 和 node
	pods, err := gcc.podLister.List(labels.Everything())
	if err != nil {
		klog.FromContext(ctx).Error(err, "Error while listing all pods")
		return
	}
	nodes, err := gcc.nodeLister.List(labels.Everything())
	if err != nil {
		klog.FromContext(ctx).Error(err, "Error while listing all nodes")
		return
	}
	if gcc.terminatedPodThreshold > 0 {
		gcc.gcTerminated(ctx, pods)
	}
	gcc.gcTerminating(ctx, pods)
	gcc.gcOrphaned(ctx, pods, nodes)
	gcc.gcUnscheduledTerminating(ctx, pods)
}
```
##### gcTerminated
顾名思义，就是删除处于 terminated 状态的 pod
```go
func (gcc *PodGCController) gcTerminated(ctx context.Context, pods []*v1.Pod) {
    // 过滤出处于 terminated 状态的 pod，即状态并不为 pending running unknown 的 pod
	terminatedPods := []*v1.Pod{}
	for _, pod := range pods {
		if isPodTerminated(pod) {
			terminatedPods = append(terminatedPods, pod)
		}
	}
    // 判断是否需要进行 gc
	terminatedPodCount := len(terminatedPods)
	deleteCount := terminatedPodCount - gcc.terminatedPodThreshold

	if deleteCount <= 0 {
		return
	}

	logger := klog.FromContext(ctx)
	logger.Info("Garbage collecting pods", "numPods", deleteCount)
	// sort only when necessary
    // 对 terminatedPods 进行排序， 转成 byEvictionAndCreationTimestamp 类型，实现了 sort 接口，具体逻辑下面再看
	sort.Sort(byEvictionAndCreationTimestamp(terminatedPods))
	var wait sync.WaitGroup
    // 使用 goroutine 并发执行 pod 的删除操作
	for i := 0; i < deleteCount; i++ {
		wait.Add(1)
		go func(pod *v1.Pod) {
			defer wait.Done()
            // 标记失败并删除 pod，实际上就是调用 kubeclient 直接删除
			if err := gcc.markFailedAndDeletePod(ctx, pod); err != nil {
				// ignore not founds
				defer utilruntime.HandleError(err)
				metrics.DeletingPodsErrorTotal.WithLabelValues(pod.Namespace, metrics.PodGCReasonTerminated).Inc()
			}
			metrics.DeletingPodsTotal.WithLabelValues(pod.Namespace, metrics.PodGCReasonTerminated).Inc()
		}(terminatedPods[i])
	}
	wait.Wait()
}
```
##### markFailedAndDeletePod
功能很简单，就是强制删除处于 terminated 状态的 pod。不过这里有特性门控 PodDisruptionConditions 和 JobPodReplacementPolicy。这里后面再详细看，先看主流程
```go
func (gcc *PodGCController) markFailedAndDeletePod(ctx context.Context, pod *v1.Pod) error {
	return gcc.markFailedAndDeletePodWithCondition(ctx, pod, nil)
}

func (gcc *PodGCController) markFailedAndDeletePodWithCondition(ctx context.Context, pod *v1.Pod, condition *v1.PodCondition) error {
	logger := klog.FromContext(ctx)
	logger.Info("PodGC is force deleting Pod", "pod", klog.KObj(pod))
	// Patch the pod to make sure it is transitioned to the Failed phase before deletion.
	// This is needed for the JobPodReplacementPolicy feature to make sure Job replacement pods are created.
	// See https://github.com/kubernetes/enhancements/tree/master/keps/sig-apps/3939-allow-replacement-when-fully-terminated#risks-and-mitigations
	// for more details.
	if utilfeature.DefaultFeatureGate.Enabled(features.PodDisruptionConditions) || utilfeature.DefaultFeatureGate.Enabled(features.JobPodReplacementPolicy) {

		// Mark the pod as failed - this is especially important in case the pod
		// is orphaned, in which case the pod would remain in the Running phase
		// forever as there is no kubelet running to change the phase.
		if pod.Status.Phase != v1.PodSucceeded && pod.Status.Phase != v1.PodFailed {
			newStatus := pod.Status.DeepCopy()
			newStatus.Phase = v1.PodFailed
			if condition != nil && utilfeature.DefaultFeatureGate.Enabled(features.PodDisruptionConditions) {
				apipod.UpdatePodCondition(newStatus, condition)
			}
			if _, _, _, err := utilpod.PatchPodStatus(ctx, gcc.kubeClient, pod.Namespace, pod.Name, pod.UID, pod.Status, *newStatus); err != nil {
				return err
			}
		}
	}
    // 调用 client 的 Delete 方法强制删除 pod
	return gcc.kubeClient.CoreV1().Pods(pod.Namespace).Delete(ctx, pod.Name, *metav1.NewDeleteOptions(0))
}
```
##### gcTerminating
删除处于 terminating 状态的 pod
```go
func (gcc *PodGCController) gcTerminating(ctx context.Context, pods []*v1.Pod) {
	logger := klog.FromContext(ctx)
	logger.V(4).Info("GC'ing terminating pods that are on out-of-service nodes")
	terminatingPods := []*v1.Pod{}
	for _, pod := range pods {
        // pod 的 deletion timestamp 不为 nil，说明处于 terminating 状态
		if isPodTerminating(pod) {
			node, err := gcc.nodeLister.Get(pod.Spec.NodeName)
			if err != nil {
				logger.Error(err, "Failed to get node", "node", klog.KRef("", pod.Spec.NodeName))
				continue
			}
            // 如果 pod 绑定在非 ready 状态的 node 上，或者 node 存在 out-of-service 污点，那么就标记为需要删除。
			// Add this pod to terminatingPods list only if the following conditions are met:
			// 1. Node is not ready.
			// 2. Node has `node.kubernetes.io/out-of-service` taint.
			if !nodeutil.IsNodeReady(node) && taints.TaintKeyExists(node.Spec.Taints, v1.TaintNodeOutOfService) {
				logger.V(4).Info("Garbage collecting pod that is terminating", "pod", klog.KObj(pod), "phase", pod.Status.Phase)
				terminatingPods = append(terminatingPods, pod)
			}
		}
	}

	deleteCount := len(terminatingPods)
	if deleteCount == 0 {
		return
	}

	logger.V(4).Info("Garbage collecting pods that are terminating on node tainted with node.kubernetes.io/out-of-service", "numPods", deleteCount)
	// sort only when necessary
	sort.Sort(byEvictionAndCreationTimestamp(terminatingPods))
	var wait sync.WaitGroup
	for i := 0; i < deleteCount; i++ {
		wait.Add(1)
		go func(pod *v1.Pod) {
			defer wait.Done()
			metrics.DeletingPodsTotal.WithLabelValues(pod.Namespace, metrics.PodGCReasonTerminatingOutOfService).Inc()
			if err := gcc.markFailedAndDeletePod(ctx, pod); err != nil {
				// ignore not founds
				utilruntime.HandleError(err)
				metrics.DeletingPodsErrorTotal.WithLabelValues(pod.Namespace, metrics.PodGCReasonTerminatingOutOfService).Inc()
			}
		}(terminatingPods[i])
	}
	wait.Wait()
}
```
##### gcOrphan
删除那些绑定节点已经不存在的 pod
```go
func (gcc *PodGCController) gcOrphaned(ctx context.Context, pods []*v1.Pod, nodes []*v1.Node) {
	logger := klog.FromContext(ctx)
	logger.V(4).Info("GC'ing orphaned")
    // node 集合，快速判断节点是否存在
	existingNodeNames := sets.NewString()
	for _, node := range nodes {
		existingNodeNames.Insert(node.Name)
	}
	// Add newly found unknown nodes to quarantine
	for _, pod := range pods {
        // 如果 pod 绑定了节点，并且上面的节点集合中不存在 pod 所绑定的这个节点，则加入到延时队列中
		if pod.Spec.NodeName != "" && !existingNodeNames.Has(pod.Spec.NodeName) {
            // 使用延时队列的作用是等待一段时间，即 quarantineTime，以防节点短暂不可用时误删 pod
			gcc.nodeQueue.AddAfter(pod.Spec.NodeName, gcc.quarantineTime)
		}
	}
    // 检查隔离期后仍然缺失的节点，discoverDeletedNodes 看下面解释
	// Check if nodes are still missing after quarantine period
	deletedNodesNames, quit := gcc.discoverDeletedNodes(ctx, existingNodeNames)
	if quit {
		return
	}
    // 遍历所有的 pod
	// Delete orphaned pods
	for _, pod := range pods {
        // 判断 pod.spec 中 nodeName 是否在 deletedNodesNames 中，如果存在，则说明该 pod 是一个孤儿 pod，否则跳过
		if !deletedNodesNames.Has(pod.Spec.NodeName) {
			continue
		}
		logger.V(2).Info("Found orphaned Pod assigned to the Node, deleting", "pod", klog.KObj(pod), "node", klog.KRef("", pod.Spec.NodeName))
		condition := &v1.PodCondition{
			Type:    v1.DisruptionTarget,
			Status:  v1.ConditionTrue,
			Reason:  "DeletionByPodGC",
			Message: "PodGC: node no longer exists",
		}
        // 跟之前的逻辑一样，标记并强制删除
		if err := gcc.markFailedAndDeletePodWithCondition(ctx, pod, condition); err != nil {
			utilruntime.HandleError(err)
			metrics.DeletingPodsErrorTotal.WithLabelValues(pod.Namespace, metrics.PodGCReasonOrphaned).Inc()
		} else {
			logger.Info("Forced deletion of orphaned Pod succeeded", "pod", klog.KObj(pod))
		}
		metrics.DeletingPodsTotal.WithLabelValues(pod.Namespace, metrics.PodGCReasonOrphaned).Inc()
	}
}
```

##### discoverDeletedNodes
这个函数传入存在的 existingNodeNames 集合，这个集合来自于 node informer 缓存。
返回已经删除的 deletedNodesNames 集合，集合中的内容是 pod informer 中绑定但是实际上已经不存在的 node
```go
func (gcc *PodGCController) discoverDeletedNodes(ctx context.Context, existingNodeNames sets.String) (sets.String, bool) {
    // 要返回的 node 集合
	deletedNodesNames := sets.NewString()
    // node 延迟队列的长度
	for gcc.nodeQueue.Len() > 0 {
        // 从前往后遍历队列，当 queue 关闭时返回 quit 为 true
		item, quit := gcc.nodeQueue.Get()
		if quit {
			return nil, true
		}
		nodeName := item.(string)
        // 如果已知的 node 集合中没有这个 node
		if !existingNodeNames.Has(nodeName) {
            // 从 apiserver 中检查该 node 是否存在
			exists, err := gcc.checkIfNodeExists(ctx, nodeName)
			switch {
			case err != nil:
				klog.FromContext(ctx).Error(err, "Error while getting node", "node", klog.KRef("", nodeName))
				// Node will be added back to the queue in the subsequent loop if still needed
			case !exists:
                // 如果不存在，说明 node 已经被删除了，加入到集合中返回
				deletedNodesNames.Insert(nodeName)
			}
		}
		gcc.nodeQueue.Done(item)
	}
	return deletedNodesNames, false
}
```
##### gcUnscheduledTerminating
删除所有的已经处于删除状态但还没有调度到任何节点的 pod
```go
// gcUnscheduledTerminating deletes pods that are terminating and haven't been scheduled to a particular node.
func (gcc *PodGCController) gcUnscheduledTerminating(ctx context.Context, pods []*v1.Pod) {
	logger := klog.FromContext(ctx)
	logger.V(4).Info("GC'ing unscheduled pods which are terminating")

	for _, pod := range pods {
        // pod 属性判断
		if pod.DeletionTimestamp == nil || len(pod.Spec.NodeName) > 0 {
			continue
		}

		logger.V(2).Info("Found unscheduled terminating Pod not assigned to any Node, deleting", "pod", klog.KObj(pod))
		if err := gcc.markFailedAndDeletePod(ctx, pod); err != nil {
			utilruntime.HandleError(err)
			metrics.DeletingPodsErrorTotal.WithLabelValues(pod.Namespace, metrics.PodGCReasonTerminatingUnscheduled).Inc()
		} else {
			logger.Info("Forced deletion of unscheduled terminating Pod succeeded", "pod", klog.KObj(pod))
		}
		metrics.DeletingPodsTotal.WithLabelValues(pod.Namespace, metrics.PodGCReasonTerminatingUnscheduled).Inc()
	}
}
```
#### 总结
pod gc 依据 podInformer 和 nodeInformer 的数据，对 pod 状态和 node 状态进行判断，依次删除 terminated、terminating pod，再删除孤儿 pod。最后清理已经处于删除状态但还没有调度到任何节点的 pod。
要注意 gcc 中的 nodeQueue 中仅存储 pod 信息中已绑定但是 nodeInformer 中不存在的 node 名称。