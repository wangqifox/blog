---
title: canal——monitor机制
date: 2019/03/06 11:18:00
---

经过前文[canal——代码初读][1]的分析，我们对canal的工作流程有了初步的了解。本文，我们来分析canal的monitor机制。

<!-- more -->

canal的流程描述中提及了两个monitor：`InstanceConfigMonitor`、`ServerRunningMonitor`。本文重点对这两个monitor进行分析。

# InstanceConfigMonitor

从字面意思就可以得出，`InstanceConfigMonitor`是对canal实例配置的监控。实际上也确实如此，它的功能就是监控canal实例的配置文件——`instance.properties`，如果其中有增加、修改、删除，执行实例的启动、重启、停止操作。下面我们来分析`InstanceConfigMonitor`的代码。

## InstanceConfigMonitor的创建

`InstanceConfigMonitor`在`CanalController`构造函数中被创建：

首先获取`canal.auto.scan`配置，该配置控制是否监听canal实例配置的变化。如果该配置为`false`，则不需要创建`InstanceConfigMonitor`。默认该配置为`true`，此时需要创建`InstanceConfigMonitor`。

在创建`InstanceConfigMonitor`之前先创建一个`defaultAction`（实际类为`InstanceAction`），`InstanceAction`是一个接口，其中定义了三个函数，分别对应启动实例、定制实例、重启实例：

```java
public interface InstanceAction {
    /**
     * 启动destination
     */
    void start(String destination);

    /**
     * 停止destination
     */
    void stop(String destination);

    /**
     * 重载destination，可能需要stop,start操作，或者只是更新下内存配置
     */
    void reload(String destination);
}
```

`InstanceAction`的具体操作我们在后面再分析。

接着创建一个`instanceConfigMonitors`，它是一个`Map<InstanceMode, InstanceConfigMonitor>`，当获取`InstanceConfigMonitor`是发现Map中不存在时（从调用流程上看，`InstanceConfigMonitor`其实是在其将要启动时获取的），调用`apply`方法创建一个`InstanceConfigMonitor`：

```java
int scanInterval = Integer
    .valueOf(getProperty(properties, CanalConstants.CANAL_AUTO_SCAN_INTERVAL));

if (mode.isSpring()) {
    SpringInstanceConfigMonitor monitor = new SpringInstanceConfigMonitor();
    monitor.setScanIntervalInSecond(scanInterval);
    monitor.setDefaultAction(defaultAction);
    // 设置conf目录，默认是user.dir + conf目录组成
    String rootDir = getProperty(properties, CanalConstants.CANAL_CONF_DIR);
    if (StringUtils.isEmpty(rootDir)) {
        rootDir = "../conf";
    }

    if (StringUtils.equals("otter-canal", System.getProperty("appName"))) {
        monitor.setRootConf(rootDir);
    } else {
        // eclipse debug模式
        monitor.setRootConf("src/main/resources/");
    }
    return monitor;
} else if (mode.isManager()) {
    return new ManagerInstanceConfigMonitor();
} else {
    throw new UnsupportedOperationException("unknow mode :" + mode + " for monitor");
}
```

创建流程如下：

1. 获取配置`canal.auto.scan.interval`，它指定了配置监听扫描的间隔时间，默认为5秒。
2. 判断mode是否为`SPRING`。mode在配置`canal.instance.global.mode`中指定，可以正对各个destination配置不同的mode。全局mode默认为`spring`
3. 新建`SpringInstanceConfigMonitor`
4. 设置`scanInterval`
5. 设置`defaultAction`。`defaultAction`为刚刚创建的`InstanceAction`。
6. 获取并设置配置`canal.conf.dir`，这是监控进行的根目录。

## InstanceConfigMonitor的启动

`InstanceConfigMonitor`的启动位于`CanalController`的`start()`方法中。

在启动之前，还有一步遍历destination并注册`InstanceAction`。

