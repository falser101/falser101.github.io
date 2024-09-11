---
title: Kubernetes Scheduler源码分析
author: falser101
category:
  - k8s
tag:
  - Kubernetes Scheduler
---
# Kubernetes Scheduler

## 调度器

调度器是 Kubernetes 中的核心组件之一，负责将 Pod 调度到集群中的节点上。调度器的主要职责是根据一系列的调度策略和规则，选择一个最适合运行 Pod 的节点，并将 Pod 绑定到该节点上。

调度器的主要工作流程如下：

1. 调度器监听集群中的 Pod 创建事件，当有新的 Pod 需要调度时，调度器会收到通知。
2. 调度器会根据一系列的调度策略和规则，对集群中的节点进行评估，选择一个最适合运行 Pod 的节点。这些策略和规则可能包括节点的资源使用情况、节点标签、亲和性规则等。
3. 一旦选择了合适的节点，调度器会将 Pod 绑定到该节点上，并将绑定信息更新到 Kubernetes API 服务器中。
4. 节点上的 kubelet 组件会监听到 Pod 绑定事件，并开始在该节点上启动 Pod。
5. 调度器会继续监听集群中的 Pod 创建事件，并重复上述过程，直到所有的 Pod 都被调度到合适的节点上。

## 源码分析

### 调度器初始化

调度器在 Kubernetes 集群启动时会进行初始化，主要涉及到以下几个步骤：
1. 入口: `cmd/kube-scheduler/scheduler.go`
    ```go
    func main() {
        command := app.NewSchedulerCommand()
        code := cli.Run(command)
        os.Exit(code)
    }
    ```
2. 进入 `app.NewSchedulerCommand()`方法，该方法会创建一个 `*cobra.Command` 对象，并设置一些默认参数和命令行标志。主要逻辑在`runCommand`方法中
    ```go
    // runCommand runs the scheduler.
    func runCommand(cmd *cobra.Command, opts *options.Options, registryOptions ...Option) error {
        verflag.PrintAndExitIfRequested()
        fg := opts.ComponentGlobalsRegistry.FeatureGateFor(utilversion.DefaultKubeComponent)
        // Activate logging as soon as possible, after that
        // show flags with the final logging configuration.
        if err := logsapi.ValidateAndApply(opts.Logs, fg); err != nil {
            fmt.Fprintf(os.Stderr, "%v\n", err)
            os.Exit(1)
        }
        ......// 省略部分代码
        //
        cc, sched, err := Setup(ctx, opts, registryOptions...)
        if err != nil {
            return err
        }
        // add feature enablement metrics
        fg.(featuregate.MutableFeatureGate).AddMetrics()
        return Run(ctx, cc, sched)
    }
    ```
3. Setup方法主要是创建并返回一个完整的配置和调度器对象
    ```go
    // 创建一个完整的配置和调度器对象
    func Setup(ctx context.Context, opts *options.Options, outOfTreeRegistryOptions ...Option) (*schedulerserverconfig.CompletedConfig, *scheduler.Scheduler, error) {
        ......// 省略部分代码

        // Get the completed config
        cc := c.Complete()

        ......// 省略部分代码
        // Create the scheduler.
        sched, err := scheduler.New(ctx,
            cc.Client,
            cc.InformerFactory,
            cc.DynInformerFactory,
            recorderFactory,
            scheduler.WithComponentConfigVersion(cc.ComponentConfig.TypeMeta.APIVersion),
            scheduler.WithKubeConfig(cc.KubeConfig),
            scheduler.WithProfiles(cc.ComponentConfig.Profiles...),
            scheduler.WithPercentageOfNodesToScore(cc.ComponentConfig.PercentageOfNodesToScore),
            scheduler.WithFrameworkOutOfTreeRegistry(outOfTreeRegistry),
            scheduler.WithPodMaxBackoffSeconds(cc.ComponentConfig.PodMaxBackoffSeconds),
            scheduler.WithPodInitialBackoffSeconds(cc.ComponentConfig.PodInitialBackoffSeconds),
            scheduler.WithPodMaxInUnschedulablePodsDuration(cc.PodMaxInUnschedulablePodsDuration),
            scheduler.WithExtenders(cc.ComponentConfig.Extenders...),
            scheduler.WithParallelism(cc.ComponentConfig.Parallelism),
            scheduler.WithBuildFrameworkCapturer(func(profile kubeschedulerconfig.KubeSchedulerProfile) {
                // Profiles are processed during Framework instantiation to set default plugins and configurations. Capturing them for logging
                completedProfiles = append(completedProfiles, profile)
            }),
        )
        ......//省略

        return &cc, sched, nil
    }
    ```

