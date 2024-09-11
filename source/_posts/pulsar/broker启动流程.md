---
title: pulsar broker组件启动流程
author: falser101
date: 2024-01-10
headerDepth: 3
category:
  - pulsar
tag:
  - pulsar
---

# Broker组件启动流程
在收发消息前，我们得启动Broker服务，本章将介绍Broker在启动阶段都做了什么。各个模块之前的关系，服务启动不同模块的初始化顺序。

## 服务启动

使用bin目录下名为pulsar的CLI工具，执行命令时传入broker参数，会调用`PulsarBrokerStarter`这个类来启动Pulsar Broker服务。Broker总体启动流程如下图所示:

![pulsar-broker-startup](/imgs/pulsar/broker-startup/总体启动流程.png)

1. 加载配置文件。Broker启动时会读取conf/broker.conf文件，并将其中的配置信息全部转化为Key/Value的形式，通过反射创建一个`ServiceConfigration`对象。并且把值设置到这个对象的属性中。

2. 注册ShutdownHook和OOM监听器。ShutdownHook在服务关闭时执行清理💰，OOMListener在内存溢出时打印异常信息。

3. 启动`BookieStatsProvider`服务,`BookieStatsProvider`是Bookie服务的统计信息提供者，提供Bookie服务的统计信息。这一步通常只在Standalone模式下执行。

4. 启动BookieServer,让Bookie和Broker一起启动，主要用于开发测试

5. 启动`PulsarService`服务，`PulsarService`是启动Broker的主入口，内部还会启动一个`BrokerService`，这两个Service的分工不同，`PulsarService`范围更大，包括负载管理、缓存、Schema、Admin相关的Web服务等，属于管理流。而`BrokerService`则专注于消息的收发，创建`Netty EventLoopGroup`、创建内置调度器等，属于数据流。

> 什么是管理流和数据流？
>
> 增删改查Broker的配置，或者一个Topic，通过Pulsar-Admin来操作，这些操作都是管理流。
> 收发消息，这些操作都是数据流。执行操作之前，需要通过PulsarClient来创建Producer和Consumer，然后才能执行收发消息的操作。

主要代码如下：
```java
public static void main(String[] args) throws Exception {
    // ...
    BrokerStarter starter = new BrokerStarter(args);
    Runtime.getRuntime().addShutdownHook(...);

    PulsarByteBufAllocator.registerOOMListener(oomException -> {...});

    try {
        starter.start();
    }
    ...
}

# 加载配置文件
private static ServiceConfiguration loadConfig(String configFile) throws Exception {
    try (InputStream inputStream = new FileInputStream(configFile)) {
        ServiceConfiguration config = create(inputStream, ServiceConfiguration.class);
        // it validates provided configuration is completed
        isComplete(config);
        return config;
    }
}

# 启动服务
public void start() throws Exception {
    if (bookieStatsProvider != null) {
        bookieStatsProvider.start(bookieConfig);
        log.info("started bookieStatsProvider.");
    }
    if (bookieServer != null) {
        bookieStartFuture = ComponentStarter.startComponent(bookieServer);
        log.info("started bookieServer.");
    }
    if (autoRecoveryMain != null) {
        autoRecoveryMain.start();
        log.info("started bookie autoRecoveryMain.");
    }

    pulsarService.start();
    log.info("PulsarService started.");
}
```

## PulsarService.java