```
if (autoScan) {
    instanceConfigMonitors.get(config.getMode()).register(destination, defaultAction);
}

// 注册InstanceAction就是将action保存在SpringInstanceConfigMonitor的actions字段中
public void register(String destination, InstanceAction action) {
    if (action != null) {
        actions.put(destination, action);
    } else {
        actions.put(destination, defaultAction);
    }
}
```

其中执行`instanceConfigMonitors.get(config.getMode())`时发现`instanceConfigMonitors`不存在对应的`InstanceConfigMonitor`，先创建`InstanceConfigMonitor`。

接着调用`start()`方法启动。对于`SpringInstanceConfigMonitor`来说，`start()`方法中启动了一个定时任务，每隔`scanIntervalInSecond`时间（默认5秒），执行其中的`scan()`方法：

```java
private void scan() {
    // 判断instance配置的根目录是否存在
    File rootdir = new File(rootConf);
    if (!rootdir.exists()) {
        return;
    }
    // 获取所有instance的配置目录
    File[] instanceDirs = rootdir.listFiles(new FileFilter() {

        public boolean accept(File pathname) {
            String filename = pathname.getName();
            return pathname.isDirectory() && !"spring".equalsIgnoreCase(filename);
        }
    });

    // currentInstanceNames用于判断是否有目录被删除
    Set<String> currentInstanceNames = new HashSet<String>();

    // 判断目录内文件的变化
    for (File instanceDir : instanceDirs) {
        // 将现在配置的destination添加到currentInstanceNames
        String destination = instanceDir.getName();
        currentInstanceNames.add(destination);
        // 获取配置目录中的配置文件instance.properties
        File[] instanceConfigs = instanceDir.listFiles(new FilenameFilter() {

            public boolean accept(File dir, String name) {
                // return !StringUtils.endsWithIgnoreCase(name, ".dat");
                // 限制一下，只针对instance.properties文件,避免因为.svn或者其他生成的临时文件导致出现reload
                return StringUtils.equalsIgnoreCase(name, "instance.properties");
            }

        });

        if (!actions.containsKey(destination) && instanceConfigs.length > 0) {
            // 存在合法的instance.properties，并且第一次添加时，进行启动操作
            notifyStart(instanceDir, destination, instanceConfigs);
        } else if (actions.containsKey(destination)) {
            // 历史已经启动过
            if (instanceConfigs.length == 0) {
                // 如果不存在合法的instance.properties（可能是这个文件被删除了），则停止instance
                notifyStop(destination);
            } else {
                // 判断instance.properties是否被修改过
                InstanceConfigFiles lastFile = lastFiles.get(destination);
                // 历史启动过 所以配置文件信息必然存在
                if (!isFirst && CollectionUtils.isEmpty(lastFile.getInstanceFiles())) {
                    logger.error("[{}] is started, but not found instance file info.", destination);
                }

                // 通过文件的最近修改时间，判断配置文件是否被修改过
                boolean hasChanged = judgeFileChanged(instanceConfigs, lastFile.getInstanceFiles());
                // 通知变化
                if (hasChanged) {
                    notifyReload(destination);
                }

                // 如果配置文件的文件信息没有保存（可能是刚启动，或者配置文件刚添加），则首先保存文件信息
                if (hasChanged || CollectionUtils.isEmpty(lastFile.getInstanceFiles())) {
                    // 更新内容
                    List<FileInfo> newFileInfo = new ArrayList<FileInfo>();
                    for (File instanceConfig : instanceConfigs) {
                        newFileInfo.add(new FileInfo(instanceConfig.getName(), instanceConfig.lastModified()));
                    }

                    lastFile.setInstanceFiles(newFileInfo);
                }
            }
        }

    }

    // 判断目录是否删除
    Set<String> deleteInstanceNames = new HashSet<String>();
    for (String destination : actions.keySet()) {
        // destination的配置不存在了，说明它被删除了
        if (!currentInstanceNames.contains(destination)) {
            deleteInstanceNames.add(destination);
        }
    }
    for (String deleteInstanceName : deleteInstanceNames) {
        notifyStop(deleteInstanceName);
    }
}
```

