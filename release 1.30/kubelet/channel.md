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
#### newSourceFile
```go
// pkg/kubelet/config/file.go:72
func newSourceFile(path string, nodeName types.NodeName, period time.Duration, updates chan<- interface{}) *sourceFile {
    // 回调方法，当有文件发生变动时会调用这个方法推送事件到updates channel中触发pod更新
	send := func(objs []interface{}) {
		var pods []*v1.Pod
		for _, o := range objs {
			pods = append(pods, o.(*v1.Pod))
		}
		updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.FileSource}
	}
	store := cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc)
	return &sourceFile{
		path:           path,
		nodeName:       nodeName,
		period:         period,
		store:          store,
		fileKeyMapping: map[string]string{},
		updates:        updates,
		watchEvents:    make(chan *watchEvent, eventBufferLen),
	}
}

// NewUndeltaStore returns an UndeltaStore implemented with a Store.
func NewUndeltaStore(pushFunc func([]interface{}), keyFunc KeyFunc) *UndeltaStore {
	return &UndeltaStore{
		Store:    NewStore(keyFunc),
		PushFunc: pushFunc,
	}
}

// NewStore returns a Store implemented simply with a map and a lock.
func NewStore(keyFunc KeyFunc) Store {
	return &cache{
		cacheStorage: NewThreadSafeStore(Indexers{}, Indices{}),
		keyFunc:      keyFunc,
	}
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
            // 定期执行
			case <-listTicker.C:
				if err := s.listConfig(); err != nil {
					klog.ErrorS(err, "Unable to read config path", "path", s.path)
				}
            // 文件变化时触发，将文件名和pod事件类型从channel中读取，交给consumeWatchEvent处理，看下面consumeWatchEvent
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
#### listConfig
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
    // 处理监听地址是个目录的情况
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
        // 全量更新缓存
		return s.replaceStore(pods...)
    // 处理监听地址是文件的情况
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

func (s *sourceFile) replaceStore(pods ...*v1.Pod) (err error) {
	objs := []interface{}{}
	for _, pod := range pods {
		objs = append(objs, pod)
	}
    // 更新缓存中的pod内容
	return s.store.Replace(objs, "")
}
```
#### consumeWatchEvent
```go
func (s *sourceFile) consumeWatchEvent(e *watchEvent) error {
	switch e.eventType {
	case podAdd, podModify:
		pod, err := s.extractFromFile(e.fileName)
		if err != nil {
			return fmt.Errorf("can't process config file %q: %v", e.fileName, err)
		}
		return s.store.Add(pod)
	case podDelete:
		if objKey, keyExist := s.fileKeyMapping[e.fileName]; keyExist {
			pod, podExist, err := s.store.GetByKey(objKey)
			if err != nil {
				return err
			}
			if !podExist {
				return fmt.Errorf("the pod with key %s doesn't exist in cache", objKey)
			}
			if err = s.store.Delete(pod); err != nil {
				return fmt.Errorf("failed to remove deleted pod from cache: %v", err)
			}
			delete(s.fileKeyMapping, e.fileName)
		}
	}
	return nil
}
// 文件变化时调用回调函数，触发事件
func (u *UndeltaStore) Add(obj interface{}) error {
	if err := u.Store.Add(obj); err != nil {
		return err
	}
	u.PushFunc(u.Store.List())
	return nil
}

func (u *UndeltaStore) Delete(obj interface{}) error {
	if err := u.Store.Delete(obj); err != nil {
		return err
	}
	u.PushFunc(u.Store.List())
	return nil
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
        // Events结构体中包含文件名和文件操作，例如create、write、remove、rename、chmod
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
    // 忽略.开头的隐藏文件
	// Ignore file start with dots
	if strings.HasPrefix(filepath.Base(e.Name), ".") {
		klog.V(4).InfoS("Ignored pod manifest, because it starts with dots", "eventName", e.Name)
		return nil
	}
    // 组装事件类型
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
    // 将事件类型推送到channel中，这里的事件最终会被sourceFile.run里的select case语句处理实现闭环
	s.watchEvents <- &watchEvent{e.Name, eventType}
	return nil
}
```
#### NewSourceURL
```go
// NewSourceURL specifies the URL where to read the Pod configuration from, then watches it for changes.
func NewSourceURL(url string, header http.Header, nodeName types.NodeName, period time.Duration, updates chan<- interface{}) {
	config := &sourceURL{
		url:      url,
		header:   header,
		nodeName: nodeName,
		updates:  updates,
		data:     nil,
		// Timing out requests leads to retries. This client is only used to
		// read the manifest URL passed to kubelet.
		client: &http.Client{Timeout: 10 * time.Second},
	}
	klog.V(1).InfoS("Watching URL", "URL", url)
    // 主方法为config.run，看下文
	go wait.Until(config.run, period, wait.NeverStop)
}

func (s *sourceURL) run() {
	if err := s.extractFromURL(); err != nil {
		// Don't log this multiple times per minute. The first few entries should be
		// enough to get the point across.
		if s.failureLogs < 3 {
			klog.InfoS("Failed to read pods from URL", "err", err)
		} else if s.failureLogs == 3 {
			klog.InfoS("Failed to read pods from URL. Dropping verbosity of this message to V(4)", "err", err)
		} else {
			klog.V(4).InfoS("Failed to read pods from URL", "err", err)
		}
		s.failureLogs++
	} else {
		if s.failureLogs > 0 {
			klog.InfoS("Successfully read pods from URL")
			s.failureLogs = 0
		}
	}
}

func (s *sourceURL) extractFromURL() error {
	req, err := http.NewRequest("GET", s.url, nil)
	if err != nil {
		return err
	}
	req.Header = s.header
    // http 请求给定url
	resp, err := s.client.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	data, err := utilio.ReadAtMost(resp.Body, maxConfigLength)
	if err != nil {
		return err
	}
	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("%v: %v", s.url, resp.Status)
	}
    // 如果返回值为空，则触发清空事件
	if len(data) == 0 {
		// Emit an update with an empty PodList to allow HTTPSource to be marked as seen
		s.updates <- kubetypes.PodUpdate{Pods: []*v1.Pod{}, Op: kubetypes.SET, Source: kubetypes.HTTPSource}
		return fmt.Errorf("zero-length data received from %v", s.url)
	}
	// Short circuit if the data has not changed since the last time it was read.
	if bytes.Equal(data, s.data) {
		return nil
	}
	s.data = data

    // 处理单个pod的情况
	// First try as it is a single pod.
	parsed, pod, singlePodErr := tryDecodeSinglePod(data, s.applyDefaults)
	if parsed {
		if singlePodErr != nil {
			// It parsed but could not be used.
			return singlePodErr
		}
		s.updates <- kubetypes.PodUpdate{Pods: []*v1.Pod{pod}, Op: kubetypes.SET, Source: kubetypes.HTTPSource}
		return nil
	}

    // 处理多个pod的情况
	// That didn't work, so try a list of pods.
	parsed, podList, multiPodErr := tryDecodePodList(data, s.applyDefaults)
	if parsed {
		if multiPodErr != nil {
			// It parsed but could not be used.
			return multiPodErr
		}
		pods := make([]*v1.Pod, 0, len(podList.Items))
		for i := range podList.Items {
			pods = append(pods, &podList.Items[i])
		}
		s.updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.HTTPSource}
		return nil
	}

	return fmt.Errorf("%v: received '%v', but couldn't parse as "+
		"single (%v) or multiple pods (%v)",
		s.url, string(data), singlePodErr, multiPodErr)
}
// 可以看到都是最终根据内容变化封装一个PodUpdate事件推送到updates channel中触发pod变更
```