5. 进入`scheduler.New()`方法，创建一个调度器对象，并返回。
    ```go
    // New returns a Scheduler
    func New(ctx context.Context,
        client clientset.Interface,
        informerFactory informers.SharedInformerFactory,
        dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
        recorderFactory profile.RecorderFactory,
        opts ...Option) (*Scheduler, error) {

        ......//省略

        // 注册内置插件
        registry := frameworkplugins.NewInTreeRegistry()
        // merge 内置插件和外部注册插件
        if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
            return nil, err
        }

        // 注册指标
        metrics.Register()

        // 构建外部扩展器
        extenders, err := buildExtenders(logger, options.extenders, options.profiles)
        if err != nil {
            return nil, fmt.Errorf("couldn't build extenders: %w", err)
        }

        // 实例化 podLister 负责监控 pod 变化
        podLister := informerFactory.Core().V1().Pods().Lister()
        // 实例化 nodeLister 负责监控 node 变化
        nodeLister := informerFactory.Core().V1().Nodes().Lister()

        // 创建 snapshot，snapshot 作为缓存存在
        snapshot := internalcache.NewEmptySnapshot()
        metricsRecorder := metrics.NewMetricsAsyncRecorder(1000, time.Second, stopEverything)
        // waitingPods holds all the pods that are in the scheduler and waiting in the permit stage
        // 保存调度程序中等待许可的所有 pod
        waitingPods := frameworkruntime.NewWaitingPodsMap()
        ......//省略部分代码

        // 创建 profiles，profiles 中存储的是调度器框架
        profiles, err := profile.NewMap(ctx, options.profiles, registry, recorderFactory,
            frameworkruntime.WithComponentConfigVersion(options.componentConfigVersion),
            frameworkruntime.WithClientSet(client),
            frameworkruntime.WithKubeConfig(options.kubeConfig),
            frameworkruntime.WithInformerFactory(informerFactory),
            frameworkruntime.WithResourceClaimCache(resourceClaimCache),
            frameworkruntime.WithSnapshotSharedLister(snapshot),
            frameworkruntime.WithCaptureProfile(frameworkruntime.CaptureProfile(options.frameworkCapturer)),
            frameworkruntime.WithParallelism(int(options.parallelism)),
            frameworkruntime.WithExtenders(extenders),
            frameworkruntime.WithMetricsRecorder(metricsRecorder),
            frameworkruntime.WithWaitingPods(waitingPods),
        )

        queueingHintsPerProfile[profileName], err = buildQueueingHintMap(ctx, profile.EnqueueExtensions())

        // 创建优先级队列 podQueue
        podQueue := internalqueue.NewSchedulingQueue(
            profiles[options.profiles[0].SchedulerName].QueueSortFunc(),
            informerFactory,
            internalqueue.WithPodInitialBackoffDuration(time.Duration(options.podInitialBackoffSeconds)*time.Second),
            internalqueue.WithPodMaxBackoffDuration(time.Duration(options.podMaxBackoffSeconds)*time.Second),
            internalqueue.WithPodLister(podLister),
            internalqueue.WithPodMaxInUnschedulablePodsDuration(options.podMaxInUnschedulablePodsDuration),
            internalqueue.WithPreEnqueuePluginMap(preEnqueuePluginMap),
            internalqueue.WithQueueingHintMapPerProfile(queueingHintsPerProfile),
            internalqueue.WithPluginMetricsSamplePercent(pluginMetricsSamplePercent),
            internalqueue.WithMetricsRecorder(*metricsRecorder),
        )

        for _, fwk := range profiles {
            fwk.SetPodNominator(podQueue)
        }

        // 创建调度器缓存 schedulerCache
        schedulerCache := internalcache.New(ctx, durationToExpireAssumedPod)

        // 实例化调度器 Scheduler
        sched := &Scheduler{
            Cache:                    schedulerCache,
            client:                   client,
            nodeInfoSnapshot:         snapshot,
            percentageOfNodesToScore: options.percentageOfNodesToScore,
            Extenders:                extenders,
            StopEverything:           stopEverything,
            SchedulingQueue:          podQueue,
            Profiles:                 profiles,
            logger:                   logger,
        }
        // 将队列的 Pop 方法赋值给 sched.NextPod
        sched.NextPod = podQueue.Pop
        sched.applyDefaultHandlers()

        if err = addAllEventHandlers(sched, informerFactory, dynInformerFactory, resourceClaimCache, unionedGVKs(queueingHintsPerProfile)); err != nil {
            return nil, fmt.Errorf("adding event handlers: %w", err)
        }

        return sched, nil
    }
    ```
    scheduler.New 创建了 snapshot, eventHandler, profiles(framework) 和 cache 等对象，结合着调度框架将它们关联起来会更清晰
    {% asset_img 2024-09-10-01.png This is an example image %}
    ![scheduler](/images/k8s/KubernetesScheduler源码分析/2024-09-10-01.png)

