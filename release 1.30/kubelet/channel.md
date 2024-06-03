## syncLoopIteration中的configCh,plegCh,syncCh,housekeepingCh

下面这个方法是Kubelet中的主要逻辑，它处理了来自四种channel中的不同事件，下面详细分析一下。
```go
func (kl *Kubelet) syncLoopIteration(ctx context.Context, configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool
```
源码注释中写的比较清楚

|名称|数据来源|用途|
|---|---|---|
|configCh|配置变更|针对不同事件找到合适的handler对相应的pod应用配置变更|
|syncCh|定期事件同步|定期同步所有的pod状态|
|plegCh|PLEG变更|更新缓存，同步pod（部分）|
|housekeepingCh|清理事件|触发垃圾回收|

### ConfigCh
configCh的来源是startKubelet方法中的第二个参数
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
再往上找找，找到PodConfig结构体以及对应的方法
```go
// PodConfig is a configuration mux that merges many sources of pod configuration into a single
// consistent structure, and then delivers incremental change notifications to listeners
// in order.
type PodConfig struct {
	pods *podStorage
	mux  *mux

	// the channel of denormalized changes passed to listeners
	updates chan kubetypes.PodUpdate

	// contains the list of all configured sources
	sourcesLock sync.Mutex
	sources     sets.String
}

// Updates returns a channel of updates to the configuration, properly denormalized.
func (c *PodConfig) Updates() <-chan kubetypes.PodUpdate {
	return c.updates
}
```
可以看到是这里初始化的PodConfig
```go
// pkg/kubelet/kubelet.go:409
	if kubeDeps.PodConfig == nil {
		var err error
        // 看下面makePodSourceConfig
		kubeDeps.PodConfig, err = makePodSourceConfig(kubeCfg, kubeDeps, nodeName, nodeHasSynced)
		if err != nil {
			return nil, err
		}
	}
```
#### makePodSourceConfig
```go
// makePodSourceConfig creates a config.PodConfig from the given
// KubeletConfiguration or returns an error.
func makePodSourceConfig(kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *Dependencies, nodeName types.NodeName, nodeHasSynced func() bool) (*config.PodConfig, error) {
	manifestURLHeader := make(http.Header)
	if len(kubeCfg.StaticPodURLHeader) > 0 {
		for k, v := range kubeCfg.StaticPodURLHeader {
			for i := range v {
				manifestURLHeader.Add(k, v[i])
			}
		}
	}

	// source of all configuration
	cfg := config.NewPodConfig(config.PodConfigNotificationIncremental, kubeDeps.Recorder, kubeDeps.PodStartupLatencyTracker)

	// TODO:  it needs to be replaced by a proper context in the future
	ctx := context.TODO()

	// define file config source
	if kubeCfg.StaticPodPath != "" {
		klog.InfoS("Adding static pod path", "path", kubeCfg.StaticPodPath)
        // 看下面NewSourceFile
		config.NewSourceFile(kubeCfg.StaticPodPath, nodeName, kubeCfg.FileCheckFrequency.Duration, cfg.Channel(ctx, kubetypes.FileSource))
	}

	// define url config source
	if kubeCfg.StaticPodURL != "" {
		klog.InfoS("Adding pod URL with HTTP header", "URL", kubeCfg.StaticPodURL, "header", manifestURLHeader)
		config.NewSourceURL(kubeCfg.StaticPodURL, manifestURLHeader, nodeName, kubeCfg.HTTPCheckFrequency.Duration, cfg.Channel(ctx, kubetypes.HTTPSource))
	}

	if kubeDeps.KubeClient != nil {
		klog.InfoS("Adding apiserver pod source")
		config.NewSourceApiserver(kubeDeps.KubeClient, nodeName, nodeHasSynced, cfg.Channel(ctx, kubetypes.ApiserverSource))
	}
	return cfg, nil
}
```
#### NewSourceFile
```go
// 监听配置文件的变化，一般是/etc/kubernetes/manifest
// NewSourceFile watches a config file for changes.
func NewSourceFile(path string, nodeName types.NodeName, period time.Duration, updates chan<- interface{}) {
	// "github.com/sigma/go-inotify" requires a path without trailing "/"
	path = strings.TrimRight(path, string(os.PathSeparator))

	config := newSourceFile(path, nodeName, period, updates)
	klog.V(1).InfoS("Watching path", "path", path)
    // 看下面sourceFile.run
	config.run()
}
```
#### sourceFile.run
```go
func (s *sourceFile) run() {
	listTicker := time.NewTicker(s.period)

	go func() {
        // 马上执行一次，之后定时执行或者有文件变更再执行
		// Read path immediately to speed up startup.
		if err := s.listConfig(); err != nil {
			klog.ErrorS(err, "Unable to read config path", "path", s.path)
		}
		for {
			select {
			case <-listTicker.C:
				if err := s.listConfig(); err != nil {
					klog.ErrorS(err, "Unable to read config path", "path", s.path)
				}
			case e := <-s.watchEvents:
				if err := s.consumeWatchEvent(e); err != nil {
					klog.ErrorS(err, "Unable to process watch event")
				}
			}
		}
	}()
    // 看下面startWatch
	s.startWatch()
}
```

