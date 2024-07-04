# kube-controller-manager
## 初始化
与其他组件一样，先会初始化配置，加载已经有的内置参数，并通过启动的命令行参数进行参数覆盖。
还包含了所有已知controller的注册、去重操作。同时还有连接apiserver的bearToken生成。略过不看
我们直接来看run方法
```go
func Run(ctx context.Context, c *config.CompletedConfig) error {
	logger := klog.FromContext(ctx)
	stopCh := ctx.Done()

	// To help debugging, immediately log version
	logger.Info("Starting", "version", version.Get())

	logger.Info("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

    // 事件相关
	// Start events processing pipeline.
	c.EventBroadcaster.StartStructuredLogging(0)
	c.EventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: c.Client.CoreV1().Events("")})
	defer c.EventBroadcaster.Shutdown()

	if cfgz, err := configz.New(ConfigzName); err == nil {
		cfgz.Set(c.ComponentConfig)
	} else {
		logger.Error(err, "Unable to register configz")
	}

	// Setup any healthz checks we will want to use.
	var checks []healthz.HealthChecker
	var electionChecker *leaderelection.HealthzAdaptor
    // 如果controllerManager开启了节点选举机制，则需要在健康检测中也加入对leader节点的健康状态检测
    // 具体逻辑是如果该节点不是leader，则直接跳过。如果是leader，检测成为leader的持续时间是否大于租约时间+过期时间，如果大于，说明没有参与到选举中，可能出现脑裂情况，返回错误
	if c.ComponentConfig.Generic.LeaderElection.LeaderElect {
		electionChecker = leaderelection.NewLeaderHealthzAdaptor(time.Second * 20)
		checks = append(checks, electionChecker)
	}
	healthzHandler := controllerhealthz.NewMutableHealthzHandler(checks...)

	// Start the controller manager HTTP server
	// unsecuredMux is the handler for these controller *after* authn/authz filters have been applied
	var unsecuredMux *mux.PathRecorderMux
	if c.SecureServing != nil {
        // 仅注册/metrics和/healthz两种路由
		unsecuredMux = genericcontrollermanager.NewBaseHandler(&c.ComponentConfig.Generic.Debugging, healthzHandler)
		slis.SLIMetricsWithReset{}.Install(unsecuredMux)

        // 如果启用了安全认证，则需要在处理请求时加上认证和鉴权的中间件，类似于各种web框架，只有前面的所有拦截器都通过，才处理真正的请求。
		handler := genericcontrollermanager.BuildHandlerChain(unsecuredMux, &c.Authorization, &c.Authentication)
		// TODO: handle stoppedCh and listenerStoppedCh returned by c.SecureServing.Serve
		if _, _, err := c.SecureServing.Serve(handler, 0, stopCh); err != nil {
			return err
		}
	}
    // 创建client用于从apiserver获取数据。如果使用serviceAccount，则使用dynamicClient，否则的话，直接使用kubeconfig进行连接
	clientBuilder, rootClientBuilder := createClientBuilders(logger, c)
    // 初始化saTokenControllerDescriptor，看下面newServiceAccountTokenControllerDescriptor
	saTokenControllerDescriptor := newServiceAccountTokenControllerDescriptor(rootClientBuilder)

	run := func(ctx context.Context, controllerDescriptors map[string]*ControllerDescriptor) {
		controllerContext, err := CreateControllerContext(ctx, c, rootClientBuilder, clientBuilder)
		if err != nil {
			logger.Error(err, "Error building controller context")
			klog.FlushAndExit(klog.ExitFlushTimeout, 1)
		}

		if err := StartControllers(ctx, controllerContext, controllerDescriptors, unsecuredMux, healthzHandler); err != nil {
			logger.Error(err, "Error starting controllers")
			klog.FlushAndExit(klog.ExitFlushTimeout, 1)
		}

		controllerContext.InformerFactory.Start(stopCh)
		controllerContext.ObjectOrMetadataInformerFactory.Start(stopCh)
		close(controllerContext.InformersStarted)

		<-ctx.Done()
	}

	// No leader election, run directly
	if !c.ComponentConfig.Generic.LeaderElection.LeaderElect {
		controllerDescriptors := NewControllerDescriptors()
		controllerDescriptors[names.ServiceAccountTokenController] = saTokenControllerDescriptor
		run(ctx, controllerDescriptors)
		return nil
	}

	id, err := os.Hostname()
	if err != nil {
		return err
	}

	// add a uniquifier so that two processes on the same host don't accidentally both become active
	id = id + "_" + string(uuid.NewUUID())

	// leaderMigrator will be non-nil if and only if Leader Migration is enabled.
	var leaderMigrator *leadermigration.LeaderMigrator = nil

	// If leader migration is enabled, create the LeaderMigrator and prepare for migration
	if leadermigration.Enabled(&c.ComponentConfig.Generic) {
		logger.Info("starting leader migration")

		leaderMigrator = leadermigration.NewLeaderMigrator(&c.ComponentConfig.Generic.LeaderMigration,
			"kube-controller-manager")

		// startSATokenControllerInit is the original InitFunc.
		startSATokenControllerInit := saTokenControllerDescriptor.GetInitFunc()

		// Wrap saTokenControllerDescriptor to signal readiness for migration after starting
		//  the controller.
		saTokenControllerDescriptor.initFunc = func(ctx context.Context, controllerContext ControllerContext, controllerName string) (controller.Interface, bool, error) {
			defer close(leaderMigrator.MigrationReady)
			return startSATokenControllerInit(ctx, controllerContext, controllerName)
		}
	}

	// Start the main lock
	go leaderElectAndRun(ctx, c, id, electionChecker,
		c.ComponentConfig.Generic.LeaderElection.ResourceLock,
		c.ComponentConfig.Generic.LeaderElection.ResourceName,
		leaderelection.LeaderCallbacks{
			OnStartedLeading: func(ctx context.Context) {
				controllerDescriptors := NewControllerDescriptors()
				if leaderMigrator != nil {
					// If leader migration is enabled, we should start only non-migrated controllers
					//  for the main lock.
					controllerDescriptors = filteredControllerDescriptors(controllerDescriptors, leaderMigrator.FilterFunc, leadermigration.ControllerNonMigrated)
					logger.Info("leader migration: starting main controllers.")
				}
				controllerDescriptors[names.ServiceAccountTokenController] = saTokenControllerDescriptor
				run(ctx, controllerDescriptors)
			},
			OnStoppedLeading: func() {
				logger.Error(nil, "leaderelection lost")
				klog.FlushAndExit(klog.ExitFlushTimeout, 1)
			},
		})

	// If Leader Migration is enabled, proceed to attempt the migration lock.
	if leaderMigrator != nil {
		// Wait for Service Account Token Controller to start before acquiring the migration lock.
		// At this point, the main lock must have already been acquired, or the KCM process already exited.
		// We wait for the main lock before acquiring the migration lock to prevent the situation
		//  where KCM instance A holds the main lock while KCM instance B holds the migration lock.
		<-leaderMigrator.MigrationReady

		// Start the migration lock.
		go leaderElectAndRun(ctx, c, id, electionChecker,
			c.ComponentConfig.Generic.LeaderMigration.ResourceLock,
			c.ComponentConfig.Generic.LeaderMigration.LeaderName,
			leaderelection.LeaderCallbacks{
				OnStartedLeading: func(ctx context.Context) {
					logger.Info("leader migration: starting migrated controllers.")
					controllerDescriptors := NewControllerDescriptors()
					controllerDescriptors = filteredControllerDescriptors(controllerDescriptors, leaderMigrator.FilterFunc, leadermigration.ControllerMigrated)
					// DO NOT start saTokenController under migration lock
					delete(controllerDescriptors, names.ServiceAccountTokenController)
					run(ctx, controllerDescriptors)
				},
				OnStoppedLeading: func() {
					logger.Error(nil, "migration leaderelection lost")
					klog.FlushAndExit(klog.ExitFlushTimeout, 1)
				},
			})
	}

	<-stopCh
	return nil
}
```
### newServiceAccountTokenControllerDescriptor
```go
func newServiceAccountTokenControllerDescriptor(rootClientBuilder clientbuilder.ControllerClientBuilder) *ControllerDescriptor {
	return &ControllerDescriptor{
		name:    names.ServiceAccountTokenController,
		aliases: []string{"serviceaccount-token"},
		initFunc: func(ctx context.Context, controllerContext ControllerContext, controllerName string) (controller.Interface, bool, error) {
            // 这个方法应该会在controller调用Run的时候启动
			return startServiceAccountTokenController(ctx, controllerContext, controllerName, rootClientBuilder)
		},
		// will make sure it runs first before other controllers
		requiresSpecialHandling: true,
	}
}
func startServiceAccountTokenController(ctx context.Context, controllerContext ControllerContext, controllerName string, rootClientBuilder clientbuilder.ControllerClientBuilder) (controller.Interface, bool, error) {
    ...
    // 创建informer监听serviceAccount和secret资源
	tokenController, err := serviceaccountcontroller.NewTokensController(
		controllerContext.InformerFactory.Core().V1().ServiceAccounts(),
		controllerContext.InformerFactory.Core().V1().Secrets(),
		rootClientBuilder.ClientOrDie("tokens-controller"),
		serviceaccountcontroller.TokensControllerOptions{
			TokenGenerator: tokenGenerator,
			RootCA:         rootCA,
		},
	)
	if err != nil {
		return nil, true, fmt.Errorf("error creating Tokens controller: %v", err)
	}
	go tokenController.Run(ctx, int(controllerContext.ComponentConfig.SAController.ConcurrentSATokenSyncs))

	// start the first set of informers now so that other controllers can start
	controllerContext.InformerFactory.Start(ctx.Done())

	return nil, true, nil
}

func NewTokensController(serviceAccounts informers.ServiceAccountInformer, secrets informers.SecretInformer, cl clientset.Interface, options TokensControllerOptions) (*TokensController, error) {
	maxRetries := options.MaxRetries
	if maxRetries == 0 {
		maxRetries = 10
	}

	e := &TokensController{
		client: cl,
        // jwtToken生成器
		token:  options.TokenGenerator,
		rootCA: options.RootCA,
        // 同步队列
		syncServiceAccountQueue: workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "serviceaccount_tokens_service"),
		syncSecretQueue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "serviceaccount_tokens_secret"),

		maxRetries: maxRetries,
	}

    // 下面就是informer的常规调用方法，当监听到serviceAccount或secret发生增删改操作时触发对应的handler
	e.serviceAccounts = serviceAccounts.Lister()
	e.serviceAccountSynced = serviceAccounts.Informer().HasSynced
	serviceAccounts.Informer().AddEventHandlerWithResyncPeriod(
		cache.ResourceEventHandlerFuncs{
            // 队列新增对象
			AddFunc:    e.queueServiceAccountSync,
            // 也是在队列中增加待同步对象
			UpdateFunc: e.queueServiceAccountUpdateSync,
			DeleteFunc: e.queueServiceAccountSync,
		},
		options.ServiceAccountResync,
	)
    // secret下同
    ...

	return e, nil
}

func (e *TokensController) Run(ctx context.Context, workers int) {
	// 关闭队列、处理异常错误
	defer utilruntime.HandleCrash()
	defer e.syncServiceAccountQueue.ShutDown()
	defer e.syncSecretQueue.ShutDown()

    // 立即触发同步，只有serviceAccount和secret都同步完成，才往下走，否则直接返回
	if !cache.WaitForNamedCacheSync("tokens", ctx.Done(), e.serviceAccountSynced, e.secretSynced) {
		return
	}

	logger := klog.FromContext(ctx)
	logger.V(5).Info("Starting workers")
	for i := 0; i < workers; i++ {
        // 开启多个goroutine执行对应的同步任务，直到上下文取消或退出
		go wait.UntilWithContext(ctx, e.syncServiceAccount, 0)
		go wait.UntilWithContext(ctx, e.syncSecret, 0)
	}
	<-ctx.Done()
	logger.V(1).Info("Shutting down")
}

func (e *TokensController) syncServiceAccount(ctx context.Context) {
	logger := klog.FromContext(ctx)
    // 从队列中取出一个serviceAccount对象用于同步
	key, quit := e.syncServiceAccountQueue.Get()
	if quit {
		return
	}
	defer e.syncServiceAccountQueue.Done(key)

	retry := false
	defer func() {
        // 默认不重试，直接丢弃
		e.retryOrForget(logger, e.syncServiceAccountQueue, key, retry)
	}()
    // 解析类型是否正确
	saInfo, err := parseServiceAccountKey(key)
	if err != nil {
		logger.Error(err, "Parsing service account key")
		return
	}
    // 从informer缓存中查找是否存在该serviceAccount，如果没有直接返回
	sa, err := e.getServiceAccount(saInfo.namespace, saInfo.name, saInfo.uid, false)
	switch {
	case err != nil:
		logger.Error(err, "Getting service account")
		retry = true
	case sa == nil:
        // 没有查到，则从apiserver中调用删除操作
		// service account no longer exists, so delete related tokens
		logger.V(4).Info("Service account deleted, removing tokens", "namespace", saInfo.namespace, "serviceaccount", saInfo.name)
		sa = &v1.ServiceAccount{ObjectMeta: metav1.ObjectMeta{Namespace: saInfo.namespace, Name: saInfo.name, UID: saInfo.uid}}
		retry, err = e.deleteTokens(sa)
		if err != nil {
			logger.Error(err, "Error deleting serviceaccount tokens", "namespace", saInfo.namespace, "serviceaccount", saInfo.name)
		}
	}
}
```