```go
// 从apiserver中获取配置变更，list&watch
// NewSourceApiserver creates a config source that watches and pulls from the apiserver.
func NewSourceApiserver(c clientset.Interface, nodeName types.NodeName, nodeHasSynced func() bool, updates chan<- interface{}) {
    // 创建一个listWatch实例，用于获取/监听本Node下所有名称空间下的pod资源，看下面NewListWatchFromClient
	lw := cache.NewListWatchFromClient(c.CoreV1().RESTClient(), "pods", metav1.NamespaceAll, fields.OneTermEqualSelector("spec.nodeName", string(nodeName)))

	// The Reflector responsible for watching pods at the apiserver should be run only after
	// the node sync with the apiserver has completed.
	klog.InfoS("Waiting for node sync before watching apiserver pods")
	go func() {
		for {
			if nodeHasSynced() {
				klog.V(4).InfoS("node sync completed")
				break
			}
			time.Sleep(WaitForAPIServerSyncPeriod)
			klog.V(4).InfoS("node sync has not completed yet")
		}
		klog.InfoS("Watching apiserver")
        // 看下面newSourceApiserverFromLW
		newSourceApiserverFromLW(lw, updates)
	}()
}
```
#### NewListWatchFromClient
```go
// NewListWatchFromClient creates a new ListWatch from the specified client, resource, namespace and field selector.
func NewListWatchFromClient(c Getter, resource string, namespace string, fieldSelector fields.Selector) *ListWatch {
    // 过滤器
	optionsModifier := func(options *metav1.ListOptions) {
		options.FieldSelector = fieldSelector.String()
	}
	return NewFilteredListWatchFromClient(c, resource, namespace, optionsModifier)
}

// NewFilteredListWatchFromClient creates a new ListWatch from the specified client, resource, namespace, and option modifier.
// Option modifier is a function takes a ListOptions and modifies the consumed ListOptions. Provide customized modifier function
// to apply modification to ListOptions with a field selector, a label selector, or any other desired options.
func NewFilteredListWatchFromClient(c Getter, resource string, namespace string, optionsModifier func(options *metav1.ListOptions)) *ListWatch {
    // List方法，调用client-go的方法通过apiserver查询etcd
	listFunc := func(options metav1.ListOptions) (runtime.Object, error) {
		optionsModifier(&options)
		return c.Get().
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, metav1.ParameterCodec).
			Do(context.TODO()).
			Get()
	}
    // watch方法调用client-go的方法通过apiserver监听etcd
	watchFunc := func(options metav1.ListOptions) (watch.Interface, error) {
		options.Watch = true
		optionsModifier(&options)
		return c.Get().
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, metav1.ParameterCodec).
			Watch(context.TODO())
	}
	return &ListWatch{ListFunc: listFunc, WatchFunc: watchFunc}
}
```

#### newSourceApiserverFromLW
```go
// newSourceApiserverFromLW holds creates a config source that watches and pulls from the apiserver.
func newSourceApiserverFromLW(lw cache.ListerWatcher, updates chan<- interface{}) {
    // send回调方法跟上面的一样，用于触发变更事件
	send := func(objs []interface{}) {
		var pods []*v1.Pod
		for _, o := range objs {
			pods = append(pods, o.(*v1.Pod))
		}
		updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.ApiserverSource}
	}
	r := cache.NewReflector(lw, &v1.Pod{}, cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc), 0)
    // client-go
	go r.Run(wait.NeverStop)
}
// 下面的代码会跳到client-go中，就不具体看了，简单概括下就是通过apiserver进行list/watch操作，发现有变化就回调send函数触发事件
```
### 接收事件
可以看到上面的三种类型config都会发送不同的事件到不同的channel中，分为cfg.Channel(ctx, kubetypes.FileSource)，cfg.Channel(ctx, kubetypes.HTTPSource)和cfg.Channel(ctx, kubetypes.ApiserverSource)
#### Channel
```go
// pkg/kubelet/config/config.go:89
// Channel creates or returns a config source channel.  The channel
// only accepts PodUpdates
func (c *PodConfig) Channel(ctx context.Context, source string) chan<- interface{} {
	c.sourcesLock.Lock()
	defer c.sourcesLock.Unlock()
	c.sources.Insert(source)
	return c.mux.ChannelWithContext(ctx, source)
}

// ChannelWithContext returns a channel where a configuration source
// can send updates of new configurations. Multiple calls with the same
// source will return the same channel. This allows change and state based sources
// to use the same channel. Different source names however will be treated as a
// union.
func (m *mux) ChannelWithContext(ctx context.Context, source string) chan interface{} {
	if len(source) == 0 {
		panic("Channel given an empty name")
	}
	m.sourceLock.Lock()
	defer m.sourceLock.Unlock()
    // 查缓存，是否存在source相同的channel，存在则直接返回，否则新建
	channel, exists := m.sources[source]
	if exists {
		return channel
	}
	newChannel := make(chan interface{})
	m.sources[source] = newChannel
    // 监听channel
	go wait.Until(func() { m.listen(source, newChannel) }, 0, ctx.Done())
	return newChannel
}

func (m *mux) listen(source string, listenChannel <-chan interface{}) {
	for update := range listenChannel {
		m.merger.Merge(source, update)
	}
}
```
#### Merge
```go
// pkg/kubelet/config/config.go:160
// Merge normalizes a set of incoming changes from different sources into a map of all Pods
// and ensures that redundant changes are filtered out, and then pushes zero or more minimal
// updates onto the update channel.  Ensures that updates are delivered in order.
func (s *podStorage) Merge(source string, change interface{}) error {
	s.updateLock.Lock()
	defer s.updateLock.Unlock()

    // 判断数据源是否已经注册过
	seenBefore := s.sourcesSeen.Has(source)
    // 看下面merge，这里返回各种操作的pod对象清单
	adds, updates, deletes, removes, reconciles := s.merge(source, change)
	firstSet := !seenBefore && s.sourcesSeen.Has(source)

    // 判断类型，从cfg的初始化可知s.mode都是PodConfigNotificationIncremental类型的，将这些都发送到updates channel中
	// deliver update notifications
	switch s.mode {
	case PodConfigNotificationIncremental:
		if len(removes.Pods) > 0 {
			s.updates <- *removes
		}
		if len(adds.Pods) > 0 {
			s.updates <- *adds
		}
		if len(updates.Pods) > 0 {
			s.updates <- *updates
		}
		if len(deletes.Pods) > 0 {
			s.updates <- *deletes
		}
		if firstSet && len(adds.Pods) == 0 && len(updates.Pods) == 0 && len(deletes.Pods) == 0 {
			// Send an empty update when first seeing the source and there are
			// no ADD or UPDATE or DELETE pods from the source. This signals kubelet that
			// the source is ready.
			s.updates <- *adds
		}
		// Only add reconcile support here, because kubelet doesn't support Snapshot update now.
		if len(reconciles.Pods) > 0 {
			s.updates <- *reconciles
		}

	case PodConfigNotificationSnapshotAndUpdates:
		if len(removes.Pods) > 0 || len(adds.Pods) > 0 || firstSet {
			s.updates <- kubetypes.PodUpdate{Pods: s.mergedState().([]*v1.Pod), Op: kubetypes.SET, Source: source}
		}
		if len(updates.Pods) > 0 {
			s.updates <- *updates
		}
		if len(deletes.Pods) > 0 {
			s.updates <- *deletes
		}

	case PodConfigNotificationSnapshot:
		if len(updates.Pods) > 0 || len(deletes.Pods) > 0 || len(adds.Pods) > 0 || len(removes.Pods) > 0 || firstSet {
			s.updates <- kubetypes.PodUpdate{Pods: s.mergedState().([]*v1.Pod), Op: kubetypes.SET, Source: source}
		}

	case PodConfigNotificationUnknown:
		fallthrough
	default:
		panic(fmt.Sprintf("unsupported PodConfigNotificationMode: %#v", s.mode))
	}

	return nil
}
```
#### merge
```go
func (s *podStorage) merge(source string, change interface{}) (adds, updates, deletes, removes, reconciles *kubetypes.PodUpdate) {
	s.podLock.Lock()
	defer s.podLock.Unlock()

	addPods := []*v1.Pod{}
	updatePods := []*v1.Pod{}
	deletePods := []*v1.Pod{}
	removePods := []*v1.Pod{}
	reconcilePods := []*v1.Pod{}
    // 获取指定source的pod信息
	pods := s.pods[source]
	if pods == nil {
		pods = make(map[types.UID]*v1.Pod)
	}

	// updatePodFunc is the local function which updates the pod cache *oldPods* with new pods *newPods*.
	// After updated, new pod will be stored in the pod cache *pods*.
	// Notice that *pods* and *oldPods* could be the same cache.
	updatePodsFunc := func(newPods []*v1.Pod, oldPods, pods map[types.UID]*v1.Pod) {
		filtered := filterInvalidPods(newPods, source, s.recorder)
		for _, ref := range filtered {
			// Annotate the pod with the source before any comparison.
			if ref.Annotations == nil {
				ref.Annotations = make(map[string]string)
			}
            // 赋值annotations的来源
			ref.Annotations[kubetypes.ConfigSourceAnnotationKey] = source
            // 忽略静态pod
			// ignore static pods
			if !kubetypes.IsStaticPod(ref) {
                // 记录指标metrics和日志
				s.startupSLIObserver.ObservedPodOnWatch(ref, time.Now())
			}
            // 通过UID遍历oldPods，如果找到，判断pod下一步如何处理，如果没找到，记录事件，添加到addPods里
			if existing, found := oldPods[ref.UID]; found {
				pods[ref.UID] = existing
                // 判断pod需要如何处理
				needUpdate, needReconcile, needGracefulDelete := checkAndUpdatePod(existing, ref)
				if needUpdate {
					updatePods = append(updatePods, existing)
				} else if needReconcile {
					reconcilePods = append(reconcilePods, existing)
				} else if needGracefulDelete {
					deletePods = append(deletePods, existing)
				}
				continue
			}
			recordFirstSeenTime(ref)
			pods[ref.UID] = ref
			addPods = append(addPods, ref)
		}
	}

	update := change.(kubetypes.PodUpdate)
	switch update.Op {
	case kubetypes.ADD, kubetypes.UPDATE, kubetypes.DELETE:
		if update.Op == kubetypes.ADD {
			klog.V(4).InfoS("Adding new pods from source", "source", source, "pods", klog.KObjSlice(update.Pods))
		} else if update.Op == kubetypes.DELETE {
			klog.V(4).InfoS("Gracefully deleting pods from source", "source", source, "pods", klog.KObjSlice(update.Pods))
		} else {
			klog.V(4).InfoS("Updating pods from source", "source", source, "pods", klog.KObjSlice(update.Pods))
		}
        // 调用函数更新缓存
		updatePodsFunc(update.Pods, pods, pods)

	case kubetypes.REMOVE:
		klog.V(4).InfoS("Removing pods from source", "source", source, "pods", klog.KObjSlice(update.Pods))
		for _, value := range update.Pods {
			if existing, found := pods[value.UID]; found {
				// this is a delete
				delete(pods, value.UID)
				removePods = append(removePods, existing)
				continue
			}
			// this is a no-op
		}

	case kubetypes.SET:
		klog.V(4).InfoS("Setting pods for source", "source", source)
		s.markSourceSet(source)
        // 将pods内容移到oldPods中，并新创建一个pods来清空pods
		// Clear the old map entries by just creating a new map
		oldPods := pods
		pods = make(map[types.UID]*v1.Pod)
		updatePodsFunc(update.Pods, oldPods, pods)
		for uid, existing := range oldPods {
			if _, found := pods[uid]; !found {
                // 添加到removePods中
				// this is a delete
				removePods = append(removePods, existing)
			}
		}

	default:
		klog.InfoS("Received invalid update type", "type", update)

	}

	s.pods[source] = pods

	adds = &kubetypes.PodUpdate{Op: kubetypes.ADD, Pods: copyPods(addPods), Source: source}
	updates = &kubetypes.PodUpdate{Op: kubetypes.UPDATE, Pods: copyPods(updatePods), Source: source}
	deletes = &kubetypes.PodUpdate{Op: kubetypes.DELETE, Pods: copyPods(deletePods), Source: source}
	removes = &kubetypes.PodUpdate{Op: kubetypes.REMOVE, Pods: copyPods(removePods), Source: source}
	reconciles = &kubetypes.PodUpdate{Op: kubetypes.RECONCILE, Pods: copyPods(reconcilePods), Source: source}

	return adds, updates, deletes, removes, reconciles
}
```

