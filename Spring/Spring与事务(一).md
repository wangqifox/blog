---
title: Spring与事务(一)
date: 2018/01/18 17:26:00
---

上回说到，直接使用JDBC操作数据库增删改查是非常繁琐的，因此Spring提供了模板类(`JdbcTemplate`, `NamedParameterJdbcTemplate`)来简化我们和数据库的交互。同样的，直接使用JDBC进行事务的关系也需要写很多非业务代码，Spring提供了一个接口来完成事务的提交和回滚等功能，即接口`PlatformTransactionManager`，如果是使用jdbc则使用`DataSourceTransactionManager`来完成事务的提交和回滚，如果是使用hibernate则使用HibernateTransactionManager来完成事务的提交和回滚。

事务的定义接口为`TransactionDefinition`，它有两个重要的属性，事务的传播属性和隔离级别：
<!-- more -->
```java
public interface TransactionDefinition {
    int PROPAGATION_REQUIRED = 0;
    int PROPAGATION_SUPPORTS = 1;
    int PROPAGATION_MANDATORY = 2;
    int PROPAGATION_REQUIRES_NEW = 3;
    int PROPAGATION_NOT_SUPPORTED = 4;
    int PROPAGATION_NEVER = 5;
    int PROPAGATION_NESTED = 6;

    int ISOLATION_DEFAULT = -1;
    int ISOLATION_READ_UNCOMMITTED = Connection.TRANSACTION_READ_UNCOMMITTED;
    int ISOLATION_READ_COMMITTED = Connection.TRANSACTION_READ_COMMITTED;
    int ISOLATION_REPEATABLE_READ = Connection.TRANSACTION_REPEATABLE_READ;
    int ISOLATION_SERIALIZABLE = Connection.TRANSACTION_SERIALIZABLE;

    int TIMEOUT_DEFAULT = -1;

    int getPropagationBehavior();
    int getIsolationLevel();
    int getTimeout();
    boolean isReadOnly();
    String getName();
}
```

`DefaultTransactionDefinition`是对上述属性设置的一些默认值，传播属性默认为`PROPAGATION_REQUIRED`，隔离级别默认为`ISOLATION_DEFAULT`，采用的是底层数据库采用的默认隔离级别。

`TransactionAttribute`接口：

```java
public interface TransactionAttribute extends TransactionDefinition {
    // 在选择事务管理器时用到
    String getQualifier();
    // 指明什么样的异常进行回滚
    boolean rollbackOn(Throwable ex);
}
```

`DefaultTransactionAttribute`：

```java
public boolean rollbackOn(Throwable ex) {
    // 对RuntimeException和Error都进行回滚
    return (ex instanceof RuntimeException || ex instanceof Error);
}
```

## TransactionTemplate使用示例

首先新建`DataSourceTransactionManager`的bean：

```java
@Bean
public PlatformTransactionManager platformTransactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}
```

然后新建`TransactionTemplate`：

```java
@Bean
public TransactionTemplate transactionTemplate(PlatformTransactionManager platformTransactionManager) {
    TransactionTemplate transactionTemplate = new TransactionTemplate();
    transactionTemplate.setTransactionManager(platformTransactionManager);
    return transactionTemplate;
}
```

看一下`TransactionTemplate`的使用示例：

```java
public String addUserWithTransaction(User user) {
    String result = transactionTemplate.execute(new TransactionCallback<String>() {
        @Override
        public String doInTransaction(TransactionStatus status) {
            addUser(user);
            return "success";
        }
    });
    return result;
}
```

## TransactionTemplate原理

```java
public <T> T execute(TransactionCallback<T> action) throws TransactionException {
    if (this.transactionManager instanceof CallbackPreferringPlatformTransactionManager) {
        return ((CallbackPreferringPlatformTransactionManager) this.transactionManager).execute(this, action);
    }
    else {
        TransactionStatus status = this.transactionManager.getTransaction(this);
        T result;
        try {
            result = action.doInTransaction(status);
        }
        catch (RuntimeException ex) {
            // Transactional code threw application exception -> rollback
            rollbackOnException(status, ex);
            throw ex;
        }
        catch (Error err) {
            // Transactional code threw error -> rollback
            rollbackOnException(status, err);
            throw err;
        }
        catch (Throwable ex) {
            // Transactional code threw unexpected exception -> rollback
            rollbackOnException(status, ex);
            throw new UndeclaredThrowableException(ex, "TransactionCallback threw undeclared checked exception");
        }
        this.transactionManager.commit(status);
        return result;
    }
}
```