`scan()`方法的作用就扫描instance的配置文件，如果有增加、修改、删除，则分别调用`notifyStart`、`notifyReload`、`notifyStop`操作。三个操作的代码如下：

```java
private void notifyStart(File instanceDir, String destination, File[] instanceConfigs) {
    try {
        //调用defaultAction中定义的instance的启动操作
        defaultAction.start(destination);
        actions.put(destination, defaultAction);

        // 启动成功后记录配置文件信息
        InstanceConfigFiles lastFile = lastFiles.get(destination);
        List<FileInfo> newFileInfo = new ArrayList<FileInfo>();
        for (File instanceConfig : instanceConfigs) {
            newFileInfo.add(new FileInfo(instanceConfig.getName(), instanceConfig.lastModified()));
        }
        lastFile.setInstanceFiles(newFileInfo);

        logger.info("auto notify start {} successful.", destination);
    } catch (Throwable e) {
        logger.error(String.format("scan add found[%s] but start failed", destination), e);
    }
}

private void notifyReload(String destination) {
    InstanceAction action = actions.get(destination);
    if (action != null) {
        try {
            //调用defaultAction中定义的instance的重启操作
            action.reload(destination);
            logger.info("auto notify reload {} successful.", destination);
        } catch (Throwable e) {
            logger.error(String.format("scan reload found[%s] but reload failed", destination), e);
        }
    }
}

private void notifyStop(String destination) {
    InstanceAction action = actions.remove(destination);
    try {
        //调用defaultAction中定义的instance的停止操作
        action.stop(destination);
        lastFiles.remove(destination);
        logger.info("auto notify stop {} successful.", destination);
    } catch (Throwable e) {
        logger.error(String.format("scan delete found[%s] but stop failed", destination), e);
        actions.put(destination, action);// 再重新加回去，下一次scan时再执行删除
    }
}
```

可以看到`notifyStart`、`notifyReload`、`notifyStop`三个操作主要就是调用`defaultAction`中定义的Instance操作对Instance执行启动、重启、停止操作。

下面来看`defaultAction`中定义的操作：

```java
defaultAction = new InstanceAction() {
    public void start(String destination) {
        // 获取instance的配置
        InstanceConfig config = instanceConfigs.get(destination);
        if (config == null) {
            // 如果配置为null（可能是instance停止后配置被删除了），重新读取一下instance config
            config = parseInstanceConfig(properties, destination);
            instanceConfigs.put(destination, config);
        }

        if (!embededCanalServer.isStart(destination)) {
            // HA机制启动
            ServerRunningMonitor runningMonitor = ServerRunningMonitors.getRunningMonitor(destination);
            if (!config.getLazy() && !runningMonitor.isStart()) {
                // 调用ServerRunningMonitor的start()方法启动instance
                runningMonitor.start();
                if (canalMQStarter != null) {
                    canalMQStarter.startDestination(destination);
                }
            }
        }
    }

    public void stop(String destination) {
        // 此处的stop，代表强制退出，非HA机制，所以需要退出HA的monitor和配置信息
        InstanceConfig config = instanceConfigs.remove(destination);
        if (config != null) {
            if (canalMQStarter != null) {
                canalMQStarter.stopDestination(destination);
            }
            embededCanalServer.stop(destination);
            ServerRunningMonitor runningMonitor = ServerRunningMonitors.getRunningMonitor(destination);
            if (runningMonitor.isStart()) {
                // 调用ServerRunningMonitor的stop()方法停止instance
                runningMonitor.stop();
            }
        }
    }

    public void reload(String destination) {
        // 目前任何配置变化，直接重启，简单处理
        stop(destination);
        start(destination);
    }
};
```

可以看到，`defaultAction`中定义了三个操作，其中`reload`操作就是简单地停止并启动instance，`stop`操作删除instance的配置然后调用`ServerRunningMonitor`的`stop()`方法停止instance，`start`操作读取instance配置并调用调用`ServerRunningMonitor`的`start()`方法启动instance。

