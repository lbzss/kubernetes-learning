## 启动流程

cmd/kubelet/kubelet.go -> cmd/kubelet/app/server.go -> pkg/kubelet/kubelet.go
前两个文件主要是做前置校验，比如传参和服务器配置等，
```go
// cmd/kubelet/app/server.go:1269
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableServer bool) {
    // start the kubelet
    go k.Run(podCfg.Updates())
    // start the kubelet server
    if enableServer {
       go k.ListenAndServe(kubeCfg, kubeDeps.TLSOptions, kubeDeps.Auth, kubeDeps.TracerProvider)
    }
    if kubeCfg.ReadOnlyPort > 0 {
       go k.ListenAndServeReadOnly(netutils.ParseIPSloppy(kubeCfg.Address), uint(kubeCfg.ReadOnlyPort))
    }
    go k.ListenAndServePodResources()
}
```
```go
// pkg/kubelet/kubelet.go:235
type Bootstrap interface {
    GetConfiguration() kubeletconfiginternal.KubeletConfiguration
    BirthCry()
    StartGarbageCollection()
    ListenAndServe(kubeCfg *kubeletconfiginternal.KubeletConfiguration, tlsOptions *server.TLSOptions, auth server.AuthInterface, tp trace.TracerProvider)
    ListenAndServeReadOnly(address net.IP, port uint)
    ListenAndServePodResources()
    Run(<-chan kubetypes.PodUpdate)
    RunOnce(<-chan kubetypes.PodUpdate) ([]RunPodResult, error)
}
```
Bootstrap定义了Kubelet的接口方法。除ListenAndServe、ListenAndServeReadOnly、ListenAndServePodResources是kubelet做为服务端用的方法外，剩下的方法可以理解为后台执行的异步任务。
```go
// pkg/kubelet/kubelet.go:1577
// Run starts the kubelet reacting to config updates
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
    ctx := context.Background()
    if kl.logServer == nil {
       // 日志服务器相关
       ...
    }
    if kl.kubeClient == nil {
       klog.InfoS("No API server defined - no node status update will be sent")
    }
    // Start the cloud provider sync manager
    if kl.cloudResourceSyncManager != nil {
       go kl.cloudResourceSyncManager.Run(wait.NeverStop)
    }
    if err := kl.initializeModules(); err != nil {
       kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.KubeletSetupFailed, err.Error())
       klog.ErrorS(err, "Failed to initialize internal modules")
       os.Exit(1)
    }
    // Start volume manager
    go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)
    if kl.kubeClient != nil {
       // Start two go-routines to update the status.
       //
       // The first will report to the apiserver every nodeStatusUpdateFrequency and is aimed to provide regular status intervals,
       // while the second is used to provide a more timely status update during initialization and runs an one-shot update to the apiserver
       // once the node becomes ready, then exits afterwards.
       //
       // Introduce some small jittering to ensure that over time the requests won't start
       // accumulating at approximately the same time from the set of nodes due to priority and
       // fairness effect.
       go wait.JitterUntil(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, 0.04, true, wait.NeverStop)
       go kl.fastStatusUpdateOnce()
       // start syncing lease
       go kl.nodeLeaseController.Run(context.Background())
    }
    go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)
    // Set up iptables util rules
    if kl.makeIPTablesUtilChains {
       kl.initNetworkUtil()
    }
    // Start component sync loops.
    kl.statusManager.Start()
    // Start syncing RuntimeClasses if enabled.
    if kl.runtimeClassManager != nil {
       kl.runtimeClassManager.Start(wait.NeverStop)
    }
    // Start the pod lifecycle event generator.
    kl.pleg.Start()
    // Start eventedPLEG only if EventedPLEG feature gate is enabled.
    if utilfeature.DefaultFeatureGate.Enabled(features.EventedPLEG) {
       kl.eventedPleg.Start()
    }
    kl.syncLoop(ctx, updates, kl)
}
```
kubelet的初始化流程：cloudResourceSyncManager -> volumeManager -> nodeLeaseController -> NetworkUtil(iptables) -> statusManager -> runtimeClassManager -> pleg
最后进入syncLoop主循环。
#### 变更（例如 kubectl apply -f foo.yaml）
```go
// syncLoop是处理变更的主流程。监听来自file,apiserver,http三个channel并且将这三个channel中的变更合并。
// 如果发现有新的变更，则会针对期望状态和运行状态进行一次同步。
// 如果没有监听到变更，每隔一定时间同步最新的状态。永远不会返回。
func (kl *Kubelet) syncLoop(ctx context.Context, updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
    ...
    // 监听pleg中的eventChannel
    // pleg用于kublet监听容器运行时的状态并通知kublet容器的变化
    // pleg也会通知podWorkers来协调pod状态，例如容器挂了需要重启
    plegCh := kl.pleg.Watch()
    const (
       base   = 100 * time.Millisecond
       max    = 5 * time.Second
       factor = 2
    )
    duration := base
    ...
.    
    for {
       // 每100ms查一次容器运行时，如果失败，则等待2*n*100ms，最大不超过5s
       if err := kl.runtimeState.runtimeErrors(); err != nil {
          klog.ErrorS(err, "Skipping pod synchronization")
          // exponential backoff
          time.Sleep(duration)
          duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
          continue
       }
       // reset backoff if we have a success
       duration = base
       kl.syncLoopMonitor.Store(kl.clock.Now())
       // 主逻辑
       if !kl.syncLoopIteration(ctx, updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
          break
       }
       kl.syncLoopMonitor.Store(kl.clock.Now())
    }
}
```
```go
// 读取从不同channel监听到的事件->处理事件->更新时间戳，当同时监听到多个channel中的事件时，以随机顺序执行。
// 1. configCh: 将pod配置变更事件选择合适的handler进行处理
// 2. plegCh: 同步容器运行时缓存
// 3. syncCh：同步所有待更新的pod
// 4. housekeepingCh: 触发pod清理
// 5. livenessManager updateCh: 同步健康检查探针状态失败的pod
func (kl *Kubelet) syncLoopIteration(ctx context.Context, configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
    syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
    select {
    case u, open := <-configCh:
       // Update from a config source; dispatch it to the right handler
       // callback.
       if !open {
          klog.ErrorS(nil, "Update channel is closed, exiting the sync loop")
          return false
       }
       // 根据PodUpdate操作类型进行路由
       switch u.Op {
       case kubetypes.ADD:
          klog.V(2).InfoS("SyncLoop ADD", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
          // After restarting, kubelet will get all existing pods through
          // ADD as if they are new pods. These pods will then go through the
          // admission process and *may* be rejected. This can be resolved
          // once we have checkpointing.
          handler.HandlePodAdditions(u.Pods)
       case kubetypes.UPDATE:
          klog.V(2).InfoS("SyncLoop UPDATE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
          handler.HandlePodUpdates(u.Pods)
       case kubetypes.REMOVE:
          klog.V(2).InfoS("SyncLoop REMOVE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
          handler.HandlePodRemoves(u.Pods)
       case kubetypes.RECONCILE:
          klog.V(4).InfoS("SyncLoop RECONCILE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
          handler.HandlePodReconcile(u.Pods)
       case kubetypes.DELETE:
          klog.V(2).InfoS("SyncLoop DELETE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
          // DELETE is treated as a UPDATE because of graceful deletion.
          handler.HandlePodUpdates(u.Pods)
       case kubetypes.SET:
          // TODO: Do we want to support this?
          klog.ErrorS(nil, "Kubelet does not support snapshot update")
       default:
          klog.ErrorS(nil, "Invalid operation type received", "operation", u.Op)
       }
       kl.sourcesReady.AddSource(u.Source)
    case e := <-plegCh:
       if isSyncPodWorthy(e) {
          // PLEG event for a pod; sync it.
          if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
             klog.V(2).InfoS("SyncLoop (PLEG): event for pod", "pod", klog.KObj(pod), "event", e)
             handler.HandlePodSyncs([]*v1.Pod{pod})
          } else {
             // If the pod no longer exists, ignore the event.
             klog.V(4).InfoS("SyncLoop (PLEG): pod does not exist, ignore irrelevant event", "event", e)
          }
       }
       if e.Type == pleg.ContainerDied {
          if containerID, ok := e.Data.(string); ok {
             kl.cleanUpContainersInPod(e.ID, containerID)
          }
       }
    case <-syncCh:
       // Sync pods waiting for sync
       podsToSync := kl.getPodsToSync()
       if len(podsToSync) == 0 {
          break
       }
       klog.V(4).InfoS("SyncLoop (SYNC) pods", "total", len(podsToSync), "pods", klog.KObjSlice(podsToSync))
       handler.HandlePodSyncs(podsToSync)
    case update := <-kl.livenessManager.Updates():
       if update.Result == proberesults.Failure {
          handleProbeSync(kl, update, handler, "liveness", "unhealthy")
       }
    case update := <-kl.readinessManager.Updates():
       ready := update.Result == proberesults.Success
       kl.statusManager.SetContainerReadiness(update.PodUID, update.ContainerID, ready)
       status := ""
       if ready {
          status = "ready"
       }
       handleProbeSync(kl, update, handler, "readiness", status)
    case update := <-kl.startupManager.Updates():
       started := update.Result == proberesults.Success
       kl.statusManager.SetContainerStartup(update.PodUID, update.ContainerID, started)
       status := "unhealthy"
       if started {
          status = "started"
       }
       handleProbeSync(kl, update, handler, "startup", status)
    case <-housekeepingCh:
       if !kl.sourcesReady.AllReady() {
          // If the sources aren't ready or volume manager has not yet synced the states,
          // skip housekeeping, as we may accidentally delete pods from unready sources.
          klog.V(4).InfoS("SyncLoop (housekeeping, skipped): sources aren't ready yet")
       } else {
          start := time.Now()
          klog.V(4).InfoS("SyncLoop (housekeeping)")
          if err := handler.HandlePodCleanups(ctx); err != nil {
             klog.ErrorS(err, "Failed cleaning pods")
          }
          duration := time.Since(start)
          if duration > housekeepingWarningDuration {
             klog.ErrorS(fmt.Errorf("housekeeping took too long"), "Housekeeping took longer than expected", "expected", housekeepingWarningDuration, "actual", duration.Round(time.Millisecond))
          }
          klog.V(4).InfoS("SyncLoop (housekeeping) end", "duration", duration.Round(time.Millisecond))
       }
    }
    return true
}
```
### podAdd
```go
// pkg/kubelet/kubelet.go:2533
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
    ...
    for _, pod := range pods {
       existingPods := kl.podManager.GetPods()
       // podManager中存储了kubelet期望运行的pod集合和mirror pod，实际运行的pod存储在podWorkers中。仅仅是期望状态，假设需要强制删除一个pod，会直接从podManager中删除，但是它仍然存在于podWorks，因为还有收尾工作比如terminating需要完成。
       // 如果是正常删除，会先标记为删除状态，再正常删除。podManager中并不是所有的pod都是正在运行的，也不是所有正在运行的pod都在podManager中。
       // podManager中的数据来源是本地文件系统或http请求(例如静态pod)，apiserver（常规pod）
       // 其他组件可能会从podManager中查询期望状态的pod集合
       // 将需要新增的pod添加到podManager中
       kl.podManager.AddPod(pod)
       // 获取pod和mirrorPod（一些通过http和manifest定义的静态pod无法被apiserver管理，所以会创建mirrorPod用于管理这些静态pod，mirrorPod的注解中会有kubernetes.io/config.mirror）
       pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
       if wasMirror {
          if pod == nil {
             klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
             continue
          }
          kl.podWorkers.UpdatePod(UpdatePodOptions{
             Pod:        pod,
             MirrorPod:  mirrorPod,
             UpdateType: kubetypes.SyncPodUpdate,
             StartTime:  start,
          })
          continue
       }
       // Only go through the admission process if the pod is not requested
       // for termination by another part of the kubelet. If the pod is already
       // using resources (previously admitted), the pod worker is going to be
       // shutting it down. If the pod hasn't started yet, we know that when
       // the pod worker is invoked it will also avoid setting up the pod, so
       // we simply avoid doing any work.
       if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
          // We failed pods that we rejected, so activePods include all admitted
          // pods that are alive.
          activePods := kl.filterOutInactivePods(existingPods)
          if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
             // To handle kubelet restarts, test pod admissibility using AllocatedResources values
             // (for cpu & memory) from checkpoint store. If found, that is the source of truth.
             podCopy := pod.DeepCopy()
             kl.updateContainerResourceAllocation(podCopy)
             // Check if we can admit the pod; if not, reject it.
             if ok, reason, message := kl.canAdmitPod(activePods, podCopy); !ok {
                kl.rejectPod(pod, reason, message)
                continue
             }
             // For new pod, checkpoint the resource values at which the Pod has been admitted
             if err := kl.statusManager.SetPodAllocation(podCopy); err != nil {
                //TODO(vinaykul,InPlacePodVerticalScaling): Can we recover from this in some way? Investigate
                klog.ErrorS(err, "SetPodAllocation failed", "pod", klog.KObj(pod))
             }
          } else {
             // Check if we can admit the pod; if not, reject it.
             if ok, reason, message := kl.canAdmitPod(activePods, pod); !ok {
                kl.rejectPod(pod, reason, message)
                continue
             }
          }
       }
       kl.podWorkers.UpdatePod(UpdatePodOptions{
          Pod:        pod,
          MirrorPod:  mirrorPod,
          UpdateType: kubetypes.SyncPodCreate,
          StartTime:  start,
       })
    }
}
```
先更新podManager，再由podWorker进行实际的更新操作。
```go
// pkg/kubelet/pod/pod_manager.go:168
// 上面kl.podManager.AddPod(pod)的实际逻辑
// 确保mirrorPod和普通pod的关联关系
func (pm *basicManager) updatePodsInternal(pods ...*v1.Pod) {
    for _, pod := range pods {
       podFullName := kubecontainer.GetPodFullName(pod)
       // 如果是静态pod，使用mirrorPodUID更新
       if kubetypes.IsMirrorPod(pod) {
          mirrorPodUID := kubetypes.MirrorPodUID(pod.UID)
          pm.mirrorPodByUID[mirrorPodUID] = pod
          pm.mirrorPodByFullName[podFullName] = pod
          // 如果在podByFullName中存在同名pod，将mirrorPodUID与podUID绑定
          if p, ok := pm.podByFullName[podFullName]; ok {
             pm.translationByUID[mirrorPodUID] = kubetypes.ResolvedPodUID(p.UID)
          }
          // 如果不是静态pod，则直接使用解析后的podUID进行更新
       } else {
          resolvedPodUID := kubetypes.ResolvedPodUID(pod.UID)
          updateMetrics(pm.podByUID[resolvedPodUID], pod)
          pm.podByUID[resolvedPodUID] = pod
          pm.podByFullName[podFullName] = pod
          if mirror, ok := pm.mirrorPodByFullName[podFullName]; ok {
             pm.translationByUID[kubetypes.MirrorPodUID(mirror.UID)] = resolvedPodUID
          }
       }
    }
}
```