```go
// checkAndUpdatePod updates existing, and:
//   - if ref makes a meaningful change, returns needUpdate=true
//   - if ref makes a meaningful change, and this change is graceful deletion, returns needGracefulDelete=true
//   - if ref makes no meaningful change, but changes the pod status, returns needReconcile=true
//   - else return all false
//     Now, needUpdate, needGracefulDelete and needReconcile should never be both true
func checkAndUpdatePod(existing, ref *v1.Pod) (needUpdate, needReconcile, needGracefulDelete bool) {
    // 检查spec、labels、deletionTimestamp、DeletionGracePeriodSeconds、Annotations是否一致，如果有一项不一致且Status也不一样，表示需要重新协调，更新状态后直接返回
	// 1. this is a reconcile
	// TODO: it would be better to update the whole object and only preserve certain things
	//       like the source annotation or the UID (to ensure safety)
	if !podsDifferSemantically(existing, ref) {
		// this is not an update
		// Only check reconcile when it is not an update, because if the pod is going to
		// be updated, an extra reconcile is unnecessary
		if !reflect.DeepEqual(existing.Status, ref.Status) {
			// Pod with changed pod status needs reconcile, because kubelet should
			// be the source of truth of pod status.
			existing.Status = ref.Status
			needReconcile = true
		}
		return
	}
    // 新老pod的状态或者配置发生了变化，更新
	// Overwrite the first-seen time with the existing one. This is our own
	// internal annotation, there is no need to update.
	ref.Annotations[kubetypes.ConfigFirstSeenAnnotationKey] = existing.Annotations[kubetypes.ConfigFirstSeenAnnotationKey]

	existing.Spec = ref.Spec
	existing.Labels = ref.Labels
	existing.DeletionTimestamp = ref.DeletionTimestamp
	existing.DeletionGracePeriodSeconds = ref.DeletionGracePeriodSeconds
	existing.Status = ref.Status
	updateAnnotations(existing, ref)

    // 如果DeletionTimestamp有值，说明需要优雅删除，否则需要更新
	// 2. this is an graceful delete
	if ref.DeletionTimestamp != nil {
		needGracefulDelete = true
	} else {
		// 3. this is an update
		needUpdate = true
	}

	return
}
```
#### 总结
```go
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableServer bool) {
	// 最终上面的各种事件都会推到podCfg的updates channel中，k.Run的时候又会去监听这些channel，这样流程就串起来了
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

### plegCh
#### GenericPLEG
kubelet中的pleg用于监听容器运行时的状态并且向kubelet推送容器的变化，主要是让podWorkers重新协调pod的状态，例如容器挂了需要重启。
kubelet中的pleg分为两种，GenericPLEG和EventedPLEG。EventedPLEG需要kubelet启动时显式指定开启。

```go
type GenericPLEG struct {
	// The container runtime.
	runtime kubecontainer.Runtime
	// The channel from which the subscriber listens events.
	eventChannel chan *PodLifecycleEvent
	// The internal cache for pod/container information.
	podRecords podRecords
	// Time of the last relisting.
	relistTime atomic.Value
	// Cache for storing the runtime states required for syncing pods.
	cache kubecontainer.Cache
	// For testability.
	clock clock.Clock
	// Pods that failed to have their status retrieved during a relist. These pods will be
	// retried during the next relisting.
	podsToReinspect map[types.UID]*kubecontainer.Pod
	// Stop the Generic PLEG by closing the channel.
	stopCh chan struct{}
	// Locks the relisting of the Generic PLEG
	relistLock sync.Mutex
	// Indicates if the Generic PLEG is running or not
	isRunning bool
	// Locks the start/stop operation of Generic PLEG
	runningMu sync.Mutex
	// Indicates relisting related parameters
	relistDuration *RelistDuration
	// Mutex to serialize updateCache called by relist vs UpdateCache interface
	podCacheMutex sync.Mutex
}
    // pkg/kubelet/kubelet.go:744
    genericRelistDuration := &pleg.RelistDuration{
        RelistPeriod:    genericPlegRelistPeriod,
        RelistThreshold: genericPlegRelistThreshold,
    }
    // 初始化pleg
    klet.pleg = pleg.NewGenericPLEG(klet.containerRuntime, eventChannel, genericRelistDuration, klet.podCache, clock.RealClock{})
    // pkg/kubelet/kubelet.go:1664
    // 启动pleg
    kl.pleg.Start()
