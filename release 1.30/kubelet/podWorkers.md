### podWorkers
podWorker触发的三种状态的事件
- syncing: pod应当处于运行状态(syncPod)
- terminating: pod应当被停止(syncTerminatingPod)
- terminated: pod关联的所有资源应当被删除(syncTerminatedPod)

podWorkers持续追踪对pod的操作并确保每个pod被容器运行时和其他子系统协调。podWorkers同时也追踪哪些pod正在启动，哪些正在关闭但是仍然有处于运行状态的容器，哪些pod最近已经终止了并保证没有处于运行中的容器。
podWorkers是任何时候应该在节点上活跃的真实来源，并且可以通过 UpdatePod 方法根据节点的期望状态保持最新状态（由kubelet pod configs 循环和kubelet中podManager中的状态跟踪）。
在运行Pod时其他组件应当在podWorker中寻找运行状态pod的状态，而不是在kubelet podManager中。
podWorker会通过 SyncKnownPods() 方法与podManager定期协调状态，并最终负责完成所有观察到的不再存在于podManager中（不再是节点期望状态的一部分）。

传递到podWorker中的pod是synced（期望运行），终止中（有运行中的容器但是没有新的容器需要启动），已终止（没有运行中的容器，但是可能仍然有资源在被使用）或者清除（没有资源存在）。
一旦一个pod被设置为tear down，这个pod就无法再使用原来的UID（关联了删除或者驱逐动作）再次启动，直到podWorker已经完成工作（SyncTerminatingPod和syncTerminatedPod依次退出且没有报错），SyncKnownPods方法被kubelet的housekeeping调用并且这个pod不再是已知配置文件的一部分。