主要代码如下（省略了一些不重要的代码）：
```java
public void start() throws PulsarServerException {
    mutex.lock();
    try {
        ......
        localMetadataSynchronizer = StringUtils.isNotBlank(config.getMetadataSyncEventTopic())
                ? new PulsarMetadataEventSynchronizer(this, config.getMetadataSyncEventTopic())
                : null;
        localMetadataStore = createLocalMetadataStore(localMetadataSynchronizer);
        localMetadataStore.registerSessionListener(this::handleMetadataSessionEvent);

        // 创建coordinationService，用于分布式锁、Broker选主等
        coordinationService = new CoordinationServiceImpl(localMetadataStore);

        // 创建pulsarResources，用于元数据管理，元数据包括本地元数据和配置文件元数据，保存在zookeeper中
        // 主要实现方式为调用配置中心存储对于资源的元数据
        pulsarResources = newPulsarResources();

        // 创建orderedExecutor线程池，后续用于ZK的操作，拆分bundle等
        orderedExecutor = newOrderedExecutor();

        // 创建protocolHandlers，这个服务用于管理pulsar的扩展点，pulsar中可以通过*.nar文件实现自定义扩展点。
        // Pulsar启动时会默认尝试扫描./protocols目录或系统环境变量java.io.tmpdir里设置的路径下的所有nar文件。
        // 如KOP、AOP等都是通过这个扩展点实现的
        protocolHandlers = ProtocolHandlers.load(config);
        protocolHandlers.initialize(config);

        // Now we are ready to start services
        this.bkClientFactory = newBookKeeperClientFactory();

        // 创建ManagedLedgerClientFactory，用于管理ManagedLedger的客户端。
        // Pulsar是计算和存储分离的架构，managedLedger是存储的抽象，ManagedLedgerClient用来请求BookKeeper保存数据
        managedLedgerClientFactory = newManagedLedgerClientFactory();

        // 创建BrokerService, 数据流相关的服务都在这里启动
        this.brokerService = newBrokerService(this);

        // 创建loadManager，用于管理broker的负载，不管有没有打开自动负载管理都会创建
        // 默认管理器是org.apache.pulsar.broker.loadbalance.impl.ModularLoadManagerImpl
        this.loadManager.set(LoadManager.create(this));

        // needs load management service and before start broker service,
        this.startNamespaceService();

        schemaStorage = createAndStartSchemaStorage();
        // 用于管理Topic的Schema，如Schema.Type.JSON
        schemaRegistryService = SchemaRegistryService.create(
                schemaStorage, config.getSchemaRegistryCompatibilityCheckers(), this.executor);

        OffloadPoliciesImpl defaultOffloadPolicies =
                OffloadPoliciesImpl.create(this.getConfiguration().getProperties());

        OrderedScheduler offloaderScheduler = getOffloaderScheduler(defaultOffloadPolicies);

        offloaderStats = LedgerOffloaderStats.create(config.isExposeManagedLedgerMetricsInPrometheus(),
                exposeTopicMetrics, offloaderScheduler, interval);

        // 创建OffloaderManager，当有大量的消息堆积，这些消息要求保留很长时间，我们不想让这些冷数据占用线上服务的硬盘空间。
        // 这些冷数据可以使用OffloaderManager移动到其他磁盘上，以节省空间。
        this.defaultOffloader = createManagedLedgerOffloader(defaultOffloadPolicies);

        setBrokerInterceptor(newBrokerInterceptor());
        // use getter to support mocking getBrokerInterceptor method in tests
        BrokerInterceptor interceptor = getBrokerInterceptor();
        if (interceptor != null) {
            brokerService.setInterceptor(interceptor);
            interceptor.initialize(this);
        }
        // 启动brokerService
        brokerService.start();

        // Load additional servlets
        this.brokerAdditionalServlets = AdditionalServlets.load(config);

        // 创建管理流WebService，并启动
        this.webService = new WebService(this);
        createMetricsServlet();
        this.addWebServerHandlers(webService, metricsServlet, this.config);
        this.webService.start();

        ......

        // 启动Leader选举服务
        startLeaderElectionService();

        // 启动LoadManagementService
        this.startLoadManagementService();

        // Initialize namespace service, after service url assigned. Should init zk and refresh self owner info.
        this.nsService.initialize();

        // 主题策略服务，让用户可以设置主题级别的策略
        if (config.isTopicLevelPoliciesEnabled() && config.isSystemTopicEnabled()) {
            this.topicPoliciesService = new SystemTopicBasedTopicPoliciesService(this);
        }

        this.topicPoliciesService.start();

        // Register heartbeat and bootstrap namespaces.
        this.nsService.registerBootstrapNamespaces();

        // 事务相关服务
        if (config.isTransactionCoordinatorEnabled()) {
            MLTransactionMetadataStoreProvider.initBufferedWriterMetrics(getAdvertisedAddress());
            MLPendingAckStoreProvider.initBufferedWriterMetrics(getAdvertisedAddress());

            this.transactionBufferSnapshotServiceFactory = new TransactionBufferSnapshotServiceFactory(getClient());

            this.transactionTimer =
                    new HashedWheelTimer(new DefaultThreadFactory("pulsar-transaction-timer"));
            transactionBufferClient = TransactionBufferClientImpl.create(this, transactionTimer,
                    config.getTransactionBufferClientMaxConcurrentRequests(),
                    config.getTransactionBufferClientOperationTimeoutInMills());

            transactionMetadataStoreService = new TransactionMetadataStoreService(TransactionMetadataStoreProvider
                    .newProvider(config.getTransactionMetadataStoreProviderClassName()), this,
                    transactionBufferClient, transactionTimer);

            transactionBufferProvider = TransactionBufferProvider
                    .newProvider(config.getTransactionBufferProviderClassName());
            transactionPendingAckStoreProvider = TransactionPendingAckStoreProvider
                    .newProvider(config.getTransactionPendingAckStoreProviderClassName());
        }

        // 初始化metricsGenerator，用于将生成Broker级别的metrics暴露给Prometheus
        this.metricsGenerator = new MetricsGenerator(this);

        // Initialize the message protocol handlers.
        // start the protocol handlers only after the broker is ready,
        // so that the protocol handlers can access broker service properly.
        this.protocolHandlers.start(brokerService);
        Map<String, Map<InetSocketAddress, ChannelInitializer<SocketChannel>>> protocolHandlerChannelInitializers =
            this.protocolHandlers.newChannelInitializers();
        this.brokerService.startProtocolHandlers(protocolHandlerChannelInitializers);

        acquireSLANamespace();

        // 创建并启动function worker服务
        this.startWorkerService(brokerService.getAuthenticationService(), brokerService.getAuthorizationService());

        // 创建并启动packages management服务,用于管理Function的用户上传的代码包
        if (config.isEnablePackagesManagement()) {
            this.startPackagesManagementService();
        }

        ......
    }
}
```