### 启动调度器
1. 继续进入`Run`方法，执行根据配置生成的调度器，只在发生错误或上下文完成时返回。
    ```go
    // Run executes the scheduler based on the given configuration. It only returns on error or when context is done.
    func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {

        // 启动事件处理器
        cc.EventBroadcaster.StartRecordingToSink(ctx.Done())
        defer cc.EventBroadcaster.Shutdown()

        // 设置健康检查
        var checks, readyzChecks []healthz.HealthChecker
        if cc.ComponentConfig.LeaderElection.LeaderElect {
            checks = append(checks, cc.LeaderElection.WatchDog)
            readyzChecks = append(readyzChecks, cc.LeaderElection.WatchDog)
        }
        readyzChecks = append(readyzChecks, healthz.NewShutdownHealthz(ctx.Done()))
        // 选举leader
        waitingForLeader := make(chan struct{})
        isLeader := func() bool {
            select {
            case _, ok := <-waitingForLeader:
                // if channel is closed, we are leading
                return !ok
            default:
                // channel is open, we are waiting for a leader
                return false
            }
        }
        ......//省略

        // 运行 informer
        startInformersAndWaitForSync := func(ctx context.Context) {
            // Start all informers.
            // 启动所有的informers
            cc.InformerFactory.Start(ctx.Done())
            // DynInformerFactory can be nil in tests.
            if cc.DynInformerFactory != nil {
                cc.DynInformerFactory.Start(ctx.Done())
            }

            // Wait for all caches to sync before scheduling.
            // 在调度之前等待所有的缓存同步
            cc.InformerFactory.WaitForCacheSync(ctx.Done())
            // DynInformerFactory can be nil in tests.
            if cc.DynInformerFactory != nil {
                cc.DynInformerFactory.WaitForCacheSync(ctx.Done())
            }

            // Wait for all handlers to sync (all items in the initial list delivered) before scheduling.
            // 在调度之前等待所有的处理器同步
            if err := sched.WaitForHandlersSync(ctx); err != nil {
                logger.Error(err, "waiting for handlers to sync")
            }

            close(handlerSyncReadyCh)
            logger.V(3).Info("Handlers synced")
        }
        if !cc.ComponentConfig.DelayCacheUntilActive || cc.LeaderElection == nil {
            startInformersAndWaitForSync(ctx)
        }
        // If leader election is enabled, runCommand via LeaderElector until done and exit.
        if cc.LeaderElection != nil {
        }

        // Leader election is disabled, so runCommand inline until done.
        close(waitingForLeader)
        sched.Run(ctx)
        return fmt.Errorf("finished without leader elect")
    }
    ```

    Run 函数内包含三部分处理：

    - 选举 leader 节点。如果是单节点，则跳过选举。
    - 运行 informer，负责监控 pod 和 node 变化。
    - 运行调度器

