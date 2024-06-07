### Status_Manager
statusManager从podWorker中接收pod状态更新事件和这些pod匹配的API状态的更新。
statusManager是podWorker和需要检查pod是否仍处于运行状态的其他组件（这些组件访问statusManger来获取pod运行状态而不是直接去访问podWorker）的下游节点。
- 状态缓存管理：statusManager维护一个本地缓存，用于存储集群中各种资源的状态信息，如pod的运行状态、Node的健康状态等。定期从apiserver中获取最新的状态，并将其更新到缓存中，提高性能和响应速度。
- 状态更新和同步：statusManager负责从apiserver中获取最新的状态信息，并更新到本地缓存。通过监听apiserver的事件和定期轮询两种方式来获取更新，一旦有新的更新事件，statusManager就更新本地缓存，确保缓存中的状态信息与apiserver中的一致。
- 状态处理和计算：statusManager对获取的状态信息进行处理和计算，以生成更高级的状态信息或指标。例如，根据pod的容器状态计算出整个pod的运行状态，根据Node的资源使用情况计算出节点的负载情况等。
- 状态监控和报警：statusManager监控集群中各个资源的状态，并根据定义的规则和策略触发告警。它可以检测到异常的状态变化或运行问题，并发送警报通知管理员或自动触发一些处理操作。
- 状态查询和提供接口：statusManager提供查询接口，供其他组件或工具获取状态信息。通过这些接口，其他组件可以获取最新的状态信息，从而进一步处理、分析或者展示。
statusManager的主要功能是管理和维护集群中各个资源的状态信息，确保状态的准确性、一致性，并提供对外的接口和功能，以支持状态的查询、处理、监控和报警等操作。
同时，statusManager中的podStatuses也管理了当前缓存的运行状态，这个状态要同步到apiserver中来保证数据的一致性。
#### 定义
```go
// 保证kubelet中pod状态的准确性，并且同步API server保持数据是最新的
// Manager is the Source of truth for kubelet pod status, and should be kept up-to-date with
// the latest v1.PodStatus. It also syncs updates back to the API server.
type Manager interface {
	PodStatusProvider

    // 后台开启statusManager的同步，分为定时同步和事件触发同步
	// Start the API server status sync loop.
	Start()

    // 更新本地缓存，同时触发事件，通过apiserver更新etcd中对应的pod状态
	// SetPodStatus caches updates the cached status for the given pod, and triggers a status update.
	SetPodStatus(pod *v1.Pod, status v1.PodStatus)

    // 同上
	// SetContainerReadiness updates the cached container status with the given readiness, and
	// triggers a status update.
	SetContainerReadiness(podUID types.UID, containerID kubecontainer.ContainerID, ready bool)
    
    // 同上
	// SetContainerStartup updates the cached container status with the given startup, and
	// triggers a status update.
	SetContainerStartup(podUID types.UID, containerID kubecontainer.ContainerID, started bool)

    // 同上
	// TerminatePod resets the container status for the provided pod to terminated and triggers
	// a status update.
	TerminatePod(pod *v1.Pod)

	// RemoveOrphanedStatuses scans the status cache and removes any entries for pods not included in
	// the provided podUIDs.
	RemoveOrphanedStatuses(podUIDs map[types.UID]bool)

	// GetContainerResourceAllocation returns checkpointed AllocatedResources value for the container
	GetContainerResourceAllocation(podUID string, containerName string) (v1.ResourceList, bool)

	// GetPodResizeStatus returns checkpointed PodStatus.Resize value
	GetPodResizeStatus(podUID string) (v1.PodResizeStatus, bool)

	// SetPodAllocation checkpoints the resources allocated to a pod's containers.
	SetPodAllocation(pod *v1.Pod) error

	// SetPodResizeStatus checkpoints the last resizing decision for the pod.
	SetPodResizeStatus(podUID types.UID, resize v1.PodResizeStatus) error
}

// Updates pod statuses in apiserver. Writes only when new status has changed.
// All methods are thread-safe.
type manager struct {
    // apiserver 客户端
	kubeClient clientset.Interface
	podManager PodManager
    // podUID到pod运行状态的缓存信息
	// Map from pod UID to sync status of the corresponding pod.
	podStatuses      map[types.UID]versionedPodStatus
	podStatusesLock  sync.RWMutex
    // 监听事件的channel，当里面能读到东西就需要完成一次同步
	podStatusChannel chan struct{}
    // podUID到成功发送到apiserver的最新状态版本的映射
	// Map from (mirror) pod UID to latest status version successfully sent to the API server.
	// apiStatusVersions must only be accessed from the sync thread.
	apiStatusVersions map[kubetypes.MirrorPodUID]uint64
	podDeletionSafety PodDeletionSafetyProvider

	podStartupLatencyHelper PodStartupLatencyStateHelper
    // 允许保存/恢复pod资源分配，并容忍kubelet重新启动
	// state allows to save/restore pod resource allocation and tolerate kubelet restarts.
	state state.State
	// stateFileDirectory holds the directory where the state file for checkpoints is held.
    // 存放检查点状态文件的目录
	stateFileDirectory string
}
```

