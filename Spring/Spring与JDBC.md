---
title: Spring与JDBC
date: 2018/01/18 14:17:00
---

我们知道，如果在程序中直接使用JDBC访问数据的话，需要写一大堆与业务逻辑无关的代码。例如，我们都需要获取一个到数据存储的连接并在处理完成后释放资源。这都是在数据访问处理过程中的固定步骤，但是每种数据访问方法又会有些不同。我们会查询不同的对象或不同的方法更新数据，这都是数据访问过程中变化的部分。

Spring将数据访问的过程中固定的和可变的部分明确划分为两个不同的类：模板(template)和回调(callback)。Spring的模板类处理数据访问的固定部分——事务控制、资源管理、处理异常。同时，应用程序相关的数据访问——语句、绑定参数以及整理结果集——在回调的实现中处理。
<!-- more -->
## 配置数据源

无论选择Spring的哪种数据访问方式，你都需要配置一个数据源的引用。Spring提供了在Spring上下文中配置数据源bean的多种方法，包括：

- 通过JDBC驱动程序定义的数据源
- 通过JNDI查找的数据源
- 连接池的数据源

### 使用数据源连接池

尽管Spring并没有提供数据源连接池实现，但是我们有多项可用的方案，包括如下开源的实现：

- Apache Commons DBCP
- c3p0
- BoneCP

这些连接池中的大多数都能配置为Spring的数据源，在一定程序上与Spring自带的DriverManagerDataSource或SingleConnectionDataSource很类似。

配置DBCP BasicDataSource的方式：

```java
BasicDataSource ds = new BasicDataSource();
ds.setDriverClassName("com.mysql.jdbc.Driver");
ds.setUrl("jdbc:mysql://localhost:3306/demo");
ds.setUsername("root");
ds.setPassword("root");
ds.setInitialSize(5);
```

配置c3p0 ComboPooledDataSource的方式：

```java
ComboPooledDataSource ds = new ComboPooledDataSource();
ds.setDriverClass("com.mysql.jdbc.Driver");
ds.setJdbcUrl("jdbc:mysql://localhost:3306/demo");
ds.setUser("root");
ds.setPassword("root");
ds.setMaxStatements(180);
```

### 基于JDBC驱动的数据源

在Spring中，通过JDBC驱动定义数据源是最简单的配置方式。Spring提供了三个这样的数据源类(均位于`org.springframework.jdbc.datasource`包中)供选择：

- `DriverManagerDataSource`：在每个连接请求时都会返回一个新建的连接。与DBCP的`BasicDataSource`不同，由`DriverManagerDataSource`提供的连接并没有进行池化管理
- `SimpleDriverDataSource`：与`DriverManagerDataSource`的工作方式类似，但是它直接使用JDBC驱动，来解决在特定环境下的类加载问题，这样的环境包括OSGI容器
- `SingleConnectionDataSource`：在每个连接请求时都会返回同一个的连接。尽管`SingleConnectionDataSource`不是严格意义上的连接池数据源，但是你可以将其视为只有一个连接的池。

如下就是配置`DriverManagerDataSource`的方法：

```java
DriverManagerDataSource ds = new DriverManagerDataSource();
ds.setDriverClassName("com.mysql.jdbc.Driver");
ds.setUrl("jdbc:mysql://localhost:3306/demo");
ds.setUsername("root");
ds.setPassword("root");
```

## 使用JDBC

持久化技术有很多种，而Hibernate、iBATIS和JPA只是其中的几种而已。尽管如此，还是有很多的应用程序使用最古老的方式将Java对象保存到数据库中。

JDBC不要求我们掌握其他框架的查询语言。它是建立在SQL之上的，而SQL本身就是数据访问语言。此外，与其他的技术相比，使用JDBC能够更好地对数据访问的性能进行调优。JDBC允许你使用数据库的所有特性，而这是其他框架不鼓励甚至禁止的。

再者，相对于持久层框架，JDBC能够让我们在更低的层次上处理数据，我们可以完全控制应用程序如何读取和管理数据，包括访问和管理数据库的单例的列。这种细粒度的数据访问方式在很多应用程序中是很方便的。例如在报表应用中，如果将数据组织为对象，而接下来唯一要做的就是将其解包为原始数据，那就没有太大意义了。

如果使用JDBC所提供的直接操作数据库的API，你需要负责处理与数据库访问相关的所有事情，其中包括管理数据库资源和处理异常。

以查询一个表中所有的数据为例：

