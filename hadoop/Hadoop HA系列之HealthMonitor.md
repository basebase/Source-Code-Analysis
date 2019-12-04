###  Hadoop HA HealthMonitor


#### 说在前面
HA系列全部围绕IBM文章开始到结束<br />
https://www.ibm.com/developerworks/cn/opensource/os-cn-hadoop-name-node/index.html

#### HealthMonitor类中的一些状态变量
```java

@InterfaceAudience.Private
  public enum State {
    /**
     * The health monitor is still starting up.
     */
    INITIALIZING,  // 初始化状态, 还没有开始进行健康状况监测

    /**
     * The service is not responding to health check RPCs.
     */
    SERVICE_NOT_RESPONDING, // 调用NameNode的monitorHealth方法无响应或者响应超时

    /**
     * The service is connected and healthy.
     */
    SERVICE_HEALTHY, // NameNode状态正常

    /**
     * The service is running but unhealthy.
     */
    SERVICE_UNHEALTHY, // NameNode还在运行, 但是monitorHealth方法返回状态不正常, 磁盘资源不足等其他情况.

    /**
     * The health monitor itself failed unrecoverably and can
     * no longer provide accurate information.
     */
    HEALTH_MONITOR_FAILED; // HealthMonitor 自己在运行过程中发生了异常, 不能继续监测NameNode的健康状况, 会导致ZKFC进程退出
  }
```
HealthMonitor.State在状态检测中起主要作用, 在HealthMonitor.State发生变化的时候, HealthMonitor会回调ZKFailoverController
的相应方法来进行处理, 这里就不描述了。



#### HealthMonitor一探究竟


```java
// TestHealthMonitor.java

// 在监控之前, 先把相关参数配置好来
@Before
  public void setupHM() throws InterruptedException, IOException {
    Configuration conf = new Configuration();
    // 客户端连接重试次数
    conf.setInt(CommonConfigurationKeys.IPC_CLIENT_CONNECT_MAX_RETRIES_KEY, 1);
    // HA功能的健康监控连接间隔(毫秒)
    conf.setInt(CommonConfigurationKeys.HA_HM_CHECK_INTERVAL_KEY, 50);
    // HA功能的健康监控连接重试间隔(毫秒)
    conf.setInt(CommonConfigurationKeys.HA_HM_CONNECT_RETRY_INTERVAL_KEY, 50);
    // HA功能的健康监控，在因网络问题失去连接后休眠多久。用于避免立即重试(毫秒)
    conf.setInt(CommonConfigurationKeys.HA_HM_SLEEP_AFTER_DISCONNECT_KEY, 50);

    // 从这里看的话, 就是创建了一个虚拟服务, 然后状态是主NameNode
    svc = new DummyHAService(HAServiceState.ACTIVE,
        new InetSocketAddress("0.0.0.0", 0), true);


    hm = new HealthMonitor(conf, svc) {
      @Override
      protected HAServiceProtocol createProxy() throws IOException {
        createProxyCount.incrementAndGet();
        if (throwOOMEOnCreate) {
          throw new OutOfMemoryError("oome");
        }
        return super.createProxy();
      }
    };
    LOG.info("Starting health monitor");

    // HealthMonitor内部类MonitorDaemon线程进行轮询
    hm.start();

    LOG.info("Waiting for HEALTHY signal");    
    waitForState(hm, HealthMonitor.State.SERVICE_HEALTHY);
  }


  @Test(timeout=15000)
    public void testMonitor() throws Exception {
      LOG.info("Mocking bad health check, waiting for UNHEALTHY");
      svc.isHealthy = false;
      waitForState(hm, HealthMonitor.State.SERVICE_UNHEALTHY);

      LOG.info("Returning to healthy state, waiting for HEALTHY");
      svc.isHealthy = true;
      waitForState(hm, HealthMonitor.State.SERVICE_HEALTHY);

      LOG.info("Returning an IOException, as if node went down");
      // should expect many rapid retries
      int countBefore = createProxyCount.get();
      svc.actUnreachable = true;
      waitForState(hm, HealthMonitor.State.SERVICE_NOT_RESPONDING);

      // Should retry several times
      while (createProxyCount.get() < countBefore + 3) {
        Thread.sleep(10);
      }

      LOG.info("Returning to healthy state, waiting for HEALTHY");
      svc.actUnreachable = false;
      waitForState(hm, HealthMonitor.State.SERVICE_HEALTHY);

      hm.shutdown();
      hm.join();
      assertFalse(hm.isAlive());
    }
```

我们来看看HealthMonitor构造方法

