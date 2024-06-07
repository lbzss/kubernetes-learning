### proberManager
用于管理pod探针，proberManager为每一个定义了探针的容器创建对应的探针工作器（worker）。
工作器周期性探测分配的容器并且缓存其结果。当UpdatePodStatus被调用时proberManager使用缓存的探测结果在PodStatus中设置合适的状态。
```go
// Manager manages pod probing. It creates a probe "worker" for every container that specifies a
// probe (AddPod). The worker periodically probes its assigned container and caches the results. The
// manager use the cached probe results to set the appropriate Ready state in the PodStatus when
// requested (UpdatePodStatus). Updating probe parameters is not currently supported.
type Manager interface {
    // 为每一个容器新建一个探针工作器。该方法在每个pod创建时被调用
	// AddPod creates new probe workers for every container probe. This should be called for every
	// pod created.
	AddPod(pod *v1.Pod)

    // 在终止期间停止存活和启动探针检测
	// StopLivenessAndStartup handles stopping liveness and startup probes during termination.
	StopLivenessAndStartup(pod *v1.Pod)

    // 清除已移除pod的状态，包括停止探针工作器和删除缓存结果
	// RemovePod handles cleaning up the removed pod state, including terminating probe workers and
	// deleting cached results.
	RemovePod(pod *v1.Pod)

    // 清理那些不应该再运行的pod，传入的参数desiredPods是期望正常运行的pod清单，这部分pod不会被删掉
	// CleanupPods handles cleaning up pods which should no longer be running.
	// It takes a map of "desired pods" which should not be cleaned up.
	CleanupPods(desiredPods map[types.UID]sets.Empty)

    // 根据容器运行状态、缓存的探针探测结果和探针工作器的状态为每一个容器更新合适的状态
	// UpdatePodStatus modifies the given PodStatus with the appropriate Ready state for each
	// container based on container running status, cached probe results and worker states.
	UpdatePodStatus(*v1.Pod, *v1.PodStatus)
}

type manager struct {
    // podUID,容器名称，探针类型为key，工作器为value的map
	// Map of active workers for probes
	workers map[probeKey]*worker
	// Lock for accessing & mutating workers
	workerLock sync.RWMutex
    // 从statusManager中读取podIP和容器ID
	// The statusManager cache provides pod IP and container IDs for probing.
	statusManager status.Manager

	// readinessManager manages the results of readiness probes
	readinessManager results.Manager

	// livenessManager manages the results of liveness probes
	livenessManager results.Manager

	// startupManager manages the results of startup probes
	startupManager results.Manager

    // 探针工作方法
	// prober executes the probe actions.
	prober *prober

	start time.Time
}
```
#### NewManager初始化方法
```go
// NewManager creates a Manager for pod probing.
func NewManager(
	statusManager status.Manager,
	livenessManager results.Manager,
	readinessManager results.Manager,
	startupManager results.Manager,
	runner kubecontainer.CommandRunner,
	recorder record.EventRecorder) Manager {

    // 用于初始化各种执行探测的客户端
	prober := newProber(runner, recorder)
	return &manager{
		statusManager:    statusManager,
		prober:           prober,
		readinessManager: readinessManager,
		livenessManager:  livenessManager,
		startupManager:   startupManager,
		workers:          make(map[probeKey]*worker),
		start:            clock.RealClock{}.Now(),
	}
}

```