## BrokerService.java

```java
public BrokerService(PulsarService pulsar, EventLoopGroup eventLoopGroup) throws Exception {
    ......
    // 创建topicOrderedExecutor，这是一个线程池，由bookkeeper封装，保证同一个Topic名，
    // 永远使用同一个线程来执行，从而保证同一个topic的操作是有序的
    this.topicOrderedExecutor = OrderedExecutor.newBuilder()
                .numThreads(pulsar.getConfiguration().getTopicOrderedExecutorThreadNum())
                .name("broker-topic-workers").build();

    // 使用Netty创建acceptorGroup处理I/O事件
    this.acceptorGroup = EventLoopUtil.newEventLoopGroup(
                pulsar.getConfiguration().getNumAcceptorThreads(), false, acceptorThreadFactory);

    // 使用Netty创建workerGroup处理业务逻辑
    this.workerGroup = eventLoopGroup;
    ......

    // Pulsar权限校验，支持Produce，Consume、Function三种权限
    this.authorizationService = new AuthorizationService(
                pulsar.getConfiguration(), pulsar().getPulsarResources());
    // 创建各种监视器
    // 定期检查长期未使用的主题并删除
    this.inactivityMonitor = OrderedScheduler.newSchedulerBuilder()
                .name("pulsar-inactivity-monitor")
                .numThreads(1)
                .build();
    // 定期检查过期消息并删除
    this.messageExpiryMonitor = ...
    // 定期压缩主题
    this.compactionMonitor = ...
    this.consumedLedgersMonitor = ...

    this.backlogQuotaManager = new BacklogQuotaManager(pulsar);
    // backlog检查器
    this.backlogQuotaChecker = ...
    // 认证服务
    this.authenticationService = new AuthenticationService(pulsar.getConfiguration());
    ......
    // 创建动态配置的监听器
    updateConfigurationAndRegisterListeners();

    // 一些信号量
    this.lookupRequestSemaphore = new AtomicReference<Semaphore>(
            new Semaphore(pulsar.getConfiguration().getMaxConcurrentLookupRequest(), false));
    this.topicLoadRequestSemaphore = new AtomicReference<Semaphore>(
            new Semaphore(pulsar.getConfiguration().getMaxConcurrentTopicLoadRequest(), false));
    if (pulsar.getConfiguration().getMaxUnackedMessagesPerBroker() > 0
            && pulsar.getConfiguration().getMaxUnackedMessagesPerSubscriptionOnBrokerBlocked() > 0.0) {
        // 最大未确认消息数
        this.maxUnackedMessages = pulsar.getConfiguration().getMaxUnackedMessagesPerBroker();
        this.maxUnackedMsgsPerDispatcher = (int) (maxUnackedMessages
                * pulsar.getConfiguration().getMaxUnackedMessagesPerSubscriptionOnBrokerBlocked());
    } else {
        this.maxUnackedMessages = 0;
        this.maxUnackedMsgsPerDispatcher = 0;
    }
    // Pulsar的延迟消息的实现类会通过这个工厂创建出来
    this.delayedDeliveryTrackerFactory = DelayedDeliveryTrackerLoader
        .loadDelayedDeliveryTrackerFactory(pulsar);
    // Entry元数据的拦截器，用于用户自定义扩展
    this.brokerEntryMetadataInterceptors = BrokerEntryMetadataUtils
        .loadBrokerEntryMetadataInterceptors(pulsar.getConfiguration().getBrokerEntryMetadataInterceptors(),
                BrokerService.class.getClassLoader());
```