```
#### NewGenericPLEG
```go
// pkg/kubelet/pleg/generic.go:119
// eventChannel := make(chan *pleg.PodLifecycleEvent, plegChannelCapacity)
// klet.podCache = kubecontainer.NewCache()
func NewGenericPLEG(runtime kubecontainer.Runtime, eventChannel chan *PodLifecycleEvent,
	relistDuration *RelistDuration, cache kubecontainer.Cache,
	clock clock.Clock) PodLifecycleEventGenerator {
	return &GenericPLEG{
		relistDuration: relistDuration,
		runtime:        runtime,
		eventChannel:   eventChannel,
		podRecords:     make(podRecords),
		cache:          cache,
		clock:          clock,
	}
}
```
#### NewCache
```go
// 初始化cache
// NewCache creates a pod cache.
func NewCache() Cache {
	return &cache{pods: map[types.UID]*data{}, subscribers: map[types.UID][]*subRecord{}}
}
```
#### syncLoop
```go
// pkg/kubelet/kubelet.go:2336
func (kl *Kubelet) syncLoop(ctx context.Context, updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
    ......
    // pleg的event channel
	plegCh := kl.pleg.Watch()
	......
	for {
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
		if !kl.syncLoopIteration(ctx, updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}
```
#### Start
```go
// pkg/kubelet/pleg/generic.go:140
func (g *GenericPLEG) Start() {
	g.runningMu.Lock()
	defer g.runningMu.Unlock()
	if !g.isRunning {
		g.isRunning = true
		g.stopCh = make(chan struct{})
		go wait.Until(g.Relist, g.relistDuration.RelistPeriod, g.stopCh)
	}
}
```
#### Relist
```go
// 查询容器运行时获取pods/containers的数据，与内部维护的缓存中的数据进行对比，根据结果生成不同的事件
// Relist queries the container runtime for list of pods/containers, compare
// with the internal pods/containers, and generates events accordingly.
func (g *GenericPLEG) Relist() {
	g.relistLock.Lock()
	defer g.relistLock.Unlock()

	ctx := context.Background()
	klog.V(5).InfoS("GenericPLEG: Relisting")

    // metrics指标中记录relist耗时
	if lastRelistTime := g.getRelistTime(); !lastRelistTime.IsZero() {
		metrics.PLEGRelistInterval.Observe(metrics.SinceInSeconds(lastRelistTime))
	}

	timestamp := g.clock.Now()
	defer func() {
		metrics.PLEGRelistDuration.Observe(metrics.SinceInSeconds(timestamp))
	}()

    // 通过容器运行时获取所有pod的数据，包括sandbox容器
	// Get all the pods.
	podList, err := g.runtime.GetPods(ctx, true)
	if err != nil {
		klog.ErrorS(err, "GenericPLEG: Unable to retrieve pods")
		return
	}
    // 更新时间
	g.updateRelistTime(timestamp)
    // 转换类型
	pods := kubecontainer.Pods(podList)
    // 设置metres，计算处于不同状态的容器数量
	// update running pod and container count
	updateRunningPodAndContainerMetrics(pods)
    // 更新pod缓存
	g.podRecords.setCurrent(pods)

    // 比对，生成事件
	// Compare the old and the current pods, and generate events.
	eventsByPodID := map[types.UID][]*PodLifecycleEvent{}
	for pid := range g.podRecords {
		oldPod := g.podRecords.getOld(pid)
		pod := g.podRecords.getCurrent(pid)
        // 获取所有容器，包含普通容器和sandbox容器
		// Get all containers in the old and the new pod.
		allContainers := getContainersFromPods(oldPod, pod)
		for _, container := range allContainers {
            // 计算事件，看下面computeEvents
			events := computeEvents(oldPod, pod, &container.ID)
			for _, e := range events {
                // 更新事件，即更新eventsByPodID
				updateEvents(eventsByPodID, e)
			}
		}
	}

	var needsReinspection map[types.UID]*kubecontainer.Pod
    // 缓存是否开启，如果开启，需要再次检查上次未通过的pod
	if g.cacheEnabled() {
		needsReinspection = make(map[types.UID]*kubecontainer.Pod)
	}

    // 如果有podID关联的事件产生，则更新缓存
	// If there are events associated with a pod, we should update the
	// podCache.
	for pid, events := range eventsByPodID {
		pod := g.podRecords.getCurrent(pid)
		if g.cacheEnabled() {
            // 检查并且更新缓存，如果缓存开启，且检查出现error，则将其置入needsReinspection下一个周期再检查
			// updateCache() will inspect the pod and update the cache. If an
			// error occurs during the inspection, we want PLEG to retry again
			// in the next relist. To achieve this, we do not update the
			// associated podRecord of the pod, so that the change will be
			// detect again in the next relist.
			// TODO: If many pods changed during the same relist period,
			// inspecting the pod and getting the PodStatus to update the cache
			// serially may take a while. We should be aware of this and
			// parallelize if needed.
			if err, updated := g.updateCache(ctx, pod, pid); err != nil {
				// Rely on updateCache calling GetPodStatus to log the actual error.
				klog.V(4).ErrorS(err, "PLEG: Ignoring events for pod", "pod", klog.KRef(pod.Namespace, pod.Name))

				// make sure we try to reinspect the pod during the next relisting
				needsReinspection[pid] = pod

				continue
			} else {
                // 因为这个pod已经关联事件了，如果不删除的话这个pod将会被检查两次
				// this pod was in the list to reinspect and we did so because it had events, so remove it
				// from the list (we don't want the reinspection code below to inspect it a second time in
				// this relist execution)
				delete(g.podsToReinspect, pid)
				if utilfeature.DefaultFeatureGate.Enabled(features.EventedPLEG) {
					if !updated {
						continue
					}
				}
			}
		}
        // 更新缓存，r.old = r.current r.current = nil，老的变成新的，新的置为nil
		// Update the internal storage and send out the events.
		g.podRecords.update(pid)

		// Map from containerId to exit code; used as a temporary cache for lookup
		containerExitCode := make(map[string]int)

		for i := range events {
            // 跳过，该类型目前为未知状态，且没有其他组件使用
			// Filter out events that are not reliable and no other components use yet.
			if events[i].Type == ContainerChanged {
				continue
			}
			select {
            // 推送到plegCh，这个channel是通过上面kl.pleg.Watch()得来的
			case g.eventChannel <- events[i]:
			default:
				metrics.PLEGDiscardEvents.Inc()
				klog.ErrorS(nil, "Event channel is full, discard this relist() cycle event")
			}
            // 容器挂了
			// Log exit code of containers when they finished in a particular event
			if events[i].Type == ContainerDied {
                // 没有退出码
				// Fill up containerExitCode map for ContainerDied event when first time appeared
				if len(containerExitCode) == 0 && pod != nil && g.cache != nil {
                    // 获取缓存中已更新的pod状态，补充退出码
					// Get updated podStatus
					status, err := g.cache.Get(pod.ID)
					if err == nil {
						for _, containerStatus := range status.ContainerStatuses {
							containerExitCode[containerStatus.ID.ID] = containerStatus.ExitCode
						}
					}
				}
                // 如果有退出码，记录容器ID，容器ID对应的退出码
				if containerID, ok := events[i].Data.(string); ok {
					if exitCode, ok := containerExitCode[containerID]; ok && pod != nil {
						klog.V(2).InfoS("Generic (PLEG): container finished", "podID", pod.ID, "containerID", containerID, "exitCode", exitCode)
					}
				}
			}
		}
	}
    // 如果开启了缓存
	if g.cacheEnabled() {
        // 存在需要重新检验的pod
		// reinspect any pods that failed inspection during the previous relist
		if len(g.podsToReinspect) > 0 {
			klog.V(5).InfoS("GenericPLEG: Reinspecting pods that previously failed inspection")
			for pid, pod := range g.podsToReinspect {
                // 更新缓存
				if err, _ := g.updateCache(ctx, pod, pid); err != nil {
					// Rely on updateCache calling GetPodStatus to log the actual error.
					klog.V(5).ErrorS(err, "PLEG: pod failed reinspection", "pod", klog.KRef(pod.Namespace, pod.Name))
					needsReinspection[pid] = pod
				}
			}
		}
        // 更新缓存时间
		// Update the cache timestamp.  This needs to happen *after*
		// all pods have been properly updated in the cache.
		g.cache.UpdateTime(timestamp)
	}
    // 赋值
	// make sure we retain the list of pods that need reinspecting the next time relist is called
	g.podsToReinspect = needsReinspection
}
```

#### computeEvents 
```go
func computeEvents(oldPod, newPod *kubecontainer.Pod, cid *kubecontainer.ContainerID) []*PodLifecycleEvent {
	var pid types.UID
	if oldPod != nil {
		pid = oldPod.ID
	} else if newPod != nil {
		pid = newPod.ID
	}
	oldState := getContainerState(oldPod, cid)
	newState := getContainerState(newPod, cid)
	return generateEvents(pid, cid.ID, oldState, newState)
}

func getContainerState(pod *kubecontainer.Pod, cid *kubecontainer.ContainerID) plegContainerState {
	// Default to the non-existent state.
	state := plegContainerNonExistent
	if pod == nil {
		return state
	}
    // 寻找pod中对应的容器id
	c := pod.FindContainerByID(*cid)
	if c != nil {
		return convertState(c.State)
	}
	// Search through sandboxes too.
	c = pod.FindSandboxByID(*cid)
	if c != nil {
		return convertState(c.State)
	}

	return state
}
// 容器状态转换为pleg状态
// created-> unknown running -> running exited -> exited unknown -> unknown
func convertState(state kubecontainer.State) plegContainerState {
	switch state {
	case kubecontainer.ContainerStateCreated:
		// kubelet doesn't use the "created" state yet, hence convert it to "unknown".
		return plegContainerUnknown
	case kubecontainer.ContainerStateRunning:
		return plegContainerRunning
	case kubecontainer.ContainerStateExited:
		return plegContainerExited
	case kubecontainer.ContainerStateUnknown:
		return plegContainerUnknown
	default:
		panic(fmt.Sprintf("unrecognized container state: %v", state))
	}
}
// 生成事件
func generateEvents(podID types.UID, cid string, oldState, newState plegContainerState) []*PodLifecycleEvent {
	if newState == oldState {
		return nil
	}

	klog.V(4).InfoS("GenericPLEG", "podUID", podID, "containerID", cid, "oldState", oldState, "newState", newState)
	switch newState {
	case plegContainerRunning:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerStarted, Data: cid}}
	case plegContainerExited:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerDied, Data: cid}}
	case plegContainerUnknown:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerChanged, Data: cid}}
	case plegContainerNonExistent:
		switch oldState {
		case plegContainerExited:
			// We already reported that the container died before.
			return []*PodLifecycleEvent{{ID: podID, Type: ContainerRemoved, Data: cid}}
		default:
			return []*PodLifecycleEvent{{ID: podID, Type: ContainerDied, Data: cid}, {ID: podID, Type: ContainerRemoved, Data: cid}}
		}
	default:
		panic(fmt.Sprintf("unrecognized container state: %v", newState))
	}
}
```


```go
	case e := <-plegCh:
        // 如果事件类型不是ContainerRemoved
		if isSyncPodWorthy(e) {
            // 从podManager中获取pod详情
			// PLEG event for a pod; sync it.
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				klog.V(2).InfoS("SyncLoop (PLEG): event for pod", "pod", klog.KObj(pod), "event", e)
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} else {
				// If the pod no longer exists, ignore the event.
				klog.V(4).InfoS("SyncLoop (PLEG): pod does not exist, ignore irrelevant event", "event", e)
			}
		}
        // 如果事件类型是ContainerDied
		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
                // 清除pod相关的容器
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
```

#### HandlePodSyncs
```go
func (kl *Kubelet) HandlePodSyncs(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
        // 从podManager的缓存中获取pod相关信息
		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		if wasMirror {
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
			// Syncing a mirror pod is a programmer error since the intent of sync is to
			// batch notify all pending work. We should make it impossible to double sync,
			// but for now log a programmer error to prevent accidental introduction.
			klog.V(3).InfoS("Programmer error, HandlePodSyncs does not expect to receive mirror pods", "podUID", pod.UID, "mirrorPodUID", mirrorPod.UID)
			continue
		}
        // 生成事件
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			Pod:        pod,
			MirrorPod:  mirrorPod,
			UpdateType: kubetypes.SyncPodSync,
			StartTime:  start,
		})
	}
}
```

#### 总结
plegCh数据的来源是kubelet从容器运行时中获取的pod和container列表，并自身维护了一个缓存，固定间隔时间获取新的数据，并与缓存中的数据进行比较，根据比对结果生成事件推送到plegCh中。podWorkers从plegCh中读取数据进行处理。

### syncCh
```go
// syncCh每隔一秒读一次
func (kl *Kubelet) syncLoopIteration(ctx context.Context, configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
    ......
    case <-syncCh:
		// Sync pods waiting for sync
        // 看下面getPodsToSync
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		klog.V(4).InfoS("SyncLoop (SYNC) pods", "total", len(podsToSync), "pods", klog.KObjSlice(podsToSync))
        // 调用podWorkers执行更新操作
		handler.HandlePodSyncs(podsToSync)
    ......
}
```
#### getPodsToSync
```go
// Get pods which should be resynchronized. Currently, the following pod should be resynchronized:
//   - pod whose work is ready.
//   - internal modules that request sync of a pod.
//
// This method does not return orphaned pods (those known only to the pod worker that may have
// been deleted from configuration). Those pods are synced by HandlePodCleanups as a consequence
// of driving the state machine to completion.
//
// TODO: Consider synchronizing all pods which have not recently been acted on to be resilient
// to bugs that might prevent updates from being delivered (such as the previous bug with
// orphaned pods). Instead of asking the work queue for pending work, consider asking the
// PodWorker which pods should be synced.
func (kl *Kubelet) getPodsToSync() []*v1.Pod {
    // 从podManager中获取所有pod缓存信息
	allPods := kl.podManager.GetPods()
    // workQueue是一个用于触发podWorker的队列，实际实现中包含了时钟、锁和一个键为podID值为时间的map
    // 看下面workQueue，这里获取时间早于现在的队列中的内容
	podUIDs := kl.workQueue.GetWork()
	podUIDSet := sets.NewString()
    // 转为set
	for _, podUID := range podUIDs {
		podUIDSet.Insert(string(podUID))
	}
	var podsToSync []*v1.Pod
	for _, pod := range allPods {
        // 遍历所有的pod，如果podID存在于从workQueue中获取的podUID，则说明这个pod需要同步
		if podUIDSet.Has(string(pod.UID)) {
			// The work of the pod is ready
			podsToSync = append(podsToSync, pod)
			continue
		}
        // 如果不存在，检查pod是否能通过podSyncLoopHandler校验，如果可以则这个pod需要同步
		for _, podSyncLoopHandler := range kl.PodSyncLoopHandlers {
            // 看下面ShouldSync
			if podSyncLoopHandler.ShouldSync(pod) {
				podsToSync = append(podsToSync, pod)
				break
			}
		}
	}
	return podsToSync
}
```
#### workQueue
workQueue中的数据来源：syncLoopIteration方法中处理来自不同channel中的事件，进入不同的handler，交给podWorkers.UpdatePod函数处理，最终进去podWorkerLoop调用podWorkers.CompleteWork。
```go
type basicWorkQueue struct {
	clock clock.Clock
	lock  sync.Mutex
	queue map[types.UID]time.Time
}