# ServerRunningMonitor

`ServerRunningMonitor`是对canal实例进行控制并监控的类。

`ServerRunningMonitor`在`ServerRunningMonitors`中维护，`ServerRunningMonitors`中的`runningMonitors`变量保存各个destination对应的`ServerRunningMonitor`。

## 控制canal的实例的启停

`ServerRunningMonitor`第一个功能就是控制canal的实例的启停，由`start()`、`stop()`两个方法负责：

```java
public synchronized void start() {
    super.start();
    try {
        processStart();
        if (zkClient != null) {
            // 如果需要尽可能释放instance资源，不需要监听running节点，不然即使stop了这台机器，另一台机器立马会start
            String path = ZookeeperPathUtils.getDestinationServerRunning(destination);
            zkClient.subscribeDataChanges(path, dataListener);

            initRunning();
        } else {
            processActiveEnter();// 没有zk，直接启动
        }
    } catch (Exception e) {
        logger.error("start failed", e);
        // 没有正常启动，重置一下状态，避免干扰下一次start
        stop();
    }

}


public synchronized void stop() {
    super.stop();

    if (zkClient != null) {
        String path = ZookeeperPathUtils.getDestinationServerRunning(destination);
        zkClient.unsubscribeDataChanges(path, dataListener);

        releaseRunning(); // 尝试一下release
    } else {
        processActiveExit(); // 没有zk，直接启动
    }
    processStop();
}
```

先介绍其中涉及到四个`process**`回调方法：`processStart`、`processStop`、`processActiveEnter`、`processActiveExit`。这个四个方法执行的功能非常简单，就是调用`ServerRunningListener`中相应的方法。`ServerRunningListener`是一个接口，其中定义了四个instance状态发生改变时回调的方法：

```java
public interface ServerRunningListener {
    /**
     * 启动时回调做点事情
     */
    public void processStart();

    /**
     * 关闭时回调做点事情
     */
    public void processStop();

    /**
     * 触发现在轮到自己做为active，需要载入上一个active的上下文数据
     */
    public void processActiveEnter();

    /**
     * 触发一下当前active模式失败
     */
    public void processActiveExit();
}
```

canal在`ServerRunningMonitor`的新建过程中新建并设置了`ServerRunningListener`的实现，下面分别介绍这个四个方法的实现：

- `processStart`：如果使用了zookeeper，在zookeeper中新建cluster的目录（目录名：`/otter/canal/destinations/{destination}/cluster/{ip}:{port}`），监控zookeeper状态改变
- `processStop`：如果使用了zookeeper，删除zookeeper中的cluster目录
- `processActiveEnter`：调用`CanalServerWithEmbedded`的`start`方法启动destination对应的实例
- `processActiveExit`：调用`CanalServerWithEmbedded`的`stop`方法停止destination对应的实例

现在回到`ServerRunningMonitor`的`start`和`stop`方法：

- `start()`：

    1. 调用`processStart`方法，作用如上文所示
    2. 如果使用了zookeeper，监听zookeeper中的`/otter/canal/destinations/{destination}/running`节点。调用`initRunning`方法，运行canal实例
    3. 如果没有使用zookeeper，直接调用`processActiveEnter`方法启动实例

- `stop()`：

    1. 如果使用了zookeeper，取消监听zookeeper中的`/otter/canal/destinations/{destination}/running`节点。调用`releaseRunning`方法停止canal实例
    2. 如果没有使用zookeeper，直接调用`processActiveExit`方法停止实例

`initRunning`方法如下所示：

它的作用是在zookeeper的`/otter/canal/destinations/{destination}/running`节点中写入当前运行的canal服务器信息，并调用`processActiveEnter`方法启动实例。

如果该节点已经存在，则重新获取数据并将该数据作为正在运行的canal信息保存在`activeData`中。