```go
// pkg/kubelet/config/file.go:121
func (s *sourceFile) listConfig() error {
	path := s.path
	statInfo, err := os.Stat(path)
	if err != nil {
		if !os.IsNotExist(err) {
			return err
		}
        // 如果目录不存在，则触发一个清空来自fileSource所有pod的操作，这就是为什么删除/etc/kubernetes/manifest目录下的yaml配置文件后集群中该节点的对应静态pod会被删除的原因
		// Emit an update with an empty PodList to allow FileSource to be marked as seen
		s.updates <- kubetypes.PodUpdate{Pods: []*v1.Pod{}, Op: kubetypes.SET, Source: kubetypes.FileSource}
		return fmt.Errorf("path does not exist, ignoring")
	}

	switch {
	case statInfo.Mode().IsDir():
		pods, err := s.extractFromDir(path)
		if err != nil {
			return err
		}
		if len(pods) == 0 {
			// Emit an update with an empty PodList to allow FileSource to be marked as seen
			s.updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.FileSource}
			return nil
		}
		return s.replaceStore(pods...)

	case statInfo.Mode().IsRegular():
		pod, err := s.extractFromFile(path)
		if err != nil {
			return err
		}
		return s.replaceStore(pod)

	default:
		return fmt.Errorf("path is not a directory or file")
	}
}
```

```go
func (s *sourceFile) replaceStore(pods ...*v1.Pod) (err error) {
	objs := []interface{}{}
	for _, pod := range pods {
		objs = append(objs, pod)
	}
    // 更新缓存中的pod内容
	return s.store.Replace(objs, "")
}
```

```go
// pkg/kubelet/config/file_linux.go:51
func (s *sourceFile) startWatch() {
	backOff := flowcontrol.NewBackOff(retryPeriod, maxRetryPeriod)
	backOffID := "watch"

	go wait.Forever(func() {
		if backOff.IsInBackOffSinceUpdate(backOffID, time.Now()) {
			return
		}

		if err := s.doWatch(); err != nil {
			klog.ErrorS(err, "Unable to read config path", "path", s.path)
			if _, retryable := err.(*retryableError); !retryable {
				backOff.Next(backOffID, time.Now())
			}
		}
	}, retryPeriod)
}

// pkg/kubelet/config/file_linux.go:69
func (s *sourceFile) doWatch() error {
	_, err := os.Stat(s.path)
	if err != nil {
		if !os.IsNotExist(err) {
			return err
		}
		// Emit an update with an empty PodList to allow FileSource to be marked as seen
		s.updates <- kubetypes.PodUpdate{Pods: []*v1.Pod{}, Op: kubetypes.SET, Source: kubetypes.FileSource}
		return &retryableError{"path does not exist, ignoring"}
	}

	w, err := fsnotify.NewWatcher()
	if err != nil {
		return fmt.Errorf("unable to create inotify: %v", err)
	}
	defer w.Close()

	err = w.Add(s.path)
	if err != nil {
		return fmt.Errorf("unable to create inotify for path %q: %v", s.path, err)
	}

	for {
		select {
		case event := <-w.Events:
			if err = s.produceWatchEvent(&event); err != nil {
				return fmt.Errorf("error while processing inotify event (%+v): %v", event, err)
			}
		case err = <-w.Errors:
			return fmt.Errorf("error while watching %q: %v", s.path, err)
		}
	}
}

// pkg/kubelet/config/file_linux.go:103
func (s *sourceFile) produceWatchEvent(e *fsnotify.Event) error {
	// Ignore file start with dots
	if strings.HasPrefix(filepath.Base(e.Name), ".") {
		klog.V(4).InfoS("Ignored pod manifest, because it starts with dots", "eventName", e.Name)
		return nil
	}
	var eventType podEventType
	switch {
	case (e.Op & fsnotify.Create) > 0:
		eventType = podAdd
	case (e.Op & fsnotify.Write) > 0:
		eventType = podModify
	case (e.Op & fsnotify.Chmod) > 0:
		eventType = podModify
	case (e.Op & fsnotify.Remove) > 0:
		eventType = podDelete
	case (e.Op & fsnotify.Rename) > 0:
		eventType = podDelete
	default:
		// Ignore rest events
		return nil
	}

	s.watchEvents <- &watchEvent{e.Name, eventType}
	return nil
}
```