// pkg/kubelet/pod_workers.go:1484
// 在出现错误或者在下一个同步周期时重新排队，然后执行要处理的工作
// completeWork requeues on error or the next sync interval and then immediately executes any pending
// work.
func (p *podWorkers) completeWork(podUID types.UID, phaseTransition bool, syncErr error) {
	// Requeue the last update if the last sync returned error.
	switch {
    // 阶段转换
	case phaseTransition:
		p.workQueue.Enqueue(podUID, 0)
    // 同步没错误
	case syncErr == nil:
		// No error; requeue at the regular resync interval.
		p.workQueue.Enqueue(podUID, wait.Jitter(p.resyncInterval, workerResyncIntervalJitterFactor))
    // 网络未就绪错误
	case strings.Contains(syncErr.Error(), NetworkNotReadyErrorMsg):
		// Network is not ready; back off for short period of time and retry as network might be ready soon.
		p.workQueue.Enqueue(podUID, wait.Jitter(backOffOnTransientErrorPeriod, workerBackOffPeriodJitterFactor))
	default:
		// Error occurred during the sync; back off and then retry.
		p.workQueue.Enqueue(podUID, wait.Jitter(p.backOffPeriod, workerBackOffPeriodJitterFactor))
	}

	// if there is a pending update for this worker, requeue immediately, otherwise
	// clear working status
	p.podLock.Lock()
	defer p.podLock.Unlock()
	if status, ok := p.podSyncStatuses[podUID]; ok {
        // 如果有挂起的更新，立即重新排队
		if status.pendingUpdate != nil {
			select {
			case p.podUpdates[podUID] <- struct{}{}:
				klog.V(4).InfoS("Requeueing pod due to pending update", "podUID", podUID)
			default:
				klog.V(4).InfoS("Pending update already queued", "podUID", podUID)
			}
		} else {
			status.working = false
		}
	}
}