#### AddPod
```go
// pkg/kubelet/prober/prober_manager.go:181
func (m *manager) AddPod(pod *v1.Pod) {
	m.workerLock.Lock()
	defer m.workerLock.Unlock()

	key := probeKey{podUID: pod.UID}
    // 为每一个容器添加探针，针对pod中的普通容器和可重启的init容器
	for _, c := range append(pod.Spec.Containers, getRestartableInitContainers(pod)...) {
		key.containerName = c.Name

		if c.StartupProbe != nil {
			key.probeType = startup
			if _, ok := m.workers[key]; ok {
				klog.V(8).ErrorS(nil, "Startup probe already exists for container",
					"pod", klog.KObj(pod), "containerName", c.Name)
				return
			}
			w := newWorker(m, startup, pod, c)
			m.workers[key] = w
			go w.run()
		}

		if c.ReadinessProbe != nil {
			key.probeType = readiness
			if _, ok := m.workers[key]; ok {
				klog.V(8).ErrorS(nil, "Readiness probe already exists for container",
					"pod", klog.KObj(pod), "containerName", c.Name)
				return
			}
			w := newWorker(m, readiness, pod, c)
			m.workers[key] = w
			go w.run()
		}

		if c.LivenessProbe != nil {
			key.probeType = liveness
			if _, ok := m.workers[key]; ok {
				klog.V(8).ErrorS(nil, "Liveness probe already exists for container",
					"pod", klog.KObj(pod), "containerName", c.Name)
				return
			}
			w := newWorker(m, liveness, pod, c)
			m.workers[key] = w
			go w.run()
		}
	}
}
```
#### run
```go
func (w *worker) run() {
	ctx := context.Background()
	probeTickerPeriod := time.Duration(w.spec.PeriodSeconds) * time.Second

    // 如果kubelet重启了，给一个随机因子乘以本身设置的间隔，待启动完成后等一段事件再处理下面的逻辑
	// If kubelet restarted the probes could be started in rapid succession.
	// Let the worker wait for a random portion of tickerPeriod before probing.
	// Do it only if the kubelet has started recently.
	if probeTickerPeriod > time.Since(w.probeManager.start) {
		time.Sleep(time.Duration(rand.Float64() * float64(probeTickerPeriod)))
	}

	probeTicker := time.NewTicker(probeTickerPeriod)

	defer func() {
		// Clean up.
		probeTicker.Stop()
		if !w.containerID.IsEmpty() {
			w.resultsManager.Remove(w.containerID)
		}

		w.probeManager.removeWorker(w.pod.UID, w.container.Name, w.probeType)
		ProberResults.Delete(w.proberResultsSuccessfulMetricLabels)
		ProberResults.Delete(w.proberResultsFailedMetricLabels)
		ProberResults.Delete(w.proberResultsUnknownMetricLabels)
		ProberDuration.Delete(w.proberDurationSuccessfulMetricLabels)
		ProberDuration.Delete(w.proberDurationUnknownMetricLabels)
	}()

probeLoop:
	for w.doProbe(ctx) {
		// Wait for next probe tick.
		select {
		case <-w.stopCh:
			break probeLoop
		case <-probeTicker.C:
		case <-w.manualTriggerCh:
			// continue
		}
	}
}
```

