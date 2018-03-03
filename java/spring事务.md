---
title: Spring事务
date: 2017/12/18 09:43:00
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

#### Propagation.REQUIRED

如果当前已经存在事务，那么加入该事务，如果不存在事务，创建一个事务，这是默认的传播属性值

```java
@Transactional
public void service() {
	serviceA();
	serviceB();
}

@Transactional
serviceA();
@Transactional
serviceB();
```

serviceA和serviceB都声明了事务，默认情况下，propagation=Propagation.REQUIRED，整个service调用过程中，只存在一个共享的事务，当有任何异常发生的时候，所有操作回滚。

#### Propagation.NOT_SUPPORTED

如果当前存在事务，挂起当前事务，然后新的方法在没有事务的环境中执行，没有spring事务的环境下，sql的提交完全依赖于defaultAutoCommit属性值。

```java
@Transactional
public void service() {
	serviceB();
	serviceA();
}

serviceB() {
	do sql
}

@Transactional(propagation=Propagation.NOT_SUPPORTED)
serviceA() {
	do sql 1
	1/0
	do sql 2
}
```

#### Propagation.REQUIRES_NEW

如果当前存在事务，先把当前事务相关内容封装到一个实体，然后重新创建一个新事务，接受这个实体为参数，用于事务的恢复。更直白的说法就是暂停当前事务(当前无事务则不需要)，创建一个新事务。针对这中情况，两个事务没有依赖关系，可以实现新事务回滚，但外部事务继续执行。

```java
@Transactional
public void service() {
	serviceB();
	try {
		serviceA();
	} catch(Exception e) {
	}
}

serviceB() {
	do sql
}

@Transactional(propagation=Propagation.REQUIRES_NEW)
serviceA() {
	do sql 1
	1/0
	do sql 2
}
```

当调用service接口时，由于serviceA使用的是REQUIRES_NEW，它会创建一个新的事务，但由于serviceA抛出了运行时异常，导致serviceA整个被回滚了，而在service方法中，捕获了异常，所以serviceB是正常提交的。注意，service中的try...catch代码是必须的，否则service也会抛出异常，导致serviceB也被回滚。

#### Propagation.MANDATORY

当前必须存在一个事务，否则抛出异常

```java
public void service() {
	serviceB();
	serviceA();
}

serviceB() {
	do sql
}

@Transactional(propagation=Propagation.MANDATORY)
serviceA() {
	do sql
}
```

这种情况执行service会抛出异常，如果defaultAutoCommit=true，则serviceB是不会回滚的，defaultAutoCommit=false，则serviceB执行无效。

#### Propagation.NEVER

如果当前存在事务，则抛出异常，否则在无事务环境上执行代码

```java
public void service() {
	serviceB();
	serviceA();
}
serviceB() {
	do sql
}
@Transactional(propagation=Propagation.NEVER)
serviceA() {
	do sql 1
	1/0
	do sql 2
}
```

上面的示例调用service后，若defaultAutoCommit=true，则serviceB方法及serviceA中的sql1都会生效

#### Propagation.SUPPORTS

如果当前已经存在事务，那么加入该事务，否则创建一个所谓的空事务(可以认为无事务执行)

```java
public void service() {
	serviceA();
	throw new RunTimeException();
}

@Transactional(propagation=Propagation.SUPPORTS)
serviceA();
```

serviceA执行时当前没有事务，所以service中抛出的异常不会导致serviceA回滚

```java
public void service() {
	serviceA();
}

@Transactional(propagation=Propagation.SUPPORTS)
serviceA() {
	do sql 1
	1/0;
	do sql 2;
}
```

由于serviceA运行时没有事务，这时候，如果底层数据源defaultAutoCommit=true，那么sql1是生效的，如果defaultAutoCommit=false，那么sql1无效，如果service有@Transactional标签，serviceA共用service的事务(不再依赖defaultAutoCommit)，此时，serviceA全部被回滚

#### Propagation.NESTED

如果当前存在事务，则使用SavePoint技术把当前事务状态进行保存，然后底层共用一个连接，当NESTED内部出错的时候，自行回滚到SavePoint这个状态，只要外部捕获到了异常，就可以继续进行外部的事务提交，而不会受到内嵌业务的干扰，但是，如果外部事务抛出了异常，整个大事务都会回滚。