```java
private void initRunning() {
    if (!isStart()) {
        return;
    }

    String path = ZookeeperPathUtils.getDestinationServerRunning(destination);
    // 序列化
    byte[] bytes = JsonUtils.marshalToByte(serverData);
    try {
        mutex.set(false);
        zkClient.create(path, bytes, CreateMode.EPHEMERAL);
        activeData = serverData;
        processActiveEnter();// 触发一下事件
        mutex.set(true);
    } catch (ZkNodeExistsException e) {
        bytes = zkClient.readData(path, true);
        if (bytes == null) {// 如果不存在节点，立即尝试一次
            initRunning();
        } else {
            activeData = JsonUtils.unmarshalFromByte(bytes, ServerRunningData.class);
        }
    } catch (ZkNoNodeException e) {
        zkClient.createPersistent(ZookeeperPathUtils.getDestinationPath(destination), true); // 尝试创建父节点
        initRunning();
    }
}
```

`releaseRunning`方法如下所示。它的作用是删除zookeeper的`/otter/canal/destinations/{destination}/running`节点，并调用`processActiveExit`方法停止实例。

```java
private boolean releaseRunning() {
    if (check()) {
        String path = ZookeeperPathUtils.getDestinationServerRunning(destination);
        zkClient.delete(path);
        mutex.set(false);
        processActiveExit();
        return true;
    }

    return false;
}
```

## 监听zookeeper，如果zookeeper中的数据发生改变采取措施

`ServerRunningMonitor`第二个功能就是监听zookeeper，如果zookeeper中的数据发生改变采取措施。

前面我们看到`ServerRunningMonitor`在执行`start()`方法时会监听zookeeper中的`/otter/canal/destinations/{destination}/running`节点，其响应方法在dataListener中定义：

```java
dataListener = new IZkDataListener() {
    public void handleDataChange(String dataPath, Object data) throws Exception {
        MDC.put("destination", destination);
        ServerRunningData runningData = JsonUtils.unmarshalFromByte((byte[]) data, ServerRunningData.class);
        if (!isMine(runningData.getAddress())) {
            mutex.set(false);
        }

        if (!runningData.isActive() && isMine(runningData.getAddress())) { // 说明出现了主动释放的操作，并且本机之前是active
            release = true;
            releaseRunning();// 彻底释放mainstem
        }

        activeData = (ServerRunningData) runningData;
    }

    public void handleDataDeleted(String dataPath) throws Exception {
        MDC.put("destination", destination);
        mutex.set(false);
        if (!release && activeData != null && isMine(activeData.getAddress())) {
            // 如果上一次active的状态就是本机，则即时触发一下active抢占
            initRunning();
        } else {
            // 否则就是等待delayTime，避免因网络瞬端或者zk异常，导致出现频繁的切换操作
            delayExector.schedule(new Runnable() {

                public void run() {
                    initRunning();
                }
            }, delayTime, TimeUnit.SECONDS);
        }
    }

};
```

当`/otter/canal/destinations/{destination}/running`节点的数据发生修改会调用`handleDataChange`方法：

1. 获取该目录修改之后的数据
2. 如果数据显示正在运行的canal的地址和本机不同，说明已经有其他机器上的canal正在运行
3. 如果数据显示正在运行的canal就是本机，并且状态不是active，说明本机出现了主动释放的操作，此时调用`releaseRunning()`方法删除zookeeper的`/otter/canal/destinations/{destination}/running`节点，并调用`processActiveExit`方法停止实例
4. 将正在运行的canal信息保存在`activeData`中

当`/otter/canal/destinations/{destination}/running`节点的数据被删除会调用`handleDataDeleted`方法：

1. 如果上次运行canal的就是本机，则执行`initRunning`重新运行本机的canal
2. 否则等待`delayTime`（5秒）时间后再执行`initRunning`重新运行本机的canal

# 总结

本文分析了canal的两个monitor机制：`InstanceConfigMonitor`、`ServerRunningMonitor`。通过这两个监控机制，canal实现了实例配置的修改与生效，以及实例在不同机器上的高可用。















[1]: /articles/canal/canal——代码初读.html