// 就是更新时间
// pkg/kubelet/util/queue/work_queue.go:64
func (q *basicWorkQueue) Enqueue(item types.UID, delay time.Duration) {
	q.lock.Lock()
	defer q.lock.Unlock()
	q.queue[item] = q.clock.Now().Add(delay)
}

```
#### ShouldSync
```go
// ShouldSync returns true if the pod is past its active deadline.
func (m *activeDeadlineHandler) ShouldSync(pod *v1.Pod) bool {
	return m.pastActiveDeadline(pod)
}

// pkg/kubelet/active_deadline.go:79
// 检查pod是否超过了活动期限
// pastActiveDeadline returns true if the pod has been active for more than its ActiveDeadlineSeconds
func (m *activeDeadlineHandler) pastActiveDeadline(pod *v1.Pod) bool {
    // 用于限制pod的运行时间，常用于job类型的pod
	// no active deadline was specified
	if pod.Spec.ActiveDeadlineSeconds == nil {
		return false
	}
	// get the latest status to determine if it was started
	podStatus, ok := m.podStatusProvider.GetPodStatus(pod.UID)
	if !ok {
        // 使用传入的Pod的当前状态作为获取的最新pod状态
		podStatus = pod.Status
	}
	// we have no start time so just return
	if podStatus.StartTime.IsZero() {
		return false
	}
	// determine if the deadline was exceeded
	start := podStatus.StartTime.Time
	duration := m.clock.Since(start)
	allowedDuration := time.Duration(*pod.Spec.ActiveDeadlineSeconds) * time.Second
	return duration >= allowedDuration
}
```
#### 总结
syncCh会定期读取缓存中所有的pod信息和队列中的pod信息，遍历所有pod信息，校验是否需要同步，最终还是交给podWorkers完成Update操作

### housekeepingCh
```go
	case <-housekeepingCh:
        // 检查上面configCh中初始化的三种config来源，本地文件，http，listwatch资源是否准备好，确保必须的pod信息源都已经同步
		if !kl.sourcesReady.AllReady() {
			// If the sources aren't ready or volume manager has not yet synced the states,
			// skip housekeeping, as we may accidentally delete pods from unready sources.
			klog.V(4).InfoS("SyncLoop (housekeeping, skipped): sources aren't ready yet")
		} else {
			start := time.Now()
			klog.V(4).InfoS("SyncLoop (housekeeping)")
            // 看下面HandlePodCleanups
			if err := handler.HandlePodCleanups(ctx); err != nil {
				klog.ErrorS(err, "Failed cleaning pods")
			}
            // 如果清理时间过长记录错误
			duration := time.Since(start)
			if duration > housekeepingWarningDuration {
				klog.ErrorS(fmt.Errorf("housekeeping took too long"), "Housekeeping took longer than expected", "expected", housekeepingWarningDuration, "actual", duration.Round(time.Millisecond))
			}
			klog.V(4).InfoS("SyncLoop (housekeeping) end", "duration", duration.Round(time.Millisecond))
		}
	}