podWorders.UpdatePod 真正更新的部分，下面代码中忽略了不会走到的部分
```go
// pkg/kubelet/pod_workers.go:735
func (p *podWorkers) UpdatePod(options UpdatePodOptions) {
    // Handle when the pod is an orphan (no config) and we only have runtime status by running only
    // the terminating part of the lifecycle. A running pod contains only a minimal set of information
    // about the pod
    var isRuntimePod bool
    var uid types.UID
    var name, ns string
    ...
    // 用于处理runningPod，这段代码中不走这个逻辑，暂时忽略
    if {
    ...
    } else {
    uid, ns, name = options.Pod.UID, options.Pod.Namespace, options.Pod.Name
    }
    p.podLock.Lock()
    defer p.podLock.Unlock()
    // decide what to do with this pod - we are either setting it up, tearing it down, or ignoring it
    var firstTime bool
    now := p.clock.Now()
    // 检查podUID关联的syncStatus是否存在，因为是第一次创建，肯定不存在
    status, ok := p.podSyncStatuses[uid]
    if !ok {
       klog.V(4).InfoS("Pod is being synced for the first time", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
       firstTime = true
       status = &podSyncStatus{
          syncedAt: now,
          fullname: kubecontainer.BuildPodFullName(name, ns),
       }
       ...
       p.podSyncStatuses[uid] = status
    }
    pod := options.Pod
    ...
    // start the pod worker goroutine if it doesn't exist
    podUpdates, exists := p.podUpdates[uid]
    if !exists {
       // buffer the channel to avoid blocking this method
       podUpdates = make(chan struct{}, 1)
       p.podUpdates[uid] = podUpdates
       // ensure that static pods start in the order they are received by UpdatePod
       if kubetypes.IsStaticPod(pod) {
          p.waitingToStartStaticPodsByFullname[status.fullname] =
             append(p.waitingToStartStaticPodsByFullname[status.fullname], uid)
       }
       // allow testing of delays in the pod update channel
       var outCh <-chan struct{}
       if p.workerChannelFn != nil {
          outCh = p.workerChannelFn(uid, podUpdates)
       } else {
          outCh = podUpdates
       }
       // spawn a pod worker
       go func() {
          // TODO: this should be a wait.Until with backoff to handle panics, and
          // accept a context for shutdown
          defer runtime.HandleCrash()
          defer klog.V(3).InfoS("Pod worker has stopped", "podUID", uid)
          p.podWorkerLoop(uid, outCh)
       }()
    }
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
    case podUpdates <- struct{}{}:
    default:
    }
    if (becameTerminating || wasGracePeriodShortened) && status.cancelFn != nil {
       klog.V(3).InfoS("Cancelling current pod sync", "pod", klog.KRef(ns, name), "podUID", uid, "workType", status.WorkType())
       status.cancelFn()
       return
    }
}
```
```go
// pkg/kubelet/pod_workers.go:1211
func (p *podWorkers) podWorkerLoop(podUID types.UID, podUpdates <-chan struct{}) {
    var lastSyncTime time.Time
    for range podUpdates {
       ctx, update, canStart, canEverStart, ok := p.startPodSync(podUID)
       // If we had no update waiting, it means someone initialized the channel without filling out pendingUpdate.
       if !ok {
          continue
       }
       // If the pod was terminated prior to the pod being allowed to start, we exit the loop.
       if !canEverStart {
          return
       }
       // If the pod is not yet ready to start, continue and wait for more updates.
       if !canStart {
          continue
       }
       podUID, podRef := podUIDAndRefForUpdate(update.Options)
       klog.V(4).InfoS("Processing pod event", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
       var isTerminal bool
       err := func() error {
          // The worker is responsible for ensuring the sync method sees the appropriate
          // status updates on resyncs (the result of the last sync), transitions to
          // terminating (no wait), or on terminated (whatever the most recent state is).
          // Only syncing and terminating can generate pod status changes, while terminated
          // pods ensure the most recent status makes it to the api server.
          var status *kubecontainer.PodStatus
          var err error
          switch {
          case update.Options.RunningPod != nil:
             // when we receive a running pod, we don't need status at all because we are
             // guaranteed to be terminating and we skip updates to the pod
          default:
             // wait until we see the next refresh from the PLEG via the cache (max 2s)
             // TODO: this adds ~1s of latency on all transitions from sync to terminating
             //  to terminated, and on all termination retries (including evictions). We should
             //  improve latency by making the pleg continuous and by allowing pod status
             //  changes to be refreshed when key events happen (killPod, sync->terminating).
             //  Improving this latency also reduces the possibility that a terminated
             //  container's status is garbage collected before we have a chance to update the
             //  API server (thus losing the exit code).
             status, err = p.podCache.GetNewerThan(update.Options.Pod.UID, lastSyncTime)
             if err != nil {
                // This is the legacy event thrown by manage pod loop all other events are now dispatched
                // from syncPodFn
                p.recorder.Eventf(update.Options.Pod, v1.EventTypeWarning, events.FailedSync, "error determining status: %v", err)
                return err
             }
          }
          // Take the appropriate action (illegal phases are prevented by UpdatePod)
          switch {
          case update.WorkType == TerminatedPod:
             err = p.podSyncer.SyncTerminatedPod(ctx, update.Options.Pod, status)
          case update.WorkType == TerminatingPod:
             var gracePeriod *int64
             if opt := update.Options.KillPodOptions; opt != nil {
                gracePeriod = opt.PodTerminationGracePeriodSecondsOverride
             }
             podStatusFn := p.acknowledgeTerminating(podUID)
             // if we only have a running pod, terminate it directly
             if update.Options.RunningPod != nil {
                err = p.podSyncer.SyncTerminatingRuntimePod(ctx, update.Options.RunningPod)
             } else {
                err = p.podSyncer.SyncTerminatingPod(ctx, update.Options.Pod, status, gracePeriod, podStatusFn)
             }
          default:
             // 初次创建的pod会走到这里
             isTerminal, err = p.podSyncer.SyncPod(ctx, update.Options.UpdateType, update.Options.Pod, update.Options.MirrorPod, status)
          }
          lastSyncTime = p.clock.Now()
          return err
       }()
       ...
       // queue a retry if necessary, then put the next event in the channel if any
       p.completeWork(podUID, phaseTransition, err)
       if start := update.Options.StartTime; !start.IsZero() {
          metrics.PodWorkerDuration.WithLabelValues(update.Options.UpdateType.String()).Observe(metrics.SinceInSeconds(start))
       }
       klog.V(4).InfoS("Processing pod event done", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
    }
}
```
```go
// pkg/kubelet/kubelet.go:1723
// 1. 如果正在创建pod，记录启动时间
// 2. 调用generateAPIPodStatus生成pod的PodStatus
// 3. 如果第一次发现pod正在运行，记录时间
// 4. 在stausManager中记录pod的状态
// 5. 运行条件校验，如果不满足条件，停止pod中的容器
// 6. ..
// 7. 如果是静态pod并且没有mirrorPod则创建对应的mirrorPod
// 8. 创建数据目录
// 9. 挂载卷
// 10. 获取image pull Secret
// 11. 调用容器运行时的SyncPod创建容器
// 12. 更新pod的流量入口和出口策略
func (kl *Kubelet) SyncPod(ctx context.Context, updateType kubetypes.SyncPodType, pod, mirrorPod *v1.Pod, podStatus *kubecontainer.PodStatus) (isTerminal bool, err error) {
    ...
    // Generate final API pod status with pod and status manager status
    apiPodStatus := kl.generateAPIPodStatus(pod, podStatus, false)
    ...
    // 运行条件校验，如果不满足条件，停止pod中的容器
    // 如果pod不应该被运行，停止pod中的容器。跟termination不同（因为条件不满足所以停止，而不是异常退出或者手动停止，后续条件满足有可能会重启）
    runnable := kl.canRunPod(pod)
    if !runnable.Admit {
       // Pod is not runnable; and update the Pod and Container statuses to why.
       if apiPodStatus.Phase != v1.PodFailed && apiPodStatus.Phase != v1.PodSucceeded {
          apiPodStatus.Phase = v1.PodPending
       }
       apiPodStatus.Reason = runnable.Reason
       apiPodStatus.Message = runnable.Message
       // Waiting containers are not creating.
       const waitingReason = "Blocked"
       for _, cs := range apiPodStatus.InitContainerStatuses {
          if cs.State.Waiting != nil {
             cs.State.Waiting.Reason = waitingReason
          }
       }
       for _, cs := range apiPodStatus.ContainerStatuses {
          if cs.State.Waiting != nil {
             cs.State.Waiting.Reason = waitingReason
          }
       }
    }
    // Record the time it takes for the pod to become running
    // since kubelet first saw the pod if firstSeenTime is set.
    existingStatus, ok := kl.statusManager.GetPodStatus(pod.UID)
    if !ok || existingStatus.Phase == v1.PodPending && apiPodStatus.Phase == v1.PodRunning &&
       !firstSeenTime.IsZero() {
       metrics.PodStartDuration.Observe(metrics.SinceInSeconds(firstSeenTime))
    }
    kl.statusManager.SetPodStatus(pod, apiPodStatus)
    // Pods that are not runnable must be stopped - return a typed error to the pod worker
    if !runnable.Admit {
       klog.V(2).InfoS("Pod is not runnable and must have running containers stopped", "pod", klog.KObj(pod), "podUID", pod.UID, "message", runnable.Message)
       var syncErr error
       p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
       if err := kl.killPod(ctx, pod, p, nil); err != nil {
          if !wait.Interrupted(err) {
             kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToKillPod, "error killing pod: %v", err)
             syncErr = fmt.Errorf("error killing pod: %w", err)
             utilruntime.HandleError(syncErr)
          }
       } else {
          // There was no error killing the pod, but the pod cannot be run.
          // Return an error to signal that the sync loop should back off.
          syncErr = fmt.Errorf("pod cannot be run: %v", runnable.Message)
       }
       return false, syncErr
    }
    // If the network plugin is not ready, only start the pod if it uses the host network
    if err := kl.runtimeState.networkErrors(); err != nil && !kubecontainer.IsHostNetworkPod(pod) {
       kl.recorder.Eventf(pod, v1.EventTypeWarning, events.NetworkNotReady, "%s: %v", NetworkNotReadyErrorMsg, err)
       return false, fmt.Errorf("%s: %v", NetworkNotReadyErrorMsg, err)
    }
    // ensure the kubelet knows about referenced secrets or configmaps used by the pod
    if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
       if kl.secretManager != nil {
          kl.secretManager.RegisterPod(pod)
       }
       if kl.configMapManager != nil {
          kl.configMapManager.RegisterPod(pod)
       }
    }
    // Create Cgroups for the pod and apply resource parameters
    // to them if cgroups-per-qos flag is enabled.
    pcm := kl.containerManager.NewPodContainerManager()
    // If pod has already been terminated then we need not create
    // or update the pod's cgroup
    // TODO: once context cancellation is added this check can be removed
    if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
       // When the kubelet is restarted with the cgroups-per-qos
       // flag enabled, all the pod's running containers
       // should be killed intermittently and brought back up
       // under the qos cgroup hierarchy.
       // Check if this is the pod's first sync
       firstSync := true
       for _, containerStatus := range apiPodStatus.ContainerStatuses {
          if containerStatus.State.Running != nil {
             firstSync = false
             break
          }
       }
       // Don't kill containers in pod if pod's cgroups already
       // exists or the pod is running for the first time
       podKilled := false
       if !pcm.Exists(pod) && !firstSync {
          p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
          if err := kl.killPod(ctx, pod, p, nil); err == nil {
             if wait.Interrupted(err) {
                return false, err
             }
             podKilled = true
          } else {
             klog.ErrorS(err, "KillPod failed", "pod", klog.KObj(pod), "podStatus", podStatus)
          }
       }
       // Create and Update pod's Cgroups
       // Don't create cgroups for run once pod if it was killed above
       // The current policy is not to restart the run once pods when
       // the kubelet is restarted with the new flag as run once pods are
       // expected to run only once and if the kubelet is restarted then
       // they are not expected to run again.
       // We don't create and apply updates to cgroup if its a run once pod and was killed above
       if !(podKilled && pod.Spec.RestartPolicy == v1.RestartPolicyNever) {
          if !pcm.Exists(pod) {
             if err := kl.containerManager.UpdateQOSCgroups(); err != nil {
                klog.V(2).InfoS("Failed to update QoS cgroups while syncing pod", "pod", klog.KObj(pod), "err", err)
             }
             if err := pcm.EnsureExists(pod); err != nil {
                kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToCreatePodContainer, "unable to ensure pod container exists: %v", err)
                return false, fmt.Errorf("failed to ensure that the pod: %v cgroups exist and are correctly applied: %v", pod.UID, err)
             }
          }
       }
    }
    // Create Mirror Pod for Static Pod if it doesn't already exist
    if kubetypes.IsStaticPod(pod) {
       deleted := false
       if mirrorPod != nil {
          if mirrorPod.DeletionTimestamp != nil || !kubepod.IsMirrorPodOf(mirrorPod, pod) {
             // The mirror pod is semantically different from the static pod. Remove
             // it. The mirror pod will get recreated later.
             klog.InfoS("Trying to delete pod", "pod", klog.KObj(pod), "podUID", mirrorPod.ObjectMeta.UID)
             podFullName := kubecontainer.GetPodFullName(pod)
             var err error
             deleted, err = kl.mirrorPodClient.DeleteMirrorPod(podFullName, &mirrorPod.ObjectMeta.UID)
             if deleted {
                klog.InfoS("Deleted mirror pod because it is outdated", "pod", klog.KObj(mirrorPod))
             } else if err != nil {
                klog.ErrorS(err, "Failed deleting mirror pod", "pod", klog.KObj(mirrorPod))
             }
          }
       }
       if mirrorPod == nil || deleted {
          node, err := kl.GetNode()
          if err != nil {
             klog.V(4).ErrorS(err, "No need to create a mirror pod, since failed to get node info from the cluster", "node", klog.KRef("", string(kl.nodeName)))
          } else if node.DeletionTimestamp != nil {
             klog.V(4).InfoS("No need to create a mirror pod, since node has been removed from the cluster", "node", klog.KRef("", string(kl.nodeName)))
          } else {
             klog.V(4).InfoS("Creating a mirror pod for static pod", "pod", klog.KObj(pod))
             if err := kl.mirrorPodClient.CreateMirrorPod(pod); err != nil {
                klog.ErrorS(err, "Failed creating a mirror pod for", "pod", klog.KObj(pod))
             }
          }
       }
    }
    // Make data directories for the pod
    if err := kl.makePodDataDirs(pod); err != nil {
       kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToMakePodDataDirectories, "error making pod data directories: %v", err)
       klog.ErrorS(err, "Unable to make pod data directories for pod", "pod", klog.KObj(pod))
       return false, err
    }
    // Wait for volumes to attach/mount
    if err := kl.volumeManager.WaitForAttachAndMount(ctx, pod); err != nil {
       if !wait.Interrupted(err) {
          kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedMountVolume, "Unable to attach or mount volumes: %v", err)
          klog.ErrorS(err, "Unable to attach or mount volumes for pod; skipping pod", "pod", klog.KObj(pod))
       }
       return false, err
    }
    // Fetch the pull secrets for the pod
    pullSecrets := kl.getPullSecretsForPod(pod)
    // Ensure the pod is being probed
    kl.probeManager.AddPod(pod)
    if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
       // Handle pod resize here instead of doing it in HandlePodUpdates because
       // this conveniently retries any Deferred resize requests
       // TODO(vinaykul,InPlacePodVerticalScaling): Investigate doing this in HandlePodUpdates + periodic SyncLoop scan
       //     See: https://github.com/kubernetes/kubernetes/pull/102884#discussion_r663160060
       if kl.podWorkers.CouldHaveRunningContainers(pod.UID) && !kubetypes.IsStaticPod(pod) {
          pod = kl.handlePodResourcesResize(pod)
       }
    }
    // TODO(#113606): use cancellation from the incoming context parameter, which comes from the pod worker.
    // Currently, using cancellation from that context causes test failures. To remove this WithoutCancel,
    // any wait.Interrupted errors need to be filtered from result and bypass the reasonCache - cancelling
    // the context for SyncPod is a known and deliberate error, not a generic error.
    // Use WithoutCancel instead of a new context.TODO() to propagate trace context
    // Call the container runtime's SyncPod callback
    sctx := context.WithoutCancel(ctx)
    result := kl.containerRuntime.SyncPod(sctx, pod, podStatus, pullSecrets, kl.backOff)
    kl.reasonCache.Update(pod.UID, result)
    if err := result.Error(); err != nil {
       // Do not return error if the only failures were pods in backoff
       for _, r := range result.SyncResults {
          if r.Error != kubecontainer.ErrCrashLoopBackOff && r.Error != images.ErrImagePullBackOff {
             // Do not record an event here, as we keep all event logging for sync pod failures
             // local to container runtime, so we get better errors.
             return false, err
          }
       }
       return false, nil
    }
    if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) && isPodResizeInProgress(pod, &apiPodStatus) {
       // While resize is in progress, periodically call PLEG to update pod cache
       runningPod := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
       if err, _ := kl.pleg.UpdateCache(&runningPod, pod.UID); err != nil {
          klog.ErrorS(err, "Failed to update pod cache", "pod", klog.KObj(pod))
          return false, err
       }
    }
    return false, nil
}
```