```java
// HealthMonitor.java

HealthMonitor(Configuration conf, HAServiceTarget target) {
    this.targetToMonitor = target;
    this.conf = conf;

    /**
      如果我们在构造Configuration的时候没有配置上面4个相关参数的话, 则使用hadoop默认的配置值。
    */
    this.sleepAfterDisconnectMillis = conf.getLong(
        HA_HM_SLEEP_AFTER_DISCONNECT_KEY,
        HA_HM_SLEEP_AFTER_DISCONNECT_DEFAULT);
    this.checkIntervalMillis = conf.getLong(
        HA_HM_CHECK_INTERVAL_KEY,
        HA_HM_CHECK_INTERVAL_DEFAULT);
    this.connectRetryInterval = conf.getLong(
        HA_HM_CONNECT_RETRY_INTERVAL_KEY,
        HA_HM_CONNECT_RETRY_INTERVAL_DEFAULT);
    this.rpcTimeout = conf.getInt(
        HA_HM_RPC_TIMEOUT_KEY,
        HA_HM_RPC_TIMEOUT_DEFAULT);

    // 上面的hm.start()方法被调用, 也就是该监控线程被调用了。
    this.daemon = new MonitorDaemon();
}

```

下面, 看看线程中究竟隐藏了什么秘密...

```java
// HealthMonitor.java
private class MonitorDaemon extends Daemon {

  private MonitorDaemon() {
    super();
    setName("Health Monitor for " + targetToMonitor);
    // 注意这里的异常处理
    setUncaughtExceptionHandler(new UncaughtExceptionHandler() {
      @Override
      public void uncaughtException(Thread t, Throwable e) {
        LOG.fatal("Health monitor failed", e);
        // 如果出现了异常, 那么就把状态修改为HEALTH_MONITOR_FAILED
        // 自己在运行过程中发生了异常, 不能继续监测NameNode的健康状况
        enterState(HealthMonitor.State.HEALTH_MONITOR_FAILED);
      }
    });
  }


    @Override
    public void run() {
      while (shouldRun) {
        try {
          // 也就是说实际上是这两个方法来完成NameNode的健康状态监控的, 看到胜利的希望了
          loopUntilConnected();
          doHealthChecks();
        } catch (InterruptedException ie) {
          Preconditions.checkState(!shouldRun,
              "Interrupted but still supposed to run");
        }
      }
    }
  }
```



```java

// 一直循环直到连接成功为止...
private void loopUntilConnected() throws InterruptedException {
    tryConnect();
    while (proxy == null) { // 如果没有成功创建HA Service的代理对象则休眠一段时间后继续执行
      Thread.sleep(connectRetryInterval);
      tryConnect();
    }
    assert proxy != null; //
  }

  private void tryConnect() {
    Preconditions.checkState(proxy == null); // 条件判断检查

    try {
      synchronized (this) {
        proxy = createProxy();
      }
    } catch (IOException e) {
      LOG.warn("Could not connect to local service at " + targetToMonitor +
          ": " + e.getMessage());
      proxy = null;
      // 这里把状态调整到 NameNode 的 monitorHealth 方法调用无响应或响应超时
      // 刚开始我有点不明白为什么, 但是看返回的代理对象是HAServiceProtocol子类, 里面实现了monitorHealth方法
      // 包括一些切换的方法, 但是我们这里作为监控健康设置为SERVICE_NOT_RESPONDING是合理的。
      // 要是没有该代理对象, 我们没法执行doHealthChecks()方法的, so...
      enterState(State.SERVICE_NOT_RESPONDING);
    }
  }
```


```java
private void doHealthChecks() throws InterruptedException {
    while (shouldRun) {
      HAServiceStatus status = null;
      boolean healthy = false;
      try {
        status = proxy.getServiceStatus();
        proxy.monitorHealth(); // 这里不会运行到NameNode端去的, 而是调用到们的DummyHAService中的MockHAProtocolImpl内部类
        healthy = true; //
      } catch (Throwable t) {

        // 下面这些异常判断就没有什么好介绍的了。更具上面的状态码来看即可。

        if (isHealthCheckFailedException(t)) {
          LOG.warn("Service health check failed for " + targetToMonitor
              + ": " + t.getMessage());
          enterState(State.SERVICE_UNHEALTHY);
        } else {
          LOG.warn("Transport-level exception trying to monitor health of " +
              targetToMonitor + ": " + t.getCause() + " " + t.getLocalizedMessage());
          RPC.stopProxy(proxy);
          proxy = null;
          enterState(State.SERVICE_NOT_RESPONDING);
          Thread.sleep(sleepAfterDisconnectMillis);
          return;
        }
      }

      // 设置最后的状态
      if (status != null) {
        setLastServiceStatus(status);
      }
      if (healthy) {
        enterState(State.SERVICE_HEALTHY);
      }

      Thread.sleep(checkIntervalMillis);
    }
  }
```