### 动态配置
在启动Broker Service前会遍历ServiceConfiguration类中的所有成员变量（包括私有成员变量）

如果变量不是null，并且它具有FieldContext注解，那么设置其可访问性（以便在后续步骤中访问其值）

如果变量具有dynamic()属性，使用new ConfigField(field)创建一个新的ConfigField对象，并且将该对象添加到dynamicConfigurationMap

```java
    private static ConcurrentOpenHashMap<String, ConfigField> prepareDynamicConfigurationMap() {
        ConcurrentOpenHashMap<String, ConfigField> dynamicConfigurationMap =
                ConcurrentOpenHashMap.<String, ConfigField>newBuilder().build();
        for (Field field : ServiceConfiguration.class.getDeclaredFields()) {
            if (field != null && field.isAnnotationPresent(FieldContext.class)) {
                field.setAccessible(true);
                if (field.getAnnotation(FieldContext.class).dynamic()) {
                    dynamicConfigurationMap.put(field.getName(), new ConfigField(field));
                }
            }
        }
        return dynamicConfigurationMap;
    }
```

然后通过下面的方法进行注册
```java
    public <T> void registerConfigurationListener(String configKey, Consumer<T> listener) {
        validateConfigKey(configKey);
        configRegisteredListeners.put(configKey, listener);
    }
```

示例：
```java
    registerConfigurationListener("maxPublishRatePerTopicInMessages",
        maxPublishRatePerTopicInMessages -> updateMaxPublishRatePerTopicInMessages()
    );

    private void updateMaxPublishRatePerTopicInMessages() {
        this.pulsar().getExecutor().execute(() ->
            forEachTopic(topic -> {
                if (topic instanceof AbstractTopic) {
                    ((AbstractTopic) topic).updateBrokerPublishRate();
                    ((AbstractTopic) topic).updatePublishRateLimiter();
                }
            }));
    }
```