注意：Spring配置事务管理器要主动指定nestedTransactionAllowed=true，如下所示：

```java
<bean id="dataTransactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataDataSource" />
    <property name="nestedTransactionAllowed" value="true" />
</bean>
```

看一个小例子，代码如下：

```java
@Transactional
public void service() {
	serviceA();
	try {
		serviceB();
	} catch(Exception e) {
	}
}

serviceA() {
	do sql
}

@Transactional(propagation = Propagation.NESTED)
serviceB() {
	do sql1
	1/0;
	do sql2
}
```

serviceB是一个内嵌的业务，内部抛出了运行时异常，所以serviceB整个被回滚了，由于service捕获了异常，所以serviceA是可以正常提交的。

再来看一个例子，代码如下：

```java
@Transactional
public void service() {
	serviceA();
	serviceB();
	1/0;
}

@Transactional(propagation=Propagation.NESTED)
serviceA() {
	do sql
}

serviceB() {
	do sql
}
```

由于service抛出了异常，所以会导致整个service方法被回滚。（这就是跟Propagation.REQUIRES_NEW不一样的地方了，NESTED方式下的内嵌业务会受到外部事务的异常而回滚）

### 实现浅析

本小节列举Propagation.REQUIRES_NEW和Propagation.NESTED分别进行简要说明

#### Propagation.REQUIRES_NEW实现原理

如下的代码调用：

```java
@Transactional
public void service() {
	serviceB();
	try {
		serviceA();
	} catch(Exception e) {
	}
}
@Transactional(propagation=Propagation.REQUIRES_NEW)
serviceA() {
	do sql 1
	1/0;
	do sql 2
}
serviceB() {
	do sql
}
```

执行原理：

```
before service，执行a和b
	执行serviceB
	before serviceA，执行c和d
		执行serviceA
		抛出异常
	after serviceA，执行e
after service，执行f
```

1. 创建事务状态对象，获取一个新的连接，重置连接的autoCommit, fetchSize, timeout等属性
2. 把连接绑定到ThreadLocal变量
3. 挂起当前事务，把当前事务状态对象，连接等信息封装成——SuspendedResources对象，可用于恢复
4. 创建新的事务状态对象，重新获取新的连接，重置新连接的autoCommit, fetchSize, timeout等属性，同时，保存SuspendedResources对象，用于事务的恢复，把新的连接绑定到ThreadLocal变量(覆盖操作)
5. 捕获到异常，回滚ThreadLocal中的连接，恢复连接参数，关闭连接，恢复SuspendedResources
6. 提交ThreadLocal变量中的连接(导致serviceB被提交)，还原连接参数，关闭连接，连接归还数据源，所以程序执行的结果就是serviceA被回滚了，serviceB成功提交了。

#### Propagation.NESTED实现原理

如下的代码调用：

```java
@Transactional
public void service() {
	serviceA();
	try {
		serviceB();
	} catch(Exception e) {
	}
}
serviceA() {
	do sql
}
@Transactional(propagation=Propagation.NESTED)
serviceB() {
	do sql1
	1/0;
	do sql2
}
```

执行原理：

```
before service，执行a和b
	执行serviceA
	before serviceB，执行c
		执行serviceB
		抛出异常
	after serviceB，执行d
after service，执行e
```

1. 创建事务状态对象，获取一个新的连接，重置连接的autoCommit,fetchSize,timeout等属性
2. 把连接绑定到ThreadLocal变量
3. 标记使用当前事务状态对象，获取ThreadLocal连接对象，保存当前连接的SavePoint，用于异常恢复，此时的SavePoint就是执行完serviceA后的状态
4. 捕获到异常，使用c中的SavePoint进行事务回滚，也就是把状态回滚到执行serviceA后的状态，serviceB方法所有执行不生效
5. 获取ThreadLocal中的连接对象，提交事务，恢复连接属性，关闭连接

**Spring在底层数据源的基础上，利用ThreadLocal, SavePoint等技术点实现了多种事务传播属性，便于实现各种复杂的业务。只有理解了传播属性的原理才能更好地驾驭Spring事务。Spring回滚事务依赖于对异常的捕获，默认情况下，只有抛出RuntimeException和Error才会回滚事务，当然可以进行配置。**

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
> 
> http://blog.csdn.net/yanyan19880509/article/details/53041564