```java
// DummyHAService.java
/***
  (注意: 实际上是调用NameNode的, 这里只是测试写的类)

  如果monitorHealth没有响应则抛出异常了, 如果monitorHealth正常能链接就但是不健康则抛出异常
  切合上面调用的catch。
*/
private class MockHAProtocolImpl implements
      HAServiceProtocol, Closeable {
    @Override
    public void monitorHealth() throws HealthCheckFailedException,
        AccessControlException, IOException {
      checkUnreachable();
      if (!isHealthy) {
        throw new HealthCheckFailedException("not healthy");
      }
    }

    private void checkUnreachable() throws IOException {
      if (actUnreachable) {
        throw new IOException("Connection refused (fake)");
      }
    }
  }

```

```java

/**
  这里就是实际线上调用并运行的方法
*/

// NameNodeRpcServer.java

protected final NameNode nn;

@Override // HAServiceProtocol
  public synchronized void monitorHealth() throws HealthCheckFailedException,
      AccessControlException, IOException {
    checkNNStartup(); // 判断NameNode是否启动
    nn.monitorHealth(); // 请求到NameNode去了.
}

private void checkNNStartup() throws IOException {
    if (!this.nn.isStarted()) {
      throw new RetriableException(this.nn.getRole() + " still not started");
    }
}


// NameNode.java
synchronized void monitorHealth()
      throws HealthCheckFailedException, AccessControlException {
    namesystem.checkSuperuserPrivilege();
    if (!haEnabled) { // 判断是否开启了HA
      return; // no-op, if HA is not enabled
    }
    getNamesystem().checkAvailableResources(); // 检查判断是否还有可用的资源
    // 如果NameNode没有可用的资源则抛出异常信息
    if (!getNamesystem().nameNodeHasResourcesAvailable()) {
      throw new HealthCheckFailedException(
          "The NameNode has no resources available");
    }
}


// FSNamesystem.java
void checkAvailableResources() {
    Preconditions.checkState(nnResourceChecker != null,
        "nnResourceChecker not initialized");
    hasResourcesAvailable = nnResourceChecker.hasAvailableDiskSpace();
}



// NameNodeResourceChecker.java

/**
   * Return true if disk space is available on at least one of the configured
   * redundant volumes, and all of the configured required volumes.
   *
   * @return True if the configured amount of disk space is available on at
   *         least one redundant volume and all of the required volumes, false
   *         otherwise.
   */
public boolean hasAvailableDiskSpace() {
    // 这里也就是判断我们是否有可用的磁盘资源, 只有有一个可用的即返回true, 否则磁盘空间不可以返回false
    return NameNodeResourcePolicy.areResourcesAvailable(volumes.values(),
        minimumRedundantVolumes);
}

```


到这里基本上已经算是完事了, 但是我们还有最后一个方法没有介绍enterState()方法。他有什么作用呢?

```java

// HealthMonitor.java

/**
   * Listeners for state changes
   */
private List<Callback> callbacks = Collections.synchronizedList(
      new LinkedList<Callback>());

public void addCallback(Callback cb) {
  this.callbacks.add(cb);
}

private synchronized void enterState(State newState) {
    if (newState != state) {
      LOG.info("Entering state " + newState);
      state = newState; // 默认为初始化的状态, 调用enterState方法值时设置当前状态
      synchronized (callbacks) {
        for (Callback cb : callbacks) {
          cb.enteredState(newState); // 回调ZKFC的enteredState方法
        }
      }
    }
}


/**
   * Callback interface for state change events.
   *
   * This interface is called from a single thread which also performs
   * the health monitoring. If the callback processing takes a long time,
   * no further health checks will be made during this period, nor will
   * other registered callbacks be called.
   *
   * If the callback itself throws an unchecked exception, no other
   * callbacks following it will be called, and the health monitor
   * will terminate, entering HEALTH_MONITOR_FAILED state.
   */
  static interface Callback {
    void enteredState(State newState);
  }
```


```java

// ZKFailoverController.java

/**
   * Callbacks from HealthMonitor
   */
  class HealthCallbacks implements HealthMonitor.Callback {
    @Override
    public void enteredState(HealthMonitor.State newState) {
      setLastHealthState(newState);
      recheckElectability();
    }
  }
```