2. 进入 sched.Run 查看调度器是如何运行的。
    ```go
    // Run begins watching and scheduling. It starts scheduling and blocked until the context is done.
    // 开始监控和调度，它开始调度并阻塞直到上下文完成。
    func (sched *Scheduler) Run(ctx context.Context) {
        // 从队列中去需要调度的 pod
        sched.SchedulingQueue.Run(logger)

        // We need to start scheduleOne loop in a dedicated goroutine,
        // because scheduleOne function hangs on getting the next item
        // from the SchedulingQueue.
        // If there are no new pods to schedule, it will be hanging there
        // and if done in this goroutine it will be blocking closing
        // SchedulingQueue, in effect causing a deadlock on shutdown.
        // 调度 pod
        go wait.UntilWithContext(ctx, sched.ScheduleOne, 0)
    }
    ```
    sched.Run 主要做了两件事。从优先级队列中取用于调度的 pod，然后通过 sched.scheduleOne 调度该 pod。

    如何取出 pod 的呢？在 `sched.Run` 函数中，调用了 `sched.SchedulingQueue.Run(logger)`。
    ```go
    // Run starts the goroutine to pump from podBackoffQ to activeQ
    func (p *PriorityQueue) Run(logger klog.Logger) {
        go wait.Until(func() {
            p.flushBackoffQCompleted(logger)
        }, 1.0*time.Second, p.stop)
        go wait.Until(func() {
            p.flushUnschedulablePodsLeftover(logger)
        }, 30*time.Second, p.stop)
    }
    ```
    优先级队列由 ActiveQ，BackoffQ 和 UnschedulableQ 组成，其逻辑关系如下
    ![PriorityQueue.Run](/images/k8s/KubernetesScheduler源码分析/2024-09-10-02.png)

    在 PriorityQueue.Run 中启动两个 goroutine 分别运行 p.flushBackoffQCompleted 和 p.flushUnschedulablePodsLeftover 方法。p.flushBackoffQCompleted 将处于 BackOffQ 的 pod 移到 ActiveQ。p.flushUnschedulablePodsLeftover 将 UnschedulableQ 的 pod 移到 ActiveQ 或者 BackOffQ

    接着，进入 sched.scheduleOne 查看 pod 是怎么调度的。
    ```go
    func (sched *Scheduler) scheduleOne(ctx context.Context) {
        ...
        // 获取需要调度的 pod
        podInfo, err := sched.NextPod(logger)

        ...
        // 进入调度循环调度 pod
        scheduleResult, assumedPodInfo, status := sched.schedulingCycle(schedulingCycleCtx, state, fwk, podInfo, start, podsToActivate)
        if !status.IsSuccess() {
            sched.FailureHandler(schedulingCycleCtx, fwk, assumedPodInfo, status, scheduleResult.nominatingInfo, start)
            return
        }

        // 进入绑定循环绑定 pod
        go func() {
            ...
            status := sched.bindingCycle(bindingCycleCtx, state, fwk, scheduleResult, assumedPodInfo, start, podsToActivate)
            ...
        }()
    }
    ```

    `sched.scheduleOne` 主要包括三部分：获取需要调度的 pod，进入调度循环调度 pod 和进入绑定循环绑定 pod。其逻辑结构如下。
    ![sched.scheduleOne](/images/k8s/KubernetesScheduler源码分析//2024-09-10-03.png)

    sched.NextPod 获取需要调度的 pod
    ```go
    func (p *PriorityQueue) Pop(logger klog.Logger) (*framework.QueuedPodInfo, error) {
        ...
        for p.activeQ.Len() == 0 {
            if p.closed {
                logger.V(2).Info("Scheduling queue is closed")
                return nil, nil
            }

            // 如果 activeQ 没有 pod 的话，阻塞等待
            p.cond.Wait()
        }

        // 从 activeQ 中取 pod
        obj, err := p.activeQ.Pop()
        if err != nil {
            return nil, err
        }
        pInfo := obj.(*framework.QueuedPodInfo)
        ...

        return pInfo, nil
    }
    ```
    sched.NextPod 的逻辑主要是看 activeQ 队列中有没有 pod，如果有的话，取 pod 调度。如果没有的话，阻塞等待，直到 activeQ 中有 pod。

    sched.schedulingCycle 调度 pod
    ```go
    func (sched *Scheduler) schedulingCycle(
        ctx context.Context,
        state *framework.CycleState,
        fwk framework.Framework,
        podInfo *framework.QueuedPodInfo,
        start time.Time,
        podsToActivate *framework.PodsToActivate,
    ) (ScheduleResult, *framework.QueuedPodInfo, *framework.Status) {
        ...
        // 调度 Pod
        scheduleResult, err := sched.SchedulePod(ctx, fwk, state, pod)
        ...

        assumedPodInfo := podInfo.DeepCopy()
        assumedPod := assumedPodInfo.Pod
        err = sched.assume(logger, assumedPod, scheduleResult.SuggestedHost)
        ...

        // 运行 Reserve 插件的 Reserve 方法
        if sts := fwk.RunReservePluginsReserve(ctx, state, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
            ...
        }

        // 运行 Permit 插件
        runPermitStatus := fwk.RunPermitPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost)
        if !runPermitStatus.IsWait() && !runPermitStatus.IsSuccess() {
            ...
        }

        ...
        return scheduleResult, assumedPodInfo, nil
    }
    ```
    sched.schedulingCycle 包含几个步骤：sched.SchedulePod 调度 Pod，将调度的还未绑定的 Pod 作为 assumedPod 添加到缓存，运行 Reserve 插件和 Permit 插件。

    首先，看 sched.SchedulePod 是怎么调度 Pod 的。
    ```go
    // schedulePod tries to schedule the given pod to one of the nodes in the node list.
    // If it succeeds, it will return the name of the node.
    // If it fails, it will return a FitError with reasons.
    func (sched *Scheduler) schedulePod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
        // 获取合适的节点列表
        feasibleNodes, diagnosis, err := sched.findNodesThatFitPod(ctx, fwk, state, pod)
        if len(feasibleNodes) == 0 {
            return result, &framework.FitError{
                Pod:         pod,
                NumAllNodes: sched.nodeInfoSnapshot.NumNodes(),
                Diagnosis:   diagnosis,
            }
        }

        // When only one node after predicate, just use it.
        // 当只有一个节点满足条件时，直接返回
        if len(feasibleNodes) == 1 {
            return ScheduleResult{
                SuggestedHost:  feasibleNodes[0].Node().Name,
                EvaluatedNodes: 1 + diagnosis.NodeToStatus.Len(),
                FeasibleNodes:  1,
            }, nil
        }

        //否贼进行优先级排序
        priorityList, err := prioritizeNodes(ctx, sched.Extenders, fwk, state, pod, feasibleNodes)
        if err != nil {
            return result, err
        }

        host, _, err := selectHost(priorityList, numberOfHighestScoredNodesToReport)
        trace.Step("Prioritizing done")

        return ScheduleResult{
            SuggestedHost:  host,
            EvaluatedNodes: len(feasibleNodes) + diagnosis.NodeToStatus.Len(),
            FeasibleNodes:  len(feasibleNodes),
        }, err
    }
    ```

    在 sched.SchedulePod 中，sched.findNodesThatFitPod 为 Pod 寻找合适的节点。
    ```go
    // kubernetes/pkg/scheduler/schedule_one.go
    func (sched *Scheduler) findNodesThatFitPod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) ([]*framework.NodeInfo, framework.Diagnosis, error) {
        ...
        // 从 snapshot 中取所有节点
        allNodes, err := sched.nodeInfoSnapshot.NodeInfos().List()
        if err != nil {
            return nil, diagnosis, err
        }

        preRes, s := fwk.RunPreFilterPlugins(ctx, state, pod)
        if !s.IsSuccess() {
            ...
        }

        ...
        // 寻找 pod 可调用的节点
        feasibleNodes, err := sched.findNodesThatPassFilters(ctx, fwk, state, pod, &diagnosis, nodes)
        ...
    }

    // kubernetes/pkg/scheduler/schedule_one.go
    func (sched *Scheduler) findNodesThatPassFilters(
        ctx context.Context,
        fwk framework.Framework,
        state *framework.CycleState,
        pod *v1.Pod,
        diagnosis *framework.Diagnosis,
        nodes []*framework.NodeInfo) ([]*framework.NodeInfo, error) {
        ...
        checkNode := func(i int) {
            ...
            status := fwk.RunFilterPluginsWithNominatedPods(ctx, state, pod, nodeInfo)
        }
        ...
    }

    // kubernetes/pkg/scheduler/framework/runtime/framework.go
    func (f *frameworkImpl) RunFilterPluginsWithNominatedPods(ctx context.Context, state *framework.CycleState, pod *v1.Pod, info *framework.NodeInfo) *framework.Status {
        ...
        for i := 0; i < 2; i++ {
            ...
            // 运行 Filter 插件
            status = f.RunFilterPlugins(ctx, stateToUse, pod, nodeInfoToUse)
            if !status.IsSuccess() && !status.IsRejected() {
                return status
            }
        }

        return status
    }
    ```
    sched.findNodesThatFitPod 运行 Filter 插件获取可用的节点 feasibleNodes。接着，如果可用的节点只有一个，则返回调度结果。如果有多个节点则运行 priority 插件寻找最合适的节点作为调度节点。逻辑如下。
    ```go
    func (sched *Scheduler) schedulePod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
        ...
        feasibleNodes, diagnosis, err := sched.findNodesThatFitPod(ctx, fwk, state, pod)
        if err != nil {
            return result, err
        }

        ...
        if len(feasibleNodes) == 1 {
            return ScheduleResult{
                SuggestedHost:  feasibleNodes[0].Node().Name,
                EvaluatedNodes: 1 + len(diagnosis.NodeToStatusMap),
                FeasibleNodes:  1,
            }, nil
        }

        priorityList, err := sched.prioritizeNodes(ctx, fwk, state, pod, feasibleNodes)
        if err != nil {
            return result, err
        }

        host, _, err := selectHost(priorityList, numberOfHighestScoredNodesToReport)
        ...

        return ScheduleResult{
            SuggestedHost:  host,
            EvaluatedNodes: len(feasibleNodes) + len(diagnosis.NodeToStatusMap),
            FeasibleNodes:  len(feasibleNodes),
        }, err
    }
    ```

    获得调度结果 scheduleResult 后，在 sched.schedulingCycle 中的 sched.assume 将 assumePod 的 NodeName 更新为调度的节点 host，并且将 assumePod 添加到缓存中。缓存允许运行假定的操作，该操作将 Pod 临时存储在缓存中，使得 Pod 看起来像已经在快照的所有消费者的指定节点上运行那样。假定操作忽视了 kube-apiserver 和 Pod 实际更新的时间，从而增加调度器的吞吐量。

    ```go
    func (sched *Scheduler) assume(logger klog.Logger, assumed *v1.Pod, host string) error {
        assumed.Spec.NodeName = host

        if err := sched.Cache.AssumePod(logger, assumed); err != nil {
            logger.Error(err, "Scheduler cache AssumePod failed")
            return err
        }
        ...
        return nil
    }

    // kubernetes/pkg/scheduler/internal/cache/cache.go
    func (cache *cacheImpl) AssumePod(logger klog.Logger, pod *v1.Pod) error {
        ...
        return cache.addPod(logger, pod, true)
    }
    ```

    继续如 调度框架 所示，在 sched.schedulingCycle 中执行 Reserve 和 Permit 插件，插件执行通过后调度周期返回 Pod 的调度结果。

    接着，进入绑定周期。

    绑定周期是一个异步的 goroutine，负责将调度到节点的 Pod 发送给 kube-apiserver。进入绑定周期查看绑定逻辑的实现。
    ```go
    // kubernetes/pkg/scheduler/schedule_one.go
    func (sched *Scheduler) scheduleOne(ctx context.Context) {
        ...
        // 调度周期返回调度结果
        scheduleResult, assumedPodInfo, status := sched.schedulingCycle(schedulingCycleCtx, state, fwk, podInfo, start, podsToActivate)
        if !status.IsSuccess() {
            sched.FailureHandler(schedulingCycleCtx, fwk, assumedPodInfo, status, scheduleResult.nominatingInfo, start)
            return
        }

        // 绑定周期绑定调度结果
        go func() {
            ...
            status := sched.bindingCycle(bindingCycleCtx, state, fwk, scheduleResult, assumedPodInfo, start, podsToActivate)
            if !status.IsSuccess() {
                sched.handleBindingCycleError(bindingCycleCtx, state, fwk, assumedPodInfo, start, scheduleResult, status)
                return
            }
            ...
        }()
    }

    func (sched *Scheduler) bindingCycle(
        ctx context.Context,
        state *framework.CycleState,
        fwk framework.Framework,
        scheduleResult ScheduleResult,
        assumedPodInfo *framework.QueuedPodInfo,
        start time.Time,
        podsToActivate *framework.PodsToActivate) *framework.Status {
        ...
        // 运行 Permit 插件
        if status := fwk.WaitOnPermit(ctx, assumedPod); !status.IsSuccess() {
            ...
        }

        // 运行 PreBind 插件
        if status := fwk.RunPreBindPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost); !status.IsSuccess() {
            ...
        }

        // 运行 Bind 插件
        if status := sched.bind(ctx, fwk, assumedPod, scheduleResult.SuggestedHost, state); !status.IsSuccess() {
            return status
        }

        // 运行 PostBind 插件
        fwk.RunPostBindPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost)
        ...
    }
    ```

    可以看到，绑定周期运行一系列插件进行绑定，进入 Bind 插件查看绑定的行为。

    ```go
    func (sched *Scheduler) bind(ctx context.Context, fwk framework.Framework, assumed *v1.Pod, targetNode string, state *framework.CycleState) (status *framework.Status) {
        ...
        return fwk.RunBindPlugins(ctx, state, assumed, targetNode)
    }

    // kubernetes/pkg/scheduler/framework/runtime/framework.go
    func (f *frameworkImpl) RunBindPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (status *framework.Status) {
        ...
        for _, pl := range f.bindPlugins {
            status = f.runBindPlugin(ctx, pl, state, pod, nodeName)
            if status.IsSkip() {
                continue
            }
            ...
        }
        ...
    }

    func (f *frameworkImpl) runBindPlugin(ctx context.Context, bp framework.BindPlugin, state *framework.CycleState, pod *v1.Pod, nodeName string) *framework.Status {
        ...
        status := bp.Bind(ctx, state, pod, nodeName)
        ...
        return status
    }

    // kubernetes/pkg/scheduler/plugins/defaultbinder/default_binder.go
    func (b DefaultBinder) Bind(ctx context.Context, state *framework.CycleState, p *v1.Pod, nodeName string) *framework.Status {
        ...
        logger.V(3).Info("Attempting to bind pod to node", "pod", klog.KObj(p), "node", klog.KRef("", nodeName))
        binding := &v1.Binding{
            ObjectMeta: metav1.ObjectMeta{Namespace: p.Namespace, Name: p.Name, UID: p.UID},
            Target:     v1.ObjectReference{Kind: "Node", Name: nodeName},
        }
        err := b.handle.ClientSet().CoreV1().Pods(binding.Namespace).Bind(ctx, binding, metav1.CreateOptions{})
        if err != nil {
            return framework.AsStatus(err)
        }
        return nil
    }
    ```

    在 Bind 插件中调用 ClientSet 的 Bind 方法将 Pod 和 node 绑定的结果发给 kube-apiserver，实现绑定操作。