#### 初始化
```go
    // pkg/kubelet/kubelet.go:1655
	// Start component sync loops.
	kl.statusManager.Start()

// pkg/kubelet/status/status_manager.go:194
func (m *manager) Start() {
    // 初始化
	// Initialize m.state to no-op state checkpoint manager
	m.state = state.NewNoopStateCheckpoint()

	// Create pod allocation checkpoint manager even if client is nil so as to allow local get/set of AllocatedResources & Resize
	if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
		stateImpl, err := state.NewStateCheckpoint(m.stateFileDirectory, podStatusManagerStateFile)
		if err != nil {
			// This is a crictical, non-recoverable failure.
			klog.ErrorS(err, "Could not initialize pod allocation checkpoint manager, please drain node and remove policy state file")
			panic(err)
		}
		m.state = stateImpl
	}

	// Don't start the status manager if we don't have a client. This will happen
	// on the master, where the kubelet is responsible for bootstrapping the pods
	// of the master components.
	if m.kubeClient == nil {
		klog.InfoS("Kubernetes client is nil, not starting status manager")
		return
	}

	klog.InfoS("Starting to sync pod status with apiserver")

	//nolint:staticcheck // SA1015 Ticker can leak since this is only called once and doesn't handle termination.
	syncTicker := time.NewTicker(syncPeriod).C

	// syncPod and syncBatch share the same go routine to avoid sync races.
	go wait.Forever(func() {
		for {
			select {
                // 如果podStatusChannel中有数据，则同步单个pod
			case <-m.podStatusChannel:
				klog.V(4).InfoS("Syncing updated statuses")
				m.syncBatch(false)
                // 每10秒全量同步一次
			case <-syncTicker:
				klog.V(4).InfoS("Syncing all statuses")
				m.syncBatch(true)
			}
		}
	}, 0)
}

```