首先调用`DataSourceTransactionManager.getTransaction`方法获取事务状态：

### DataSourceTransactionManager.getTransaction

getTransaction方法处理传播行为

```java
public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
    // 获取事务对象
    Object transaction = doGetTransaction();

    boolean debugEnabled = logger.isDebugEnabled();

    if (definition == null) {
        definition = new DefaultTransactionDefinition();
    }

    // 检查是否已经存在事务，如果有则调用handleExistingTransaction处理
    if (isExistingTransaction(transaction)) {
        return handleExistingTransaction(definition, transaction, debugEnabled);
    }

    // Check definition settings for new transaction.
    if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
    }

    // No existing transaction found -> check propagation behavior to find out how to proceed.
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
                "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        SuspendedResourcesHolder suspendedResources = suspend(null);
        if (debugEnabled) {
            logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
        }
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            // 开始事务
            doBegin(transaction, definition);
            // 根据需要初始化事务同步
            prepareSynchronization(status, definition);
            return status;
        }
        catch (RuntimeException ex) {
            resume(null, suspendedResources);
            throw ex;
        }
        catch (Error err) {
            resume(null, suspendedResources);
            throw err;
        }
    }
    else {
        // Create "empty" transaction: no actual transaction, but potentially synchronization.
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
            logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                    "isolation level will effectively be ignored: " + definition);
        }
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
    }
}
```

首先调用`DataSourceTransactionManager.doGetTransaction`。

`doGetTransaction`方法返回一个事务对象，表示当前的事务状态。该对象通常是具体事务管理的实现，以可修改的方式携带相应的事务状态。该对象将被直接或作为`DefaultTransactionStatus`实例的一部分传递给其他模板方法(例如：doBegin和doCommit)。

返回的对象应包含有关任何现有事务的信息，即事务管理器上当前调用`getTransaction`之前已经开始的事务。因此，`doGetTransaction`实现通常会查找现有的事务，并在返回的事务对象中存储对应的状态。

```java
protected Object doGetTransaction() {
    DataSourceTransactionObject txObject = new DataSourceTransactionObject();
    txObject.setSavepointAllowed(isNestedTransactionAllowed());
    // 获取当前的连接资源
    ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(this.dataSource);
    txObject.setConnectionHolder(conHolder, false);
    return txObject;
}
```

`TransactionSynchronizationManager.getResource`：

```java
public static Object getResource(Object key) {
    Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
    Object value = doGetResource(actualKey);
    if (value != null && logger.isTraceEnabled()) {
        logger.trace("Retrieved value [" + value + "] for key [" + actualKey + "] bound to thread [" +
                Thread.currentThread().getName() + "]");
    }
    return value;
}

private static Object doGetResource(Object actualKey) {
    Map<Object, Object> map = resources.get();
    if (map == null) {
        return null;
    }
    Object value = map.get(actualKey);
    // Transparently remove ResourceHolder that was marked as void...
    if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
        map.remove(actualKey);
        // Remove entire ThreadLocal if empty...
        if (map.isEmpty()) {
            resources.remove();
        }
        value = null;
    }
    return value;
}
```

## handleExistingTransaction

回到`getTransaction`方法，调用`isExistingTransaction`判断是否已经存在事务，如果已经存在则调用`handleExistingTransaction`。

