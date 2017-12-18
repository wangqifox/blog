---
title: spring事务
---

## Spring @Transactionl的传播行为和隔离级别

### 事务注解方式 @Transactional

- 标注在类前：标示类中所有方法都进行事务处理
- 标注在接口、实现类的方法前：标示方法进行事务处理

### 事务传播行为介绍

|事务|说明|
|---|---|
|@Transactional(propagation=Propagation.REQUIRED|如果有事务，那么加入事务，没有的话新建一个(默认情况)|
|@Transactional(propagation=Propagation.NOT_SUPPORTED|容器不为这个方法开启事务|
|@Transactional(propagation=Propagation.REQUIRES_NEW|不管是否存在事务，都创建一个新的事务，原来的挂起，新的执行完毕，继续执行老的事务|
|@Transactional(propagation=Propagation.MANDATORY|必须在一个已有的事务中执行，否则抛出异常|
|@Transactional(propagation=Propagation.NEVER|必须在一个没有的事务中执行，否则抛出异常(与Propagation.MANDATORY相反)|
|@Transactional(propagation=Propagation.SUPPORTS|如果其他bean调用这个方法，在其他bean中声明事务，那就用事务。如果其他bean没有声明事务，那就不用事务|

<!--more-->

### 事务超时设置

@Transactional(timeout=30)	//默认是30秒

### 事务隔离级别

|事务隔离级别|说明|
|----------|----|
|@Transactional(isolation=Isolation.READ_UNCOMMITTED)|读取未提交数据(会出现脏读，不可重复读)，基本不使用|
|@Transactional(isolation=Isolation.READ_COMMITED|读取已提交数据(会出现不可重复读和幻读)|
|@Transactional(isolation=Isolation.REPEATABLE_READ|可重复读(会出现幻读)|
|@Transactional(isolation=Isolation.SERIALIZABLE|串行化|

- 脏读：一个事务读取到另一个事务未提交的更新数据
- 不可重复读：在同一事务中，多次读取同一数据返回的结果有所不同，换句话说，后续读取可以读到另一事务已提交的更新数据。相反，"可重复读"在同一事务中多次读取数据时，能够保证所读数据一样，也就是后续读取不能读到另一事务已提交的更新数据
- 幻读：一个事务读到另一个事务已提交的insert数据

### @Transactional的属性

|属性|类型|描述|
|---|---|----|
|value|String|可选的限定描述符，指定使用的事务管理器|
|propagation|enum:Propagation|可选的事务传播行为设置|
|isolation|enum:Isolation|可选的事务隔离级别设置|
|readOnly|boolean|读写或只读事务，默认读写|
|timeout|int(in seconds granularity)|事务超时时间设置|
|rollbackFor|Class对象数组，必须继承自Throwable|导致事务回滚的异常类数组|
|rollbackForClassName|类名数组，必须继承自Throwable|导致事务回滚的异常类名字数组|
|noRollbackFor|Class对象数组，必须继承自Throwable|不会导致事务回滚的异常类数组|
|noRollbackForClassName|类名数组，必须继承自Throwable|不会导致事务回滚的异常类名字数组|

### @Transactional的使用建议

- Spring团队的建议是你在具体的类(或类的方法)上使用@Transactional注解，而不要使用在类所要实现的任何接口上。你当然可以在接口上使用@Transactional注解，但是这将只能当你设置了基于接口的代理时它才生效。因为注解是不能继承的，这就意味着如果你正在使用基于类的代理时，那么事务的设置将不能被基于类的代理所识别，而且对象也将不会被事务代理所包装(将被确认为严重的)。因此，请接受Spring团队的建议并且在具体的类或方法上使用@Transactional注解。
- @Transactional注解标识的方法，处理过程尽量的简单。尤其是带锁的事务方法，能不放在事务里面的最好不要放在事务里面。可以将常规的数据库查询操作放在事务前面进行，而事务内进行增、删、改、加锁查询等操作。
- @Transactional注解的默认事务管理器bean是"transactionManager"，如果生命为其他名称的事务管理器，需要在方法上添加@Transactional("managerName")
- @Transactional注解标注的方法中不要出现网络调用、比较耗时的处理程序，因为事务中数据库连接是不会释放的，如果每个事务的处理时间都非常长，那么宝贵的数据库连接资源将很快被耗尽


> http://tech.lede.com/2017/02/06/rd/server/SpringTransactional/