```go
// pkg/kubelet/kubelet.go:2317
func (kl *Kubelet) canRunPod(pod *v1.Pod) lifecycle.PodAdmitResult {
	attrs := &lifecycle.PodAdmitAttributes{Pod: pod}
	// Get "OtherPods". Rejected pods are failed, so only include admitted pods that are alive.
    // 是绑定到这个kubelet下的正在运行状态的pod列表
	attrs.OtherPods = kl.GetActivePods()

    // 如下图所示，有多种准入检测实现，简单看下
    // 1. appArmor: 
    // 2. topology:
    // 3. eviction: 如果没有配置节点条件，即对节点没有要求则直接创建。判断是否是重要的pod，例如静态pod、mirrorPod或者系统Priority值较高的pod，如果是则直接通过；如果节点的内存利用率已经到达NodeMemoryPressure值，查看pod的QOS级别。如果不是BestEffort则直接创建；如果pod的亲和性标签中能接受内存压力也可以创建。其他情况则不允许创建。
    // 4. node_shutdown: 检查节点是否处于shutting down状态
    // 5. patternAllow: 如果pod定义的安全上下文以及sysctl是否生效，如果没有，则创建
    // 6. predicate: 获取node信息，如果失败则不允许创建；验证kubernetes.io/os注解是否与节点的os类型一致，若不一致则不允许创建；计算绑定到该kubelet下可见的正在运行的pod的资源使用情况（更新pod使用的pod和引用的PVC）以及亲和性，并将这些数据与NodeInfo绑定；验证SidecarContainers特性门控，如果未开启，并且Init容器的重启策略设置为Always则不允许通过；验证插件资源是否可用；验证请求的资源是否不属于kubernetes.io命名空间或者没有request.前缀（如果请求的资源在NodeInfo中不存在，则说明请求的是集群级别的资源，这些资源对node是不可见的）；再来一遍亲和性验证，主要是污点和静态pod（预选）；如果前面一步的预选返回false，则看是否可以补偿回来。如果不是静态pod或者mirrorPod则不允许创建，反之则看是否可以通过驱逐pod的方式来满足资源限制
	for _, handler := range kl.softAdmitHandlers {
		if result := handler.Admit(attrs); !result.Admit {
			return result
		}
	}

	return lifecycle.PodAdmitResult{Admit: true}
}
```
![handler.Admit](statics/handler.Admit.png)