```java
private TransactionStatus handleExistingTransaction(
        TransactionDefinition definition, Object transaction, boolean debugEnabled)
        throws TransactionException {
    // 如果传播行为是PROPAGATION_NEVER（必须在一个没有的事务中执行），直接抛出异常
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException(
                "Existing transaction found for transaction marked with propagation 'never'");
    }
    // 如果传播行为是PROPAGATION_NOT_SUPPORTED（容器不为这个方法开启事务）
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction");
        }
        // 挂起当前事务，返回一个新的事务状态
        Object suspendedResources = suspend(transaction);
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(
                definition, null, false, newSynchronization, debugEnabled, suspendedResources);
    }
    // 如果传播行为是PROPAGATION_REQUIRES_NEW（不管是否存在事务，都创建一个新的事务，原来的挂起，新的执行完毕，继续执行老的事务）
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction, creating new transaction with name [" +
                    definition.getName() + "]");
        }
        // 挂起当前事务
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            // 开启一个新事务
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
        // 如果里面的新事务发生异常，继续外面的事务
        catch (RuntimeException beginEx) {
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
        catch (Error beginErr) {
            resumeAfterBeginException(transaction, suspendedResources, beginErr);
            throw beginErr;
        }
    }

    // 如果传播行为是PROPAGATION_NESTED（使用SavePoint技术把当前事务状态进行保存，然后底层共用一个连接，当NESTED内部出错的时候，自行回滚到SavePoint这个状态，只要外部捕获到了异常，就可以继续进行外部的事务提交，而不会受到内嵌业务的干扰，但是，如果外部事务抛出了异常，整个大事务都会回滚）
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        if (!isNestedTransactionAllowed()) {
            throw new NestedTransactionNotSupportedException(
                    "Transaction manager does not allow nested transactions by default - " +
                    "specify 'nestedTransactionAllowed' property with value 'true'");
        }
        if (debugEnabled) {
            logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
        }
        if (useSavepointForNestedTransaction()) {
            // 保存savePoint
            DefaultTransactionStatus status =
                    prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
            status.createAndHoldSavepoint();
            return status;
        }
        else {
            // 嵌套事务发生在begin、commit、rollback时候
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, null);
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
    }

    // 如果传播行为是 PROPAGATION_SUPPORTS 或者 PROPAGATION_REQUIRED.
    if (debugEnabled) {
        logger.debug("Participating in existing transaction");
    }
    // 检查现在事务是否可用
    if (isValidateExistingTransaction()) {
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
            Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
            if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
                Constants isoConstants = DefaultTransactionDefinition.constants;
                throw new IllegalTransactionStateException("Participating transaction with definition [" +
                        definition + "] specifies isolation level which is incompatible with existing transaction: " +
                        (currentIsolationLevel != null ?
                                isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
                                "(unknown)"));
            }
        }
        if (!definition.isReadOnly()) {
            if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
                throw new IllegalTransactionStateException("Participating transaction with definition [" +
                        definition + "] is not marked as read-only but existing transaction is");
            }
        }
    }
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
```

## doBegin

回到`getTransaction`方法，如果当前没有事务，且传播行为是`PROPAGATION_REQUIRED`、`PROPAGATION_REQUIRES_NEW`、`PROPAGATION_NESTED`其中一种，则新建一个事务并执行。

```java
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;

    try {
        if (txObject.getConnectionHolder() == null || txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            // 从dataSource中获取一个新连接
            Connection newCon = this.dataSource.getConnection();
            if (logger.isDebugEnabled()) {
                logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
            }
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }

        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        con = txObject.getConnectionHolder().getConnection();

        Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
        txObject.setPreviousIsolationLevel(previousIsolationLevel);

        // 将自动提交设置为false
        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            if (logger.isDebugEnabled()) {
                logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
            }
            con.setAutoCommit(false);
        }

        prepareTransactionalConnection(con, definition);
        txObject.getConnectionHolder().setTransactionActive(true);

        int timeout = determineTimeout(definition);
        if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
        }

        // 将连接和当前线程绑定
        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
        }
    }

    catch (Throwable ex) {
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, this.dataSource);
            txObject.setConnectionHolder(null, false);
        }
        throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
    }
}
```

## commit

回到`execute`方法，获取事务状态后调用`doInTransaction`执行用户的代码。

如果用户的代码没有抛出异常，则调用`AbstractPlatformTransactionManager.commit`提交事务，

```java
public final void commit(TransactionStatus status) throws TransactionException {
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException(
                "Transaction is already completed - do not call commit or rollback more than once per transaction");
    }

    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    if (defStatus.isLocalRollbackOnly()) {
        if (defStatus.isDebug()) {
            logger.debug("Transactional code has requested rollback");
        }
        processRollback(defStatus);
        return;
    }
    if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
        if (defStatus.isDebug()) {
            logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
        }
        processRollback(defStatus);
        // Throw UnexpectedRollbackException only at outermost transaction boundary
        // or if explicitly asked to.
        if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
            throw new UnexpectedRollbackException(
                    "Transaction rolled back because it has been marked as rollback-only");
        }
        return;
    }
    // 如果设置了回滚则调用processRollback，否则调用doCommit提交事务
    processCommit(defStatus);
}
```

## rollbackOnException

如果用户的代码抛出异常，则调用`rollbackOnException` -> `processRollback` -> `doRollback`回滚，