podWorkers向kubelet的其他循环提供了一致的信息来源，包括了解pod状态和容器是否应该被运行。
ShouldPodContentBeRemoved() 方法跟踪了pod的内容是否仍然应当存在，这包括了在SyncKnownPods()被调用一次后后不存在的pod（根据约定，所有存在的pod应该在SyncKnownPods方法被调用前通过UpdatePod方法提供）。通常，其他同步循环期望分离 setup 和 teardown 的责任，此处的信息方法通过集中改状态来帮助每种状态，简单的事件周期虚拟图如下：
```go
// PodWorkers is an abstract interface for testability.
type PodWorkers interface {
    // UpdatePod 告诉pod worker pod的变更，这些变更会在每个podUID对应的goroutine中以先进先出的顺序进行处理。
    // pod的状态会被传到syncPod方法，除非pod被标记为已删除，到了终止阶段（succeed或failed状态）或者被kubelet主动驱逐。
    // 一旦上面的情况发生，syncTerminatingPod 方法就会被调用，直到成功退出
    // 并且从这之后对当前pod的所有的UpdatePod() 调用都会被忽略，直到签名时间结束。被终止的pod永远不会被重启。
	// UpdatePod notifies the pod worker of a change to a pod, which will then
	// be processed in FIFO order by a goroutine per pod UID. The state of the
	// pod will be passed to the syncPod method until either the pod is marked
	// as deleted, it reaches a terminal phase (Succeeded/Failed), or the pod
	// is evicted by the kubelet. Once that occurs the syncTerminatingPod method
	// will be called until it exits successfully, and after that all further
	// UpdatePod() calls will be ignored for that pod until it has been forgotten
	// due to significant time passing. A pod that is terminated will never be
	// restarted.
	UpdatePod(options UpdatePodOptions)
    // SyncKnownPods 移除那些不在期望pod清单中的或者因为有效期已经结束的pods的workers。一旦这个方法被调用一次，workers就假设已经被完全初始化并且随后对于未知pod的ShouldPodContentBeRemoved方法调用会返回true。
    // SyncKnownPods返回一个map，描述了每一个已知的pod worker的状态。调用者有责任重新添加任何不在返回的 knownPods 中的期望 Pod.
	// SyncKnownPods removes workers for pods that are not in the desiredPods set
	// and have been terminated for a significant period of time. Once this method
	// has been called once, the workers are assumed to be fully initialized and
	// subsequent calls to ShouldPodContentBeRemoved on unknown pods will return
	// true. It returns a map describing the state of each known pod worker. It
	// is the responsibility of the caller to re-add any desired pods that are not
	// returned as knownPods.
	SyncKnownPods(desiredPods []*v1.Pod) (knownPods map[types.UID]PodWorkerSync)

    // 只要SyncTerminatingPod方法成功完成，IsPodKnownTerminated就返回true，podWorker 知道传入的参数podUID应当被终止。
    // 如果pod已经被强制删除并且podWorker已经完成终止，这个方法会返回false，因此这个方法仅用于筛选出不满足条件的pod，例如准入控制。
    // 倾向于由kubelet config loop使用而不是子系统，如果是子系统调用，应该使用ShouldPod*()方法。
	// IsPodKnownTerminated returns true once SyncTerminatingPod completes
	// successfully - the provided pod UID it is known by the pod
	// worker to be terminated. If the pod has been force deleted and the pod worker
	// has completed termination this method will return false, so this method should
	// only be used to filter out pods from the desired set such as in admission.
	//
	// Intended for use by the kubelet config loops, but not subsystems, which should
	// use ShouldPod*().
	IsPodKnownTerminated(uid types.UID) bool
    // 一旦pod workers接收到了pod（调用syncPod），在pod worker已经同步对应UIID的pod之前返回true，在终止pod之后返回false（保证运行中的容器已经停止运行）
	// CouldHaveRunningContainers returns true before the pod workers have synced,
	// once the pod workers see the pod (syncPod could be called), and returns false
	// after the pod has been terminated (running containers guaranteed stopped).
	//
	// Intended for use by the kubelet config loops, but not subsystems, which should
	// use ShouldPod*().
	CouldHaveRunningContainers(uid types.UID) bool

	// ShouldPodBeFinished returns true once SyncTerminatedPod completes
	// successfully - the provided pod UID it is known to the pod worker to
	// be terminated and have resources reclaimed. It returns false before the
	// pod workers have synced (syncPod could be called). Once the pod workers
	// have synced it returns false if the pod has a sync status until
	// SyncTerminatedPod completes successfully. If the pod workers have synced,
	// but the pod does not have a status it returns true.
	//
	// Intended for use by subsystem sync loops to avoid performing background setup
	// after termination has been requested for a pod. Callers must ensure that the
	// syncPod method is non-blocking when their data is absent.
	ShouldPodBeFinished(uid types.UID) bool
	// IsPodTerminationRequested returns true when pod termination has been requested
	// until the termination completes and the pod is removed from config. This should
	// not be used in cleanup loops because it will return false if the pod has already
	// been cleaned up - use ShouldPodContainersBeTerminating instead. Also, this method
	// may return true while containers are still being initialized by the pod worker.
	//
	// Intended for use by the kubelet sync* methods, but not subsystems, which should
	// use ShouldPod*().
	IsPodTerminationRequested(uid types.UID) bool

	// ShouldPodContainersBeTerminating returns false before pod workers have synced,
	// or once a pod has started terminating. This check is similar to
	// ShouldPodRuntimeBeRemoved but is also true after pod termination is requested.
	//
	// Intended for use by subsystem sync loops to avoid performing background setup
	// after termination has been requested for a pod. Callers must ensure that the
	// syncPod method is non-blocking when their data is absent.
	ShouldPodContainersBeTerminating(uid types.UID) bool
	// ShouldPodRuntimeBeRemoved returns true if runtime managers within the Kubelet
	// should aggressively cleanup pod resources that are not containers or on disk
	// content, like attached volumes. This is true when a pod is not yet observed
	// by a worker after the first sync (meaning it can't be running yet) or after
	// all running containers are stopped.
	// TODO: Once pod logs are separated from running containers, this method should
	// be used to gate whether containers are kept.
	//
	// Intended for use by subsystem sync loops to know when to start tearing down
	// resources that are used by running containers. Callers should ensure that
	// runtime content they own is not required for post-termination - for instance
	// containers are required in docker to preserve pod logs until after the pod
	// is deleted.
	ShouldPodRuntimeBeRemoved(uid types.UID) bool
	// ShouldPodContentBeRemoved returns true if resource managers within the Kubelet
	// should aggressively cleanup all content related to the pod. This is true
	// during pod eviction (when we wish to remove that content to free resources)
	// as well as after the request to delete a pod has resulted in containers being
	// stopped (which is a more graceful action). Note that a deleting pod can still
	// be evicted.
	//
	// Intended for use by subsystem sync loops to know when to start tearing down
	// resources that are used by non-deleted pods. Content is generally preserved
	// until deletion+removal_from_etcd or eviction, although garbage collection
	// can free content when this method returns false.
	ShouldPodContentBeRemoved(uid types.UID) bool
	// IsPodForMirrorPodTerminatingByFullName returns true if a static pod with the
	// provided pod name is currently terminating and has yet to complete. It is
	// intended to be used only during orphan mirror pod cleanup to prevent us from
	// deleting a terminating static pod from the apiserver before the pod is shut
	// down.
	IsPodForMirrorPodTerminatingByFullName(podFullname string) bool
}
type podWorkers struct {
	// Protects all per worker fields.
	podLock sync.Mutex
    // pods是否已经同步，只要podWorker同步完成至少一次就是true，表示所有的pods已经通过UpdatePod()启动了
	// podsSynced is true once the pod worker has been synced at least once,
	// which means that all working pods have been started via UpdatePod().
	podsSynced bool

    // 跟踪所有的goroutine，每个pod都有自己的goroutine，这些goroutine会处理来自关联的channel发送的update事件。向这个channel中发送消息就表示关联的goroutine消费podSyncStatuses[uid].pendingUpdate
	// Tracks all running per-pod goroutines - per-pod goroutine will be
	// processing updates received through its corresponding channel. Sending
	// a message on this channel will signal the corresponding goroutine to
	// consume podSyncStatuses[uid].pendingUpdate if set.
	podUpdates map[types.UID]chan struct{}
    // 通过UID跟踪pod的状态-同步、终止、已终止和被驱逐
	// Tracks by UID the termination status of a pod - syncing, terminating,
	// terminated, and evicted.
	podSyncStatuses map[types.UID]*podSyncStatus

    // 根据pod全名来追踪已启动的静态podUID
	// Tracks all uids for started static pods by full name
	startedStaticPodsByFullname map[string]types.UID
    // 根据pod全名来追踪待启动的静态podUID列表
	// Tracks all uids for static pods that are waiting to start by full name
	waitingToStartStaticPodsByFullname map[string][]types.UID
    // 工作队列
	workQueue queue.WorkQueue
    // 包含SyncPod，SyncTerminatingPod，SyncTerminatingRuntimePod和SyncTerminatedPod四种方法
	// This function is run to sync the desired state of pod.
	// NOTE: This function has to be thread-safe - it can be called for
	// different pods at the same time.
	podSyncer podSyncer

	// workerChannelFn is exposed for testing to allow unit tests to impose delays
	// in channel communication. The function is invoked once each time a new worker
	// goroutine starts.
	workerChannelFn func(uid types.UID, in chan struct{}) (out <-chan struct{})

	// The EventRecorder to use
	recorder record.EventRecorder

	// backOffPeriod is the duration to back off when there is a sync error.
	backOffPeriod time.Duration

	// resyncInterval is the duration to wait until the next sync.
	resyncInterval time.Duration
    // 缓存
	// podCache stores kubecontainer.PodStatus for all pods.
	podCache kubecontainer.Cache

	// clock is used for testing timing
	clock clock.PassiveClock
}
```