```go
// pkg/kubelet/status/status_manager.go:766
// 从apiserver中同步pod状态
// syncBatch syncs pods statuses with the apiserver. Returns the number of syncs
// attempted for testing.
func (m *manager) syncBatch(all bool) int {
	type podSync struct {
		podUID    types.UID
		statusUID kubetypes.MirrorPodUID
		status    versionedPodStatus
	}

	var updatedStatuses []podSync
    // 从podManager缓存中获取pod到mirrorPod和mirrorPod到pod的映射
	podToMirror, mirrorToPod := m.podManager.GetUIDTranslations()
	func() { // Critical section
		m.podStatusesLock.RLock()
		defer m.podStatusesLock.RUnlock()

        // 全量更新
		// Clean up orphaned versions.
		if all {
			for uid := range m.apiStatusVersions {
				_, hasPod := m.podStatuses[types.UID(uid)]
				_, hasMirror := mirrorToPod[uid]
				if !hasPod && !hasMirror {
                    // 获取api最新版本的pod，并查看缓存中是否存在，如果没有，直接删掉
					delete(m.apiStatusVersions, uid)
				}
			}
		}

		// Decide which pods need status updates.
		for uid, status := range m.podStatuses {
            // 静态pod在源码中通过pod UID识别，但是在api中通过镜像pod的UID识别，需要转换成pod UID
			// translate the pod UID (source) to the status UID (API pod) -
			// static pods are identified in source by pod UID but tracked in the
			// API via the uid of the mirror pod
			uidOfStatus := kubetypes.MirrorPodUID(uid)
			if mirrorUID, ok := podToMirror[kubetypes.ResolvedPodUID(uid)]; ok {
				if mirrorUID == "" {
					klog.V(5).InfoS("Static pod does not have a corresponding mirror pod; skipping",
						"podUID", uid,
						"pod", klog.KRef(status.podNamespace, status.podName))
					continue
				}
				uidOfStatus = mirrorUID
			}

			// if a new status update has been delivered, trigger an update, otherwise the
			// pod can wait for the next bulk check (which performs reconciliation as well)
			if !all {
                // 如果不是全量更新，验证一下apiserver中pod的版本号和缓存的版本号，如果大于则跳过，小于则更新
				if m.apiStatusVersions[uidOfStatus] >= status.version {
					continue
				}
				updatedStatuses = append(updatedStatuses, podSync{uid, uidOfStatus, status})
				continue
			}

            // 如果是全量更新，验证当前这个pod是否需要更新
			// Ensure that any new status, or mismatched status, or pod that is ready for
			// deletion gets updated. If a status update fails we retry the next time any
			// other pod is updated.
			if m.needsUpdate(types.UID(uidOfStatus), status) {
				updatedStatuses = append(updatedStatuses, podSync{uid, uidOfStatus, status})
                // 是否需要重新协调
			} else if m.needsReconcile(uid, status.status) {
                // 删掉当前pod的api版本号，之后会由syncPod重新写进来
				// Delete the apiStatusVersions here to force an update on the pod status
				// In most cases the deleted apiStatusVersions here should be filled
				// soon after the following syncPod() [If the syncPod() sync an update
				// successfully].
				delete(m.apiStatusVersions, uidOfStatus)
				updatedStatuses = append(updatedStatuses, podSync{uid, uidOfStatus, status})
			}
		}
	}()

	for _, update := range updatedStatuses {
		klog.V(5).InfoS("Sync pod status", "podUID", update.podUID, "statusUID", update.statusUID, "version", update.status.version)
		m.syncPod(update.podUID, update.status)
	}

	return len(updatedStatuses)
}
```

