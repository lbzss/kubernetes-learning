### podManager
用于管理pod的生命周期和状态，提供了管理和访问Pod信息的功能。
- pod的创建和删除
- pod的状态管理
- pod的信息存储和检索
- 镜像Pod的处理
- Pod的更新和同步
- 与其他组件交互

在kubelet要对pod进行操作时，会先在podManager中更新对应的pod元数据，之后再将需要变更的对象信息传给podWorker用于处理

```go
klet.podManager = kubepod.NewBasicPodManager()

// NewBasicPodManager returns a functional Manager.
func NewBasicPodManager() Manager {
	pm := &basicManager{}
	pm.SetPods(nil)
	return pm
}

// 实际上就是本地维护的一堆缓存，可以通过UID进行查找
// basicManager is a functional Manager.
//
// All fields in basicManager are read-only and are updated calling SetPods,
// AddPod, UpdatePod, or RemovePod.
type basicManager struct {
	// Protects all internal maps.
	lock sync.RWMutex

    // 通过UID检索pod
	// Regular pods indexed by UID.
	podByUID map[kubetypes.ResolvedPodUID]*v1.Pod
    // 通过MirrorPodID检索Pod
	// Mirror pods indexed by UID.
	mirrorPodByUID map[kubetypes.MirrorPodUID]*v1.Pod

    // 通过全名检索
	// Pods indexed by full name for easy access.
	podByFullName       map[string]*v1.Pod
	mirrorPodByFullName map[string]*v1.Pod

    // 镜像PodId到普通podId的映射关系
	// Mirror pod UID to pod UID map.
	translationByUID map[kubetypes.MirrorPodUID]kubetypes.ResolvedPodUID
}
```

#### updatePodsInternal
```go
// pkg/kubelet/pod/pod_manager.go:168
// 更新Pod信息，更新各个哈希表
// updatePodsInternal replaces the given pods in the current state of the
// manager, updating the various indices. The caller is assumed to hold the
// lock.
func (pm *basicManager) updatePodsInternal(pods ...*v1.Pod) {
	for _, pod := range pods {
		podFullName := kubecontainer.GetPodFullName(pod)
		// This logic relies on a static pod and its mirror to have the same name.
		// It is safe to type convert here due to the IsMirrorPod guard.
		if kubetypes.IsMirrorPod(pod) {
			mirrorPodUID := kubetypes.MirrorPodUID(pod.UID)
			pm.mirrorPodByUID[mirrorPodUID] = pod
			pm.mirrorPodByFullName[podFullName] = pod
			if p, ok := pm.podByFullName[podFullName]; ok {
				pm.translationByUID[mirrorPodUID] = kubetypes.ResolvedPodUID(p.UID)
			}
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