```go
// pkg/kubelet/prober/worker.go:198
// 比较简单，就是判断容器的状态，是否继续循环探测，如果容器状态为failed、succeed、terminated或者deleteTimestamp不为空，或者容器的uid不匹配（说明容器已经重启了，但是缓存中没来得及更新），不继续探测，其他情况下都要继续探测
// doProbe probes the container once and records the result.
// Returns whether the worker should continue.
func (w *worker) doProbe(ctx context.Context) (keepGoing bool) {
	defer func() { recover() }() // Actually eat panics (HandleCrash takes care of logging)
	defer runtime.HandleCrash(func(_ interface{}) { keepGoing = true })

	startTime := time.Now()
    // statusManager中找不到说明要么pod已经被删除了，要么还没在同步apiserver之前还没来得及创建pod，继续探测
	status, ok := w.probeManager.statusManager.GetPodStatus(w.pod.UID)
	if !ok {
		// Either the pod has not been created yet, or it was already deleted.
		klog.V(3).InfoS("No status for pod", "pod", klog.KObj(w.pod))
		return true
	}

    // 如果pod的状态已经是failed或者succeed，则需要退出循环
	// Worker should terminate if pod is terminated.
	if status.Phase == v1.PodFailed || status.Phase == v1.PodSucceeded {
		klog.V(3).InfoS("Pod is terminated, exiting probe worker",
			"pod", klog.KObj(w.pod), "phase", status.Phase)
		return false
	}

    // statusManager中已经有pod了，查指定容器名称的容器是否存在
	c, ok := podutil.GetContainerStatus(status.ContainerStatuses, w.container.Name)
    // 不存在，或者容器ID为空，查init容器是否存在
	if !ok || len(c.ContainerID) == 0 {
		c, ok = podutil.GetContainerStatus(status.InitContainerStatuses, w.container.Name)
        // 不存在，或容器ID为空，说明已经被删除或者还没有被创建，继续探测，等statusManager中的数据缓存更新再判断
		if !ok || len(c.ContainerID) == 0 {
			// Either the container has not been created yet, or it was deleted.
			klog.V(3).InfoS("Probe target container not found",
				"pod", klog.KObj(w.pod), "containerName", w.container.Name)
			return true // Wait for more information.
		}
	}

	if w.containerID.String() != c.ContainerID {
		if !w.containerID.IsEmpty() {
			w.resultsManager.Remove(w.containerID)
		}
		w.containerID = kubecontainer.ParseContainerID(c.ContainerID)
		w.resultsManager.Set(w.containerID, w.initialValue, w.pod)
		// We've got a new container; resume probing.
		w.onHold = false
	}

	if w.onHold {
		// Worker is on hold until there is a new container.
		return true
	}

	if c.State.Running == nil {
		klog.V(3).InfoS("Non-running container probed",
			"pod", klog.KObj(w.pod), "containerName", w.container.Name)
		if !w.containerID.IsEmpty() {
			w.resultsManager.Set(w.containerID, results.Failure, w.pod)
		}
		// Abort if the container will not be restarted.
		return c.State.Terminated == nil ||
			w.pod.Spec.RestartPolicy != v1.RestartPolicyNever
	}

	// Graceful shutdown of the pod.
	if w.pod.ObjectMeta.DeletionTimestamp != nil && (w.probeType == liveness || w.probeType == startup) {
		klog.V(3).InfoS("Pod deletion requested, setting probe result to success",
			"probeType", w.probeType, "pod", klog.KObj(w.pod), "containerName", w.container.Name)
		if w.probeType == startup {
			klog.InfoS("Pod deletion requested before container has fully started",
				"pod", klog.KObj(w.pod), "containerName", w.container.Name)
		}
		// Set a last result to ensure quiet shutdown.
		w.resultsManager.Set(w.containerID, results.Success, w.pod)
		// Stop probing at this point.
		return false
	}

	// Probe disabled for InitialDelaySeconds.
	if int32(time.Since(c.State.Running.StartedAt.Time).Seconds()) < w.spec.InitialDelaySeconds {
		return true
	}

	if c.Started != nil && *c.Started {
		// Stop probing for startup once container has started.
		// we keep it running to make sure it will work for restarted container.
		if w.probeType == startup {
			return true
		}
	} else {
		// Disable other probes until container has started.
		if w.probeType != startup {
			return true
		}
	}

    // 真正执行探针探测
	// Note, exec probe does NOT have access to pod environment variables or downward API
	result, err := w.probeManager.prober.probe(ctx, w.probeType, w.pod, status, w.container, w.containerID)
	if err != nil {
		// Prober error, throw away the result.
		return true
	}

    // metrics指标
	switch result {
	case results.Success:
		ProberResults.With(w.proberResultsSuccessfulMetricLabels).Inc()
		ProberDuration.With(w.proberDurationSuccessfulMetricLabels).Observe(time.Since(startTime).Seconds())
	case results.Failure:
		ProberResults.With(w.proberResultsFailedMetricLabels).Inc()
	default:
		ProberResults.With(w.proberResultsUnknownMetricLabels).Inc()
		ProberDuration.With(w.proberDurationUnknownMetricLabels).Observe(time.Since(startTime).Seconds())
	}

	if w.lastResult == result {
		w.resultRun++
	} else {
		w.lastResult = result
		w.resultRun = 1
	}

	if (result == results.Failure && w.resultRun < int(w.spec.FailureThreshold)) ||
		(result == results.Success && w.resultRun < int(w.spec.SuccessThreshold)) {
		// Success or failure is below threshold - leave the probe state unchanged.
		return true
	}
    // 更新结果
	w.resultsManager.Set(w.containerID, result, w.pod)

	if (w.probeType == liveness || w.probeType == startup) && result == results.Failure {
		// The container fails a liveness/startup check, it will need to be restarted.
		// Stop probing until we see a new container ID. This is to reduce the
		// chance of hitting #21751, where running `docker exec` when a
		// container is being stopped may lead to corrupted container state.
		w.onHold = true
		w.resultRun = 0
	}

	return true
}
```