```go
// needsUpdate returns whether the status is stale for the given pod UID.
// This method is not thread safe, and must only be accessed by the sync thread.
func (m *manager) needsUpdate(uid types.UID, status versionedPodStatus) bool {
    // 判断给定的podID是否在apiserver中有版本号，如果没有或者apiserver中的版本小于当前版本，则返回true，说明需要更新
	latest, ok := m.apiStatusVersions[kubetypes.MirrorPodUID(uid)]
	if !ok || latest < status.version {
		return true
	}
    // 如果podManager缓存中不存在该podUID，则返回false，说明不需要更新
	pod, ok := m.podManager.GetPodByUID(uid)
	if !ok {
		return false
	}
    // 验证pod是否可以删除
	return m.canBeDeleted(pod, status.status, status.podIsFinished)
}

func (m *manager) canBeDeleted(pod *v1.Pod, status v1.PodStatus, podIsFinished bool) bool {
	if pod.DeletionTimestamp == nil || kubetypes.IsMirrorPod(pod) {
		return false
	}
	// Delay deletion of pods until the phase is terminal, based on pod.Status
	// which comes from pod manager.
	if !podutil.IsPodPhaseTerminal(pod.Status.Phase) {
		// For debugging purposes we also log the kubelet's local phase, when the deletion is delayed.
		klog.V(3).InfoS("Delaying pod deletion as the phase is non-terminal", "phase", pod.Status.Phase, "localPhase", status.Phase, "pod", klog.KObj(pod), "podUID", pod.UID)
		return false
	}
	// If this is an update completing pod termination then we know the pod termination is finished.
	if podIsFinished {
		klog.V(3).InfoS("The pod termination is finished as SyncTerminatedPod completes its execution", "phase", pod.Status.Phase, "localPhase", status.Phase, "pod", klog.KObj(pod), "podUID", pod.UID)
		return true
	}
	return false
}

func (m *manager) needsReconcile(uid types.UID, status v1.PodStatus) bool {
	// The pod could be a static pod, so we should translate first.
	pod, ok := m.podManager.GetPodByUID(uid)
	if !ok {
		klog.V(4).InfoS("Pod has been deleted, no need to reconcile", "podUID", string(uid))
		return false
	}
	// If the pod is a static pod, we should check its mirror pod, because only status in mirror pod is meaningful to us.
	if kubetypes.IsStaticPod(pod) {
		mirrorPod, ok := m.podManager.GetMirrorPodByPod(pod)
		if !ok {
			klog.V(4).InfoS("Static pod has no corresponding mirror pod, no need to reconcile", "pod", klog.KObj(pod))
			return false
		}
		pod = mirrorPod
	}

	podStatus := pod.Status.DeepCopy()
	normalizeStatus(pod, podStatus)

	if isPodStatusByKubeletEqual(podStatus, &status) {
		// If the status from the source is the same with the cached status,
		// reconcile is not needed. Just return.
		return false
	}
	klog.V(3).InfoS("Pod status is inconsistent with cached status for pod, a reconciliation should be triggered",
		"pod", klog.KObj(pod),
		"statusDiff", cmp.Diff(podStatus, &status))

	return true
}
```
#### syncPod
```go
// pkg/kubelet/status/status_manager.go:841
// syncPod syncs the given status with the API server. The caller must not hold the status lock.
func (m *manager) syncPod(uid types.UID, status versionedPodStatus) {
    // 从apiserver同步pod状态
	// TODO: make me easier to express from client code
	pod, err := m.kubeClient.CoreV1().Pods(status.podNamespace).Get(context.TODO(), status.podName, metav1.GetOptions{})
	if errors.IsNotFound(err) {
		klog.V(3).InfoS("Pod does not exist on the server",
			"podUID", uid,
			"pod", klog.KRef(status.podNamespace, status.podName))
		// If the Pod is deleted the status will be cleared in
		// RemoveOrphanedStatuses, so we just ignore the update here.
		return
	}
	if err != nil {
		klog.InfoS("Failed to get status for pod",
			"podUID", uid,
			"pod", klog.KRef(status.podNamespace, status.podName),
			"err", err)
		return
	}

	translatedUID := m.podManager.TranslatePodUID(pod.UID)
    // 如果查询到的uid和pod的uid不同，代表pod被删除后重建了，删除pod状态然后跳过更新
	// Type convert original uid just for the purpose of comparison.
	if len(translatedUID) > 0 && translatedUID != kubetypes.ResolvedPodUID(uid) {
		klog.V(2).InfoS("Pod was deleted and then recreated, skipping status update",
			"pod", klog.KObj(pod),
			"oldPodUID", uid,
			"podUID", translatedUID)
		m.deletePodStatus(uid)
		return
	}

    // 将api-server的pod信息和statusManager的pod信息比较然后合并成最新的（api-server的状态就是statusManager的状态，要确保api-server的状态保持与statusManager状态同步）
	mergedStatus := mergePodStatus(pod.Status, status.status, m.podDeletionSafety.PodCouldHaveRunningContainers(pod))
    // 通过patch方法将status更新到apiserver
	newPod, patchBytes, unchanged, err := statusutil.PatchPodStatus(context.TODO(), m.kubeClient, pod.Namespace, pod.Name, pod.UID, pod.Status, mergedStatus)
	klog.V(3).InfoS("Patch status for pod", "pod", klog.KObj(pod), "podUID", uid, "patch", string(patchBytes))

	if err != nil {
		klog.InfoS("Failed to update status for pod", "pod", klog.KObj(pod), "err", err)
		return
	}
	if unchanged {
		klog.V(3).InfoS("Status for pod is up-to-date", "pod", klog.KObj(pod), "statusVersion", status.version)
	} else {
		klog.V(3).InfoS("Status for pod updated successfully", "pod", klog.KObj(pod), "statusVersion", status.version, "status", mergedStatus)
		pod = newPod
		// We pass a new object (result of API call which contains updated ResourceVersion)
		m.podStartupLatencyHelper.RecordStatusUpdated(pod)
	}

	// measure how long the status update took to propagate from generation to update on the server
	if status.at.IsZero() {
		klog.V(3).InfoS("Pod had no status time set", "pod", klog.KObj(pod), "podUID", uid, "version", status.version)
	} else {
		duration := time.Since(status.at).Truncate(time.Millisecond)
		metrics.PodStatusSyncDuration.Observe(duration.Seconds())
	}
    // 更新版本号
	m.apiStatusVersions[kubetypes.MirrorPodUID(pod.UID)] = status.version

    // 验证是否删除
	// We don't handle graceful deletion of mirror pods.
	if m.canBeDeleted(pod, status.status, status.podIsFinished) {
		deleteOptions := metav1.DeleteOptions{
			GracePeriodSeconds: new(int64),
			// Use the pod UID as the precondition for deletion to prevent deleting a
			// newly created pod with the same name and namespace.
			Preconditions: metav1.NewUIDPreconditions(string(pod.UID)),
		}
		err = m.kubeClient.CoreV1().Pods(pod.Namespace).Delete(context.TODO(), pod.Name, deleteOptions)
		if err != nil {
			klog.InfoS("Failed to delete status for pod", "pod", klog.KObj(pod), "err", err)
			return
		}
		klog.V(3).InfoS("Pod fully terminated and removed from etcd", "pod", klog.KObj(pod))
		m.deletePodStatus(uid)
	}
}
```
#### SetPodStatus
```go
// 将status更新到pod中
func (m *manager) SetPodStatus(pod *v1.Pod, status v1.PodStatus) {
	m.podStatusesLock.Lock()
	defer m.podStatusesLock.Unlock()

    // 深拷贝
	// Make sure we're caching a deep copy.
	status = *status.DeepCopy()

	// Force a status update if deletion timestamp is set. This is necessary
	// because if the pod is in the non-running state, the pod worker still
	// needs to be able to trigger an update and/or deletion.
	m.updateStatusInternal(pod, status, pod.DeletionTimestamp != nil, false)
}

// updateStatusInternal updates the internal status cache, and queues an update to the api server if
// necessary.
// This method IS NOT THREAD SAFE and must be called from a locked function.
func (m *manager) updateStatusInternal(pod *v1.Pod, status v1.PodStatus, forceUpdate, podIsFinished bool) {
	var oldStatus v1.PodStatus
	cachedStatus, isCached := m.podStatuses[pod.UID]
	if isCached {
		oldStatus = cachedStatus.status
		// TODO(#116484): Also assign terminal phase to static pods.
		if !kubetypes.IsStaticPod(pod) {
			if cachedStatus.podIsFinished && !podIsFinished {
				klog.InfoS("Got unexpected podIsFinished=false, while podIsFinished=true in status cache, programmer error.", "pod", klog.KObj(pod))
				podIsFinished = true
			}
		}
	} else if mirrorPod, ok := m.podManager.GetMirrorPodByPod(pod); ok {
		oldStatus = mirrorPod.Status
	} else {
		oldStatus = pod.Status
	}

    // 校验非法的异常情况
	// Check for illegal state transition in containers
	if err := checkContainerStateTransition(&oldStatus, &status, &pod.Spec); err != nil {
		klog.ErrorS(err, "Status update on pod aborted", "pod", klog.KObj(pod))
		return
	}
    // 跟新pod中的LastTransitionTime
	// Set ContainersReadyCondition.LastTransitionTime.
	updateLastTransitionTime(&status, &oldStatus, v1.ContainersReady)

	// Set ReadyCondition.LastTransitionTime.
	updateLastTransitionTime(&status, &oldStatus, v1.PodReady)

	// Set InitializedCondition.LastTransitionTime.
	updateLastTransitionTime(&status, &oldStatus, v1.PodInitialized)

	// Set PodReadyToStartContainersCondition.LastTransitionTime.
	updateLastTransitionTime(&status, &oldStatus, v1.PodReadyToStartContainers)

	// Set PodScheduledCondition.LastTransitionTime.
	updateLastTransitionTime(&status, &oldStatus, v1.PodScheduled)

	if utilfeature.DefaultFeatureGate.Enabled(features.PodDisruptionConditions) {
		// Set DisruptionTarget.LastTransitionTime.
		updateLastTransitionTime(&status, &oldStatus, v1.DisruptionTarget)
	}

	// ensure that the start time does not change across updates.
	if oldStatus.StartTime != nil && !oldStatus.StartTime.IsZero() {
		status.StartTime = oldStatus.StartTime
	} else if status.StartTime.IsZero() {
		// if the status has no start time, we need to set an initial time
		now := metav1.Now()
		status.StartTime = &now
	}

	normalizeStatus(pod, &status)

	// Perform some more extensive logging of container termination state to assist in
	// debugging production races (generally not needed).
	if klogV := klog.V(5); klogV.Enabled() {
		var containers []string
		for _, s := range append(append([]v1.ContainerStatus(nil), status.InitContainerStatuses...), status.ContainerStatuses...) {
			var current, previous string
			switch {
			case s.State.Running != nil:
				current = "running"
			case s.State.Waiting != nil:
				current = "waiting"
			case s.State.Terminated != nil:
				current = fmt.Sprintf("terminated=%d", s.State.Terminated.ExitCode)
			default:
				current = "unknown"
			}
			switch {
			case s.LastTerminationState.Running != nil:
				previous = "running"
			case s.LastTerminationState.Waiting != nil:
				previous = "waiting"
			case s.LastTerminationState.Terminated != nil:
				previous = fmt.Sprintf("terminated=%d", s.LastTerminationState.Terminated.ExitCode)
			default:
				previous = "<none>"
			}
			containers = append(containers, fmt.Sprintf("(%s state=%s previous=%s)", s.Name, current, previous))
		}
		sort.Strings(containers)
		klogV.InfoS("updateStatusInternal", "version", cachedStatus.version+1, "podIsFinished", podIsFinished, "pod", klog.KObj(pod), "podUID", pod.UID, "containers", strings.Join(containers, " "))
	}

	// The intent here is to prevent concurrent updates to a pod's status from
	// clobbering each other so the phase of a pod progresses monotonically.
	if isCached && isPodStatusByKubeletEqual(&cachedStatus.status, &status) && !forceUpdate {
		klog.V(3).InfoS("Ignoring same status for pod", "pod", klog.KObj(pod), "status", status)
		return
	}

	newStatus := versionedPodStatus{
		status:        status,
		version:       cachedStatus.version + 1,
		podName:       pod.Name,
		podNamespace:  pod.Namespace,
		podIsFinished: podIsFinished,
	}

	// Multiple status updates can be generated before we update the API server,
	// so we track the time from the first status update until we retire it to
	// the API.
	if cachedStatus.at.IsZero() {
		newStatus.at = time.Now()
	} else {
		newStatus.at = cachedStatus.at
	}

    // 更新缓存
	m.podStatuses[pod.UID] = newStatus

	select {
        // 触发更新，触发上面的syncBatch
	case m.podStatusChannel <- struct{}{}:
	default:
		// there's already a status update pending
	}
}
```