```
#### HandlePodCleanups
```go
// pkg/kubelet/kubelet_pods.go:1101
// 清理工作包含终止pod workers，杀掉不需要的pod，移除孤儿容器、卷目录等
// HandlePodCleanups performs a series of cleanup work, including terminating
// pod workers, killing unwanted pods, and removing orphaned volumes/pod
// directories. No config changes are sent to pod workers while this method
// is executing which means no new pods can appear. After this method completes
// the desired state of the kubelet should be reconciled with the actual state
// in the pod worker and other pod-related components.
//
// This function is executed by the main sync loop, so it must execute quickly
// and all nested calls should be asynchronous. Any slow reconciliation actions
// should be performed by other components (like the volume manager). The duration
// of this call is the minimum latency for static pods to be restarted if they
// are updated with a fixed UID (most should use a dynamic UID), and no config
// updates are delivered to the pod workers while this method is running.
func (kl *Kubelet) HandlePodCleanups(ctx context.Context) error {
	// The kubelet lacks checkpointing, so we need to introspect the set of pods
	// in the cgroup tree prior to inspecting the set of pods in our pod manager.
	// this ensures our view of the cgroup tree does not mistakenly observe pods
	// that are added after the fact...
	var (
		cgroupPods map[types.UID]cm.CgroupName
		err        error
	)
	if kl.cgroupsPerQOS {
		pcm := kl.containerManager.NewPodContainerManager()
        // 查看pod用到的cgroup
		cgroupPods, err = pcm.GetAllPodsFromCgroups()
		if err != nil {
			return fmt.Errorf("failed to get list of pods that still exist on cgroup mounts: %v", err)
		}
	}
    // 从podManager中获取pod和mirrorPod
	allPods, mirrorPods, orphanedMirrorPodFullnames := kl.podManager.GetPodsAndMirrorPods()

	// Pod phase progresses monotonically. Once a pod has reached a final state,
	// it should never leave regardless of the restart policy. The statuses
	// of such pods should not be changed, and there is no need to sync them.
	// TODO: the logic here does not handle two cases:
	//   1. If the containers were removed immediately after they died, kubelet
	//      may fail to generate correct statuses, let alone filtering correctly.
	//   2. If kubelet restarted before writing the terminated status for a pod
	//      to the apiserver, it could still restart the terminated pod (even
	//      though the pod was not considered terminated by the apiserver).
	// These two conditions could be alleviated by checkpointing kubelet.

	// Stop the workers for terminated pods not in the config source
	klog.V(3).InfoS("Clean up pod workers for terminated pods")
    // 从podWorkers中获取所有有状态的pod，看下面SyncKnownPods
	workingPods := kl.podWorkers.SyncKnownPods(allPods)

	// Reconcile: At this point the pod workers have been pruned to the set of
	// desired pods. Pods that must be restarted due to UID reuse, or leftover
	// pods from previous runs, are not known to the pod worker.

	allPodsByUID := make(map[types.UID]*v1.Pod)
	for _, pod := range allPods {
		allPodsByUID[pod.UID] = pod
	}

	// Identify the set of pods that have workers, which should be all pods
	// from config that are not terminated, as well as any terminating pods
	// that have already been removed from config. Pods that are terminating
	// will be added to possiblyRunningPods, to prevent overly aggressive
	// cleanup of pod cgroups.
	stringIfTrue := func(t bool) string {
		if t {
			return "true"
		}
		return ""
	}
	runningPods := make(map[types.UID]sets.Empty)
	possiblyRunningPods := make(map[types.UID]sets.Empty)
	for uid, sync := range workingPods {
		switch sync.State {
		case SyncPod:
			runningPods[uid] = struct{}{}
			possiblyRunningPods[uid] = struct{}{}
		case TerminatingPod:
			possiblyRunningPods[uid] = struct{}{}
		default:
		}
	}

    // 更新缓存，记录时间戳
	// Retrieve the list of running containers from the runtime to perform cleanup.
	// We need the latest state to avoid delaying restarts of static pods that reuse
	// a UID.
	if err := kl.runtimeCache.ForceUpdateIfOlder(ctx, kl.clock.Now()); err != nil {
		klog.ErrorS(err, "Error listing containers")
		return err
	}
    // 从缓存中获取pod
	runningRuntimePods, err := kl.runtimeCache.GetPods(ctx)
	if err != nil {
		klog.ErrorS(err, "Error listing containers")
		return err
	}

	// Stop probing pods that are not running
	klog.V(3).InfoS("Clean up probes for terminated pods")
    // 如果探针不存在于possiblyRunningPods，则停止探针
	kl.probeManager.CleanupPods(possiblyRunningPods)

	// Remove orphaned pod statuses not in the total list of known config pods
	klog.V(3).InfoS("Clean up orphaned pod statuses")
    // statusManager中删除不存在于pod清单中的状态缓存，管理器中存在，allPod列表不存在
	kl.removeOrphanedPodStatuses(allPods, mirrorPods)

    // 清理只有孤儿pod的命名空间分配
	// Remove orphaned pod user namespace allocations (if any).
	klog.V(3).InfoS("Clean up orphaned pod user namespace allocations")
	if err = kl.usernsManager.CleanupOrphanedPodUsernsAllocations(allPods, runningRuntimePods); err != nil {
		klog.ErrorS(err, "Failed cleaning up orphaned pod user namespaces allocations")
	}

    // 清理孤儿pod的挂载卷占用
	// Remove orphaned volumes from pods that are known not to have any
	// containers. Note that we pass all pods (including terminated pods) to
	// the function, so that we don't remove volumes associated with terminated
	// but not yet deleted pods.
	// TODO: this method could more aggressively cleanup terminated pods
	// in the future (volumes, mount dirs, logs, and containers could all be
	// better separated)
	klog.V(3).InfoS("Clean up orphaned pod directories")
	err = kl.cleanupOrphanedPodDirs(allPods, runningRuntimePods)
	if err != nil {
		// We want all cleanup tasks to be run even if one of them failed. So
		// we just log an error here and continue other cleanup tasks.
		// This also applies to the other clean up tasks.
		klog.ErrorS(err, "Failed cleaning up orphaned pod directories")
	}

    // 删除镜像Pod
	// Remove any orphaned mirror pods (mirror pods are tracked by name via the
	// pod worker)
	klog.V(3).InfoS("Clean up orphaned mirror pods")
	for _, podFullname := range orphanedMirrorPodFullnames {
		if !kl.podWorkers.IsPodForMirrorPodTerminatingByFullName(podFullname) {
			_, err := kl.mirrorPodClient.DeleteMirrorPod(podFullname, nil)
			if err != nil {
				klog.ErrorS(err, "Encountered error when deleting mirror pod", "podName", podFullname)
			} else {
				klog.V(3).InfoS("Deleted mirror pod", "podName", podFullname)
			}
		}
	}

	// After pruning pod workers for terminated pods get the list of active pods for
	// metrics and to determine restarts.
	activePods := kl.filterOutInactivePods(allPods)
	allRegularPods, allStaticPods := splitPodsByStatic(allPods)
	activeRegularPods, activeStaticPods := splitPodsByStatic(activePods)
	metrics.DesiredPodCount.WithLabelValues("").Set(float64(len(allRegularPods)))
	metrics.DesiredPodCount.WithLabelValues("true").Set(float64(len(allStaticPods)))
	metrics.ActivePodCount.WithLabelValues("").Set(float64(len(activeRegularPods)))
	metrics.ActivePodCount.WithLabelValues("true").Set(float64(len(activeStaticPods)))
	metrics.MirrorPodCount.Set(float64(len(mirrorPods)))

	// At this point, the pod worker is aware of which pods are not desired (SyncKnownPods).
	// We now look through the set of active pods for those that the pod worker is not aware of
	// and deliver an update. The most common reason a pod is not known is because the pod was
	// deleted and recreated with the same UID while the pod worker was driving its lifecycle (very
	// very rare for API pods, common for static pods with fixed UIDs). Containers that may still
	// be running from a previous execution must be reconciled by the pod worker's sync method.
	// We must use active pods because that is the set of admitted pods (podManager includes pods
	// that will never be run, and statusManager tracks already rejected pods).
	var restartCount, restartCountStatic int
	for _, desiredPod := range activePods {
        // 如果期望的pod不存在于podWorkers中，则重新推给podWorkers执行重启操作
		if _, knownPod := workingPods[desiredPod.UID]; knownPod {
			continue
		}

		klog.V(3).InfoS("Pod will be restarted because it is in the desired set and not known to the pod workers (likely due to UID reuse)", "podUID", desiredPod.UID)
		isStatic := kubetypes.IsStaticPod(desiredPod)
		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(desiredPod)
		if pod == nil || wasMirror {
			klog.V(2).InfoS("Programmer error, restartable pod was a mirror pod but activePods should never contain a mirror pod", "podUID", desiredPod.UID)
			continue
		}
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			UpdateType: kubetypes.SyncPodCreate,
			Pod:        pod,
			MirrorPod:  mirrorPod,
		})
        // 更新缓存
		// the desired pod is now known as well
		workingPods[desiredPod.UID] = PodWorkerSync{State: SyncPod, HasConfig: true, Static: isStatic}
		if isStatic {
			// restartable static pods are the normal case
			restartCountStatic++
		} else {
			// almost certainly means shenanigans, as API pods should never have the same UID after being deleted and recreated
			// unless there is a major API violation
			restartCount++
		}
	}
	metrics.RestartedPodTotal.WithLabelValues("true").Add(float64(restartCountStatic))
	metrics.RestartedPodTotal.WithLabelValues("").Add(float64(restartCount))

    // 过滤出那些没有DeletionTimestamp，不在terminal阶段，且不在podWorkers中的pod，然后去掉那些在容器运行时缓存中状态为running的pod，剩下的pod需要被kill掉，同样，推送给podWorkers去做
	// Complete termination of deleted pods that are not runtime pods (don't have
	// running containers), are terminal, and are not known to pod workers.
	// An example is pods rejected during kubelet admission that have never
	// started before (i.e. does not have an orphaned pod).
	// Adding the pods with SyncPodKill to pod workers allows to proceed with
	// force-deletion of such pods, yet preventing re-entry of the routine in the
	// next invocation of HandlePodCleanups.
	for _, pod := range kl.filterTerminalPodsToDelete(allPods, runningRuntimePods, workingPods) {
		klog.V(3).InfoS("Handling termination and deletion of the pod to pod workers", "pod", klog.KObj(pod), "podUID", pod.UID)
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			UpdateType: kubetypes.SyncPodKill,
			Pod:        pod,
		})
	}

    // 终止那些容器运行时缓存中存在，但不在podWorkers缓存中的pod
	// Finally, terminate any pods that are observed in the runtime but not present in the list of
	// known running pods from config. If we do terminate running runtime pods that will happen
	// asynchronously in the background and those will be processed in the next invocation of
	// HandlePodCleanups.
	var orphanCount int
	for _, runningPod := range runningRuntimePods {
		// If there are orphaned pod resources in CRI that are unknown to the pod worker, terminate them
		// now. Since housekeeping is exclusive to other pod worker updates, we know that no pods have
		// been added to the pod worker in the meantime. Note that pods that are not visible in the runtime
		// but which were previously known are terminated by SyncKnownPods().
		_, knownPod := workingPods[runningPod.ID]
		if !knownPod {
			one := int64(1)
			killPodOptions := &KillPodOptions{
				PodTerminationGracePeriodSecondsOverride: &one,
			}
			klog.V(2).InfoS("Clean up containers for orphaned pod we had not seen before", "podUID", runningPod.ID, "killPodOptions", killPodOptions)
			kl.podWorkers.UpdatePod(UpdatePodOptions{
				UpdateType:     kubetypes.SyncPodKill,
				RunningPod:     runningPod,
				KillPodOptions: killPodOptions,
			})

            // 写入workingPods缓存
			// the running pod is now known as well
			workingPods[runningPod.ID] = PodWorkerSync{State: TerminatingPod, Orphan: true}
			orphanCount++
		}
	}
	metrics.OrphanedRuntimePodTotal.Add(float64(orphanCount))

	// Now that we have recorded any terminating pods, and added new pods that should be running,
	// record a summary here. Not all possible combinations of PodWorkerSync values are valid.
	counts := make(map[PodWorkerSync]int)
	for _, sync := range workingPods {
		counts[sync]++
	}
	for validSync, configState := range map[PodWorkerSync]string{
		{HasConfig: true, Static: true}:                "desired",
		{HasConfig: true, Static: false}:               "desired",
		{Orphan: true, HasConfig: true, Static: true}:  "orphan",
		{Orphan: true, HasConfig: true, Static: false}: "orphan",
		{Orphan: true, HasConfig: false}:               "runtime_only",
	} {
		for _, state := range []PodWorkerState{SyncPod, TerminatingPod, TerminatedPod} {
			validSync.State = state
			count := counts[validSync]
			delete(counts, validSync)
			staticString := stringIfTrue(validSync.Static)
			if !validSync.HasConfig {
				staticString = "unknown"
			}
			metrics.WorkingPodCount.WithLabelValues(state.String(), configState, staticString).Set(float64(count))
		}
	}
	if len(counts) > 0 {
		// in case a combination is lost
		klog.V(3).InfoS("Programmer error, did not report a kubelet_working_pods metric for a value returned by SyncKnownPods", "counts", counts)
	}

    // 如果开启了cgroup，清理cgroups
	// Remove any cgroups in the hierarchy for pods that are definitely no longer
	// running (not in the container runtime).
	if kl.cgroupsPerQOS {
		pcm := kl.containerManager.NewPodContainerManager()
		klog.V(3).InfoS("Clean up orphaned pod cgroups")
		kl.cleanupOrphanedPodCgroups(pcm, cgroupPods, possiblyRunningPods)
	}

	// Cleanup any backoff entries.
	kl.backOff.GC()
	return nil
}
```

#### SyncKnownPods
```go
// SyncKnownPods will purge any fully terminated pods that are not in the desiredPods
// list, which means SyncKnownPods must be called in a threadsafe manner from calls
// to UpdatePods for new pods. Because the podworker is dependent on UpdatePod being
// invoked to drive a pod's state machine, if a pod is missing in the desired list the
// pod worker must be responsible for delivering that update. The method returns a map
// of known workers that are not finished with a value of SyncPodTerminated,
// SyncPodKill, or SyncPodSync depending on whether the pod is terminated, terminating,
// or syncing.
func (p *podWorkers) SyncKnownPods(desiredPods []*v1.Pod) map[types.UID]PodWorkerSync {
	workers := make(map[types.UID]PodWorkerSync)
	known := make(map[types.UID]struct{})
	for _, pod := range desiredPods {
		known[pod.UID] = struct{}{}
	}

	p.podLock.Lock()
	defer p.podLock.Unlock()

	p.podsSynced = true
    // podSyncStatuses来源于pkg/kubelet/pod_workers.go：798
	for uid, status := range p.podSyncStatuses {
		// We retain the worker history of any pod that is still desired according to
		// its UID. However, there are two scenarios during a sync that result in us
		// needing to purge the history:
		//
		// 1. The pod is no longer desired (the local version is orphaned)
		// 2. The pod received a kill update and then a subsequent create, which means
		//    the UID was reused in the source config (vanishingly rare for API servers,
		//    common for static pods that have specified a fixed UID)
		//
		// In the former case we wish to bound the amount of information we store for
		// deleted pods. In the latter case we wish to minimize the amount of time before
		// we restart the static pod. If we succeed at removing the worker, then we
		// omit it from the returned map of known workers, and the caller of SyncKnownPods
		// is expected to send a new UpdatePod({UpdateType: Create}).
		_, knownPod := known[uid]
        // 是否是孤儿pod
		orphan := !knownPod
        // 如果pod需要被重启或者是个孤儿pod
		if status.restartRequested || orphan {
            // 删除pod的工作状态
			if p.removeTerminatedWorker(uid, status, orphan) {
				// no worker running, we won't return it
				continue
			}
		}
        // 重新赋值
		sync := PodWorkerSync{
			State:  status.WorkType(),
			Orphan: orphan,
		}
		switch {
		case status.activeUpdate != nil:
			if status.activeUpdate.Pod != nil {
				sync.HasConfig = true
				sync.Static = kubetypes.IsStaticPod(status.activeUpdate.Pod)
			}
		case status.pendingUpdate != nil:
			if status.pendingUpdate.Pod != nil {
				sync.HasConfig = true
				sync.Static = kubetypes.IsStaticPod(status.pendingUpdate.Pod)
			}
		}
		workers[uid] = sync
	}
	return workers
}
```