```go
// probe probes the container.
func (pb *prober) probe(ctx context.Context, probeType probeType, pod *v1.Pod, status v1.PodStatus, container v1.Container, containerID kubecontainer.ContainerID) (results.Result, error) {
	var probeSpec *v1.Probe
	switch probeType {
	case readiness:
		probeSpec = container.ReadinessProbe
	case liveness:
		probeSpec = container.LivenessProbe
	case startup:
		probeSpec = container.StartupProbe
	default:
		return results.Failure, fmt.Errorf("unknown probe type: %q", probeType)
	}

	if probeSpec == nil {
		klog.InfoS("Probe is nil", "probeType", probeType, "pod", klog.KObj(pod), "podUID", pod.UID, "containerName", container.Name)
		return results.Success, nil
	}
    // 组装不同的参数配置，使用客户端去执行操作
	result, output, err := pb.runProbeWithRetries(ctx, probeType, probeSpec, pod, status, container, containerID, maxProbeRetries)
	if err != nil || (result != probe.Success && result != probe.Warning) {
		// Probe failed in one way or another.
		if err != nil {
			klog.V(1).ErrorS(err, "Probe errored", "probeType", probeType, "pod", klog.KObj(pod), "podUID", pod.UID, "containerName", container.Name)
			pb.recordContainerEvent(pod, &container, v1.EventTypeWarning, events.ContainerUnhealthy, "%s probe errored: %v", probeType, err)
		} else { // result != probe.Success
			klog.V(1).InfoS("Probe failed", "probeType", probeType, "pod", klog.KObj(pod), "podUID", pod.UID, "containerName", container.Name, "probeResult", result, "output", output)
			pb.recordContainerEvent(pod, &container, v1.EventTypeWarning, events.ContainerUnhealthy, "%s probe failed: %s", probeType, output)
		}
		return results.Failure, err
	}
	if result == probe.Warning {
		pb.recordContainerEvent(pod, &container, v1.EventTypeWarning, events.ContainerProbeWarning, "%s probe warning: %s", probeType, output)
		klog.V(3).InfoS("Probe succeeded with a warning", "probeType", probeType, "pod", klog.KObj(pod), "podUID", pod.UID, "containerName", container.Name, "output", output)
	} else {
		klog.V(3).InfoS("Probe succeeded", "probeType", probeType, "pod", klog.KObj(pod), "podUID", pod.UID, "containerName", container.Name)
	}
	return results.Success, nil
}
```

```go
func (pb *prober) runProbe(ctx context.Context, probeType probeType, p *v1.Probe, pod *v1.Pod, status v1.PodStatus, container v1.Container, containerID kubecontainer.ContainerID) (probe.Result, string, error) {
	timeout := time.Duration(p.TimeoutSeconds) * time.Second
	switch {
	case p.Exec != nil:
		klog.V(4).InfoS("Exec-Probe runProbe", "pod", klog.KObj(pod), "containerName", container.Name, "execCommand", p.Exec.Command)
		command := kubecontainer.ExpandContainerCommandOnlyStatic(p.Exec.Command, container.Env)
        // 调用cmd.Start()方法执行命令
		return pb.exec.Probe(pb.newExecInContainer(ctx, container, containerID, command, timeout))

	case p.HTTPGet != nil:
		req, err := httpprobe.NewRequestForHTTPGetAction(p.HTTPGet, &container, status.PodIP, "probe")
		if err != nil {
			return probe.Unknown, "", err
		}
		if klogV4 := klog.V(4); klogV4.Enabled() {
			port := req.URL.Port()
			host := req.URL.Hostname()
			path := req.URL.Path
			scheme := req.URL.Scheme
			headers := p.HTTPGet.HTTPHeaders
			klogV4.InfoS("HTTP-Probe", "scheme", scheme, "host", host, "port", port, "path", path, "timeout", timeout, "headers", headers)
		}
        // httpClient.Do
		return pb.http.Probe(req, timeout)

	case p.TCPSocket != nil:
		port, err := probe.ResolveContainerPort(p.TCPSocket.Port, &container)
		if err != nil {
			return probe.Unknown, "", err
		}
		host := p.TCPSocket.Host
		if host == "" {
			host = status.PodIP
		}
		klog.V(4).InfoS("TCP-Probe", "host", host, "port", port, "timeout", timeout)
        // net.Dial方法
		return pb.tcp.Probe(host, port, timeout)

	case p.GRPC != nil:
		host := status.PodIP
		service := ""
		if p.GRPC.Service != nil {
			service = *p.GRPC.Service
		}
		klog.V(4).InfoS("GRPC-Probe", "host", host, "service", service, "port", p.GRPC.Port, "timeout", timeout)
        // grpc.Dial
		return pb.grpc.Probe(host, service, int(p.GRPC.Port), timeout)

	default:
		klog.InfoS("Failed to find probe builder for container", "containerName", container.Name)
		return probe.Unknown, "", fmt.Errorf("missing probe handler for %s:%s", format.Pod(pod), container.Name)
	}
}
```