```java
public List<User> findAll() {
    Connection conn = null;
    PreparedStatement stmt = null;
    ResultSet rs = null;
    try {
        conn = dataSource.getConnection();
        stmt = conn.prepareStatement("select * from user");
        rs = stmt.executeQuery();
        List<User> userList = new ArrayList<>();
        while (rs.next()) {
            User user = new User();
            user.setId(rs.getString("id"));
            user.setUserName(rs.getString("user_name"));
            user.setAge(rs.getInt("age"));
            user.setPassword(rs.getString("password"));
            userList.add(user);
        }
        return userList;
    } catch (SQLException e) {
        e.printStackTrace();
    } finally {
        if (rs != null) {
            try {
                rs.close();
            } catch (SQLException e) {}
        }
        if (stmt != null) {
            try {
                stmt.close();
            } catch (SQLException e) {}
        }
        if (conn != null) {
            try {
                conn.close();
            } catch (SQLException e) {}
        }
    }
    return Collections.emptyList();
}
```

我们可以看到，大量的JDBC代码都是用于创建连接和语句以及异常处理的样板代码，只有20%的代码是真正用于查询数据的。

但实际上，这些样板代码是非常重要的。清理资源和处理错误确保了数据访问的健壮性。如果没有它们的话，就不会发现错误而且资源也会处于打开的状态，这将导致意外的代码和资源泄露。我们不仅需要这些代码，而且还要保证它是正确的。基于这样的原因，我们才需要框架来保证这些代码只写一次而且是正确的。

## 使用JDBC模板

Spring的JDBC框架承担了资源管理和异常处理的工作，从而简化了JDBC代码，让我们只需编写从数据库读写数据的必须代码。

Spring为JDBC提供了三个模板类供选择：

- `JdbcTemplate`：最基本的Spring JDBC模板，这个模板支持简单的JDBC数据库访问功能以及基于索引参数的查询
- `NamedParameterJdbcTemplate`：使用该模板类执行查询时可以将值以命名参数的形式绑定到SQL中，而不是使用简单的索引参数
- `SimpleJdbcTemplate`：该模板类利用Java 5的一些特性如自动装箱、泛型以及可变参数列表来简化JDBC模板的使用

从Spring 3.1开始，`SimpleJdbcTemplate`已经被废弃了，其Java 5的特性被转移到了`JdbcTemplate`中，并且只有你需要使用命名参数的时候，才需要使用`NamedParameterJdbcTemplate`。

### 使用JdbcTemplate来插入数据

为了让JdbcTemplate正常工作，只需要为其设置DataSource就可以了，这使得在Spring中配置JdbcTemplate非常容易：

```java
@Bean
public JdbcTemplate jdbcTemplate(DataSource dataSource) {
    return new JdbcTemplate(dataSource);
}
```

在这里，DataSource是通过构造器参数注入进来的。这里所引用的dataSource可以是javax.sql.DataSource的任意实现，包括前面提到的`BasicDataSource`、`ComboPooledDataSource`、`DriverManagerDataSource`。

现在，我们可以将jdbcTemplate装配到Repository中并使用它来访问数据库。

比如我们可以这样插入一条数据：

```java
@Autowired
private JdbcOperations jdbcOperations;

public void addUser(User user) {
    jdbcOperations.update("insert into user (id, user_name, age, password) values (?, ?, ?, ?)",
            user.getId(),
            user.getUserName(),
            user.getAge(),
            user.getPassword());
}
```

JdbcOperations是一个接口，定义了JdbcTemplate所实现的操作。通过注入JdbcOperations，而不是具体的JdbcTemplate，能够保证通过JdbcOperations接口达到与JdbcTemplate保持松耦合。

这里没有了创建连接和语句的代码，也没有异常处理的代码，只剩下单纯的数据插入代码。

不能因为你看不到这些样板代码，就意味着他们不存在。样板代码被巧妙地隐藏到JDBC模板类中了。当update()方法被调用的时候JdbcTemplate将会获取连接、创建语句并执行插入SQL。

在这里，你也看不到SQLException处理的代码。在内部，JdbcTemplate将会捕获所有可能抛出的SQLException，并将通用的SQLException转换为更明确的数据访问异常，然后将其重新抛出。因为Spring的数据访问异常都是运行时异常，所以我们不必在方法中进行捕获。

### 使用JdbcTemplate来查询

JdbcTemplate也简化了数据的读取操作。

```java
public User findOne(String id) {
    return jdbcOperations.queryForObject("select * from user where id = ?",
            (rs, rowNum) -> {
                User user = new User();
                user.setId(rs.getString("id"));
                user.setUserName(rs.getString("user_name"));
                user.setAge(rs.getInt("age"));
                user.setPassword(rs.getString("password"));
                return user;
            },
            id);
}
```

`findOne`使用了JdbcTemplate的回调，实现根据ID查询User，并将结果集映射为User对象。