#### 删除Pod
```go
// 当从configCh读到的事件的操作是kubetypes.UPDATE或者kubetypes.DELETE都走这个流程
// HandlePodUpdates is the callback in the SyncHandler interface for pods
// being updated from a config source.
func (kl *Kubelet) HandlePodUpdates(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
        // 先更新podManager缓存
		kl.podManager.UpdatePod(pod)
        // 从podManager中获取所有的pod
		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		if wasMirror {
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
		}
        // 交给podWorker处理
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			Pod:        pod,
			MirrorPod:  mirrorPod,
			UpdateType: kubetypes.SyncPodUpdate,
			StartTime:  start,
		})
	}
}
```

#### UpdatePod(Add相关的逻辑在kubelet.md中讲过，这里再统一看一下)
```go
// pkg/kubelet/pod_workers.go:735
// UpdatePod携带了pod的配置变更或者终止状态。pod要么是运行中的、终止中或者已终止的，并且如果这个pod在apiserver中被删除了、被kubelet驱逐了、或者被发现处于已终止状态（succeed或failed）将过渡到终止中状态。
// UpdatePod carries a configuration change or termination state to a pod. A pod is either runnable,
// terminating, or terminated, and will transition to terminating if: deleted on the apiserver,
// discovered to have a terminal phase (Succeeded or Failed), or evicted by the kubelet.
func (p *podWorkers) UpdatePod(options UpdatePodOptions) {
	// Handle when the pod is an orphan (no config) and we only have runtime status by running only
	// the terminating part of the lifecycle. A running pod contains only a minimal set of information
	// about the pod
	var isRuntimePod bool
	var uid types.UID
	var name, ns string
    // RunningPod是指那些已经不存在于配置中的pod，当pod为空时生效，如果pod有值则忽略，runningPod对应的动作只有kill，其他的忽略
	if runningPod := options.RunningPod; runningPod != nil {
		if options.Pod == nil {
			// the sythetic pod created here is used only as a placeholder and not tracked
			if options.UpdateType != kubetypes.SyncPodKill {
				klog.InfoS("Pod update is ignored, runtime pods can only be killed", "pod", klog.KRef(runningPod.Namespace, runningPod.Name), "podUID", runningPod.ID, "updateType", options.UpdateType)
				return
			}
			uid, ns, name = runningPod.ID, runningPod.Namespace, runningPod.Name
			isRuntimePod = true
		} else {
			options.RunningPod = nil
			uid, ns, name = options.Pod.UID, options.Pod.Namespace, options.Pod.Name
			klog.InfoS("Pod update included RunningPod which is only valid when Pod is not specified", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
		}
	} else {
		uid, ns, name = options.Pod.UID, options.Pod.Namespace, options.Pod.Name
	}

	p.podLock.Lock()
	defer p.podLock.Unlock()

	// decide what to do with this pod - we are either setting it up, tearing it down, or ignoring it
	var firstTime bool
	now := p.clock.Now()
	status, ok := p.podSyncStatuses[uid]
    // 如果该pod的同步状态没有找到，说明是第一次同步，对应podAddAddition
	if !ok {
		klog.V(4).InfoS("Pod is being synced for the first time", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
		firstTime = true
        // 初始化数据
		status = &podSyncStatus{
			syncedAt: now,
			fullname: kubecontainer.BuildPodFullName(name, ns),
		}
        // 如果pod是第一次同步，不是runningPod，而且pod处于不活跃状态，即处于failed或succeed状态。进入这个判断说明这个pod已经被终止了，则更新pod状态
		// if this pod is being synced for the first time, we need to make sure it is an active pod
		if options.Pod != nil && (options.Pod.Status.Phase == v1.PodFailed || options.Pod.Status.Phase == v1.PodSucceeded) {
			// Check to see if the pod is not running and the pod is terminal; if this succeeds then record in the podWorker that it is terminated.
			// This is needed because after a kubelet restart, we need to ensure terminal pods will NOT be considered active in Pod Admission. See http://issues.k8s.io/105523
			// However, `filterOutInactivePods`, considers pods that are actively terminating as active. As a result, `IsPodKnownTerminated()` needs to return true and thus `terminatedAt` needs to be set.
			if statusCache, err := p.podCache.Get(uid); err == nil {
                // 从configCh中传入的pod实例来检查这个pod是否已经被终止，运行时缓存中这个pod也已经被终止，条件是pod中所有容器的状态都不是running并且sandbox的状态不是ready
                // 意味着这个pod在过去已经完成SyncTerminatingPod处理。有可能是因为kubelet的重启导致该处于终止状态的pod再次被重新同步，并且缓存中不存在，判定为第一次同步。
				if isPodStatusCacheTerminal(statusCache) {
					// At this point we know:
					// (1) The pod is terminal based on the config source.
					// (2) The pod is terminal based on the runtime cache.
					// This implies that this pod had already completed `SyncTerminatingPod` sometime in the past. The pod is likely being synced for the first time due to a kubelet restart.
					// These pods need to complete SyncTerminatedPod to ensure that all resources are cleaned and that the status manager makes the final status updates for the pod.
					// As a result, set finished: false, to ensure a Terminated event will be sent and `SyncTerminatedPod` will run.
					status = &podSyncStatus{
						terminatedAt:       now,
						terminatingAt:      now,
						syncedAt:           now,
						startedTerminating: true,
						finished:           false,
						fullname:           kubecontainer.BuildPodFullName(name, ns),
					}
				}
			}
		}
		p.podSyncStatuses[uid] = status
	}
    // RunningPods代表了未知的pod操作并且不包括pod配置信息以执行除终止以外的任何操作。
    // 如果在真实的pod之后接收到了RunningPod，那么使用最近一次的配置来代替传入的配置。
    // 一旦观察到了运行时pod就必须驱动它完成，即使这个pod不是我们启动的
	// RunningPods represent an unknown pod execution and don't contain pod spec information
	// sufficient to perform any action other than termination. If we received a RunningPod
	// after a real pod has already been provided, use the most recent spec instead. Also,
	// once we observe a runtime pod we must drive it to completion, even if we weren't the
	// ones who started it.
	pod := options.Pod
	if isRuntimePod {
		status.observedRuntime = true
		switch {
		case status.pendingUpdate != nil && status.pendingUpdate.Pod != nil:
			pod = status.pendingUpdate.Pod
			options.Pod = pod
			options.RunningPod = nil
		case status.activeUpdate != nil && status.activeUpdate.Pod != nil:
			pod = status.activeUpdate.Pod
			options.Pod = pod
			options.RunningPod = nil
		default:
            // 只转化配置信息，不包含状态信息
			// we will continue to use RunningPod.ToAPIPod() as pod here, but
			// options.Pod will be nil and other methods must handle that appropriately.
			pod = options.RunningPod.ToAPIPod()
		}
	}

    // 如果不是第一次同步，pod已经接到了终止请求，并且操作类型为新建pod，判定其为recreate请求
    // 当我们在已终止的pod上看到创建更新是，这意味着在短时间内创建了两个具有相同UID的pod（通常是静态pod，但apiserver很少会执行类似操作）
    // 标记同步状态以指示在pod终止后将其重置为未运行，以允许后续添加/更新再次启动podWorker。
    // 这不适用于我们第一次看到pod的情况，例如当kubelet重新启动时，我们第一次看到已经终止的pod
	// When we see a create update on an already terminating pod, that implies two pods with the same UID were created in
	// close temporal proximity (usually static pod but it's possible for an apiserver to extremely rarely do something
	// similar) - flag the sync status to indicate that after the pod terminates it should be reset to "not running" to
	// allow a subsequent add/update to start the pod worker again. This does not apply to the first time we see a pod,
	// such as when the kubelet restarts and we see already terminated pods for the first time.
	if !firstTime && status.IsTerminationRequested() {
		if options.UpdateType == kubetypes.SyncPodCreate {
			status.restartRequested = true
			klog.V(4).InfoS("Pod is terminating but has been requested to restart with same UID, will be reconciled later", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			return
		}
	}
    // 一旦pod被终止，就无法再次进入podWorker处理，直接返回，后续由housekeeping进行清理
	// once a pod is terminated by UID, it cannot reenter the pod worker (until the UID is purged by housekeeping)
	if status.IsFinished() {
		klog.V(4).InfoS("Pod is finished processing, no further updates", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
		return
	}

    // 检查终止状态的转换，主要是为了打日志
	// check for a transition to terminating
	var becameTerminating bool
	if !status.IsTerminationRequested() {
		switch {
        // 判断是否是孤儿pod，标记删除
		case isRuntimePod:
			klog.V(4).InfoS("Pod is orphaned and must be torn down", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			// 代表apiserver上已经删除
            status.deleted = true
            // 现在开始终止
			status.terminatingAt = now
            // 进入终止状态
			becameTerminating = true
		case pod.DeletionTimestamp != nil:
			klog.V(4).InfoS("Pod is marked for graceful deletion, begin teardown", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			status.deleted = true
			status.terminatingAt = now
			becameTerminating = true
		case pod.Status.Phase == v1.PodFailed, pod.Status.Phase == v1.PodSucceeded:
			klog.V(4).InfoS("Pod is in a terminal phase (success/failed), begin teardown", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			status.terminatingAt = now
			becameTerminating = true
		case options.UpdateType == kubetypes.SyncPodKill:
			if options.KillPodOptions != nil && options.KillPodOptions.Evict {
				klog.V(4).InfoS("Pod is being evicted by the kubelet, begin teardown", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
				status.evicted = true
			} else {
				klog.V(4).InfoS("Pod is being removed by the kubelet, begin teardown", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			}
			status.terminatingAt = now
			becameTerminating = true
		}
	}

	// once a pod is terminating, all updates are kills and the grace period can only decrease
	var wasGracePeriodShortened bool
	switch {
	case status.IsTerminated():
        // 孤儿pod直接忽略
		// A terminated pod may still be waiting for cleanup - if we receive a runtime pod kill request
		// due to housekeeping seeing an older cached version of the runtime pod simply ignore it until
		// after the pod worker completes.
		if isRuntimePod {
			klog.V(3).InfoS("Pod is waiting for termination, ignoring runtime-only kill until after pod worker is fully terminated", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			return
		}

		if options.KillPodOptions != nil {
			if ch := options.KillPodOptions.CompletedCh; ch != nil {
				close(ch)
			}
		}
		options.KillPodOptions = nil

	case status.IsTerminationRequested():
		if options.KillPodOptions == nil {
			options.KillPodOptions = &KillPodOptions{}
		}

		if ch := options.KillPodOptions.CompletedCh; ch != nil {
			status.notifyPostTerminating = append(status.notifyPostTerminating, ch)
		}
		if fn := options.KillPodOptions.PodStatusFunc; fn != nil {
			status.statusPostTerminating = append(status.statusPostTerminating, fn)
		}

        // 判断优雅删除的时间，看下面calculateEffectiveGracePeriod
		gracePeriod, gracePeriodShortened := calculateEffectiveGracePeriod(status, pod, options.KillPodOptions)

		wasGracePeriodShortened = gracePeriodShortened
		status.gracePeriod = gracePeriod
		// always set the grace period for syncTerminatingPod so we don't have to recalculate,
		// will never be zero.
		options.KillPodOptions.PodTerminationGracePeriodSecondsOverride = &gracePeriod

	default:
		// KillPodOptions is not valid for sync actions outside of the terminating phase
		if options.KillPodOptions != nil {
			if ch := options.KillPodOptions.CompletedCh; ch != nil {
				close(ch)
			}
			options.KillPodOptions = nil
		}
	}
    // 如果uid对应的podWorker不存在则创建podWorker
	// start the pod worker goroutine if it doesn't exist
	podUpdates, exists := p.podUpdates[uid]
	if !exists {
		// buffer the channel to avoid blocking this method
		podUpdates = make(chan struct{}, 1)
		p.podUpdates[uid] = podUpdates

        // 如果该 Pod 是一个静态 Pod（即由文件定义而不是通过 API 服务器创建），则将其添加到 p.waitingToStartStaticPodsByFullname 列表中。这确保了静态 Pod 按接收到的顺序启动
		// ensure that static pods start in the order they are received by UpdatePod
		if kubetypes.IsStaticPod(pod) {
			p.waitingToStartStaticPodsByFullname[status.fullname] =
				append(p.waitingToStartStaticPodsByFullname[status.fullname], uid)
		}

        // 这部分代码允许在测试中使用自定义的通道函数 p.workerChannelFn，以便模拟不同的通道行为。如果没有自定义函数，则直接使用 podUpdates 通道。
		// allow testing of delays in the pod update channel
		var outCh <-chan struct{}
		if p.workerChannelFn != nil {
			outCh = p.workerChannelFn(uid, podUpdates)
		} else {
			outCh = podUpdates
		}

        // 启动一个新的 goroutine 来运行 p.podWorkerLoop
		// spawn a pod worker
		go func() {
			// TODO: this should be a wait.Until with backoff to handle panics, and
			// accept a context for shutdown
			defer runtime.HandleCrash()
			defer klog.V(3).InfoS("Pod worker has stopped", "podUID", uid)
			p.podWorkerLoop(uid, outCh)
		}()
	}

    // 这部分代码检查是否存在未处理的 pendingUpdate，并且其 StartTime 早于当前更新请求的 StartTime。如果是这样，将 options.StartTime 更新为 pendingUpdate.StartTime。这样可以记录从 UpdatePod 调用到 Pod worker 实际处理之间的最大延迟。
	// measure the maximum latency between a call to UpdatePod and when the pod worker reacts to it
	// by preserving the oldest StartTime
	if status.pendingUpdate != nil && !status.pendingUpdate.StartTime.IsZero() && status.pendingUpdate.StartTime.Before(options.StartTime) {
		options.StartTime = status.pendingUpdate.StartTime
	}

	// notify the pod worker there is a pending update
	status.pendingUpdate = &options
	status.working = true
	klog.V(4).InfoS("Notifying pod of pending update", "pod", klog.KRef(ns, name), "podUID", uid, "workType", status.WorkType())
	select {
    // 向 podUpdates 通道发送一个空结构体 struct{}{} 来通知 Pod worker 有待处理的更新
	case podUpdates <- struct{}{}:
	default:
	}

    // 是否需要取消当前的 Pod 同步操作。如果 Pod 进入 Terminating 状态或者宽限期（grace period）被缩短，并且存在取消函数 status.cancelFn，则调用该取消函数来中断当前的同步操作。
	if (becameTerminating || wasGracePeriodShortened) && status.cancelFn != nil {
		klog.V(3).InfoS("Cancelling current pod sync", "pod", klog.KRef(ns, name), "podUID", uid, "workType", status.WorkType())
		status.cancelFn()
		return
	}
}
```
#### calculateEffectiveGracePeriod
```go
func calculateEffectiveGracePeriod(status *podSyncStatus, pod *v1.Pod, options *KillPodOptions) (int64, bool) {
	// enforce the restriction that a grace period can only decrease and track whatever our value is,
	// then ensure a calculated value is passed down to lower levels
	gracePeriod := status.gracePeriod
    // 如果这个值是从pod配置中传过来的且小于默认的值，比如kubectl操作，那就用这个值覆盖
	// this value is bedrock truth - the apiserver owns telling us this value calculated by apiserver
	if override := pod.DeletionGracePeriodSeconds; override != nil {
		if gracePeriod == 0 || *override < gracePeriod {
			gracePeriod = *override
		}
	}
    // 如果是kubelet的驱逐操作导致的删除，如果gracePeriod等于0或者传入的值小于gracePeriod值，则使用较小的那个值
	// we allow other parts of the kubelet (namely eviction) to request this pod be terminated faster
	if options != nil {
		if override := options.PodTerminationGracePeriodSecondsOverride; override != nil {
			if gracePeriod == 0 || *override < gracePeriod {
				gracePeriod = *override
			}
		}
	}
    // 判断优雅时间是否为0，如果为0并且pod的配置文件删除时间不为空，则替换
	// make a best effort to default this value to the pod's desired intent, in the event
	// the kubelet provided no requested value (graceful termination?)
	if gracePeriod == 0 && pod.Spec.TerminationGracePeriodSeconds != nil {
		gracePeriod = *pod.Spec.TerminationGracePeriodSeconds
	}
    // 如果最终的优雅删除时间小于1，则置为1
	// no matter what, we always supply a grace period of 1
	if gracePeriod < 1 {
		gracePeriod = 1
	}
	return gracePeriod, status.gracePeriod != 0 && status.gracePeriod != gracePeriod
}
```
#### startPodSync
```go
// pkg/kubelet/pod_workers.go:1098
// 当变更事件道道podUpdate channel时每个podWorker的goroutine中会调用这个方法。
// 该方法消费了传入的update事件，初始化上下文，决定pod是否已经被启动或者可以被启动，并且更新缓存中的pod状态用于下游组件访问获取最新的pod状态。
// 如果返回的最后一个值时false，确保合适的清理工作在返回前已经执行过了。
// 该方法应该确保status.pendingUpdate被清空并且合并到status.activeUpdate队列，或者当pod无法被启动时保留在status.pendingUpdate中。
// 没有被启动的pod永远不应该存在于activeUpdate队列中，因为这个队列向向下游暴露了所有已启动的pod。
// startPodSync is invoked by each pod worker goroutine when a message arrives on the pod update channel.
// This method consumes a pending update, initializes a context, decides whether the pod is already started
// or can be started, and updates the cached pod state so that downstream components can observe what the
// pod worker goroutine is currently attempting to do. If ok is false, there is no available event. If any
// of the boolean values is false, ensure the appropriate cleanup happens before returning.
//
// This method should ensure that either status.pendingUpdate is cleared and merged into status.activeUpdate,
// or when a pod cannot be started status.pendingUpdate remains the same. Pods that have not been started
// should never have an activeUpdate because that is exposed to downstream components on started pods.
func (p *podWorkers) startPodSync(podUID types.UID) (ctx context.Context, update podWork, canStart, canEverStart, ok bool) {
	p.podLock.Lock()
	defer p.podLock.Unlock()

    // 根据podUID查询pod的同步状态，podSyncStatuses数据来源于UpdatePod
	// verify we are known to the pod worker still
	status, ok := p.podSyncStatuses[podUID]
    // 如果查不到，说明已经执行过删除操作了，podWorker应该停止
	if !ok {
		// pod status has disappeared, the worker should exit
		klog.V(4).InfoS("Pod worker no longer has status, worker should exit", "podUID", podUID)
		return nil, update, false, false, false
	}
    // 异常提示
	if !status.working {
		// working is used by unit tests to observe whether a worker is currently acting on this pod
		klog.V(4).InfoS("Pod should be marked as working by the pod worker, programmer error", "podUID", podUID)
	}
    // 异常
	if status.pendingUpdate == nil {
		// no update available, this means we were queued without work being added or there is a
		// race condition, both of which are unexpected
		status.working = false
		klog.V(4).InfoS("Pod worker received no pending work, programmer error?", "podUID", podUID)
		return nil, update, false, false, false
	}

    // 消费pendingUpdate队列，然后清空
	// consume the pending update
	update.WorkType = status.WorkType()
	update.Options = *status.pendingUpdate
	status.pendingUpdate = nil
	select {
    // 清空podUpdate对应podUID的channel，保证后续可以再写进来数据触发pod同步
	case <-p.podUpdates[podUID]:
		// ensure the pod update channel is empty (it is only ever written to under lock)
	default:
	}

    // 初始化podWorker的上下文
	// initialize a context for the worker if one does not exist
	if status.ctx == nil || status.ctx.Err() == context.Canceled {
		status.ctx, status.cancelFn = context.WithCancel(context.Background())
	}
	ctx = status.ctx

    // pod已经启动，更新状态并通知下游组件
	// if we are already started, make our state visible to downstream components
	if status.IsStarted() {
		status.mergeLastUpdate(update.Options)
		return ctx, update, true, true, true
	}

    // pod正在终止，允许将其状态标记为启动，以便可以立即过渡到终止状态。
	// if we are already terminating and we only have a running pod, allow the worker
	// to "start" since we are immediately moving to terminating
	if update.Options.RunningPod != nil && update.WorkType == TerminatingPod {
		status.mergeLastUpdate(update.Options)
		return ctx, update, true, true, true
	}

	// If we receive an update where Pod is nil (running pod is set) but haven't
	// started yet, we can only terminate the pod, not start it. We should not be
	// asked to start such a pod, but guard here just in case an accident occurs.
	if update.Options.Pod == nil {
		status.mergeLastUpdate(update.Options)
		klog.V(4).InfoS("Running pod cannot start ever, programmer error", "pod", klog.KObj(update.Options.Pod), "podUID", podUID, "updateType", update.WorkType)
		return ctx, update, false, false, true
	}

    // 验证pod是否可以启动，如果这个pod不是静态pod则都返回true
    // 通过podUID查询pod同步状态，如果没查到，说明没有执行过UpdatePod，返回false
    // 如果查到的状态已经请求了终止，返回false
    // 判断静态pod是否可以被启动，如果没有全名相同的静态pod已经启动并且podUID匹配待启动的静态pod则可以启动，反之不可以，之后重新入队列等候调度。这类pod不可以被启动，canstart为false，canEverStart为true，表示将来有可能被启动但是现在不可以启动
	// verify we can start
	canStart, canEverStart = p.allowPodStart(update.Options.Pod)
	switch {
    // 如果将来也不可以启动，则清除该podUID相关的资源，包括关闭update channel，退出podWorkerLoop循环，删除缓存，并暴露metrics
	case !canEverStart:
		p.cleanupUnstartedPod(update.Options.Pod, status)
		status.working = false
		if start := update.Options.StartTime; !start.IsZero() {
			metrics.PodWorkerDuration.WithLabelValues("terminated").Observe(metrics.SinceInSeconds(start))
		}
		klog.V(4).InfoS("Pod cannot start ever", "pod", klog.KObj(update.Options.Pod), "podUID", podUID, "updateType", update.WorkType)
		return ctx, update, canStart, canEverStart, true
    // 如果当前不能启动，将配置变更重新放入pendingUpdate
	case !canStart:
		// this is the only path we don't start the pod, so we need to put the change back in pendingUpdate
		status.pendingUpdate = &update.Options
		status.working = false
		klog.V(4).InfoS("Pod cannot start yet", "pod", klog.KObj(update.Options.Pod), "podUID", podUID)
		return ctx, update, canStart, canEverStart, true
	}

    // 添加启动时间，表明该pod已经启动
	// mark the pod as started
	status.startedAt = p.clock.Now()
    // 合并状态
	status.mergeLastUpdate(update.Options)

	// If we are admitting the pod and it is new, record the count of containers
	// TODO: We should probably move this into syncPod and add an execution count
	// to the syncPod arguments, and this should be recorded on the first sync.
	// Leaving it here complicates a particularly important loop.
	metrics.ContainersPerPodCount.Observe(float64(len(update.Options.Pod.Spec.Containers)))

	return ctx, update, true, true, true
}
```
#### completeSync

#### completeTerminating

#### completeTerminatingRuntimePod

#### completeTerminated

#### podWorkerLoop
看kubelet.md

#### completeWork
看channel.md

