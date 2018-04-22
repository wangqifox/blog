---
title: MyBatis探究(一)——使用
date: 2018/04/10 20:30:00
---

MyBatis是一款优秀的持久层框架，它支持定制化SQL、存储过程以及高级映射。MyBatis避免了几乎所有的JDBC代码和手动设置参数以及获取结果集。MyBatis可以使用简单的XML或注解来配置和映射原生信息，将接口和Java的POJOs(Plain Old Java Objects，普通的Java对象)映射成数据库中的记录。

MyBatis探究将是一个系列。从这篇文章开始，我将开始探究MyBatis的相关技术，包括：

- MyBatis的使用
- MyBatis的源码分析
- MyBatis与Spring的结合
- MyBatis与Spring boot的结合
<!-- more -->
## MyBatis使用

MyBatis以SqlSessionFactory为中心，SqlSessionFactory的实例通过SqlSessionFactoryBuilder获得。而SqlSessionFactoryBuilder可以从XML配置文件或一个预先定制的Configuration的实例构建出SqlSessionFactory的实例。

### 构建SqlSessionFactory

#### 使用XML方式构建

如下的代码从XML文件中构建SqlSessionFactory：

```java
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

XML配置文件中包含了对MyBatis系统的核心设置，包含获取数据库连接实例的数据源(DataSource)和决定事务作用域和控制方式的事务管理器(TransactionManager)。

XML文件的示例(mybatis-config.xml)如下：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!--定义数据库信息，默认使用development数据库构建环境-->
    <environments default="development">
        <environment id="development">
            <!--采用jdbc事务管理-->
            <transactionManager type="JDBC"/>
            <!--配置数据库连接信息-->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://127.0.0.1:3306/demo2"/>
                <property name="username" value="root"/>
                <property name="password" value="rootroot"/>
            </dataSource>
        </environment>
    </environments>
    <!--定义映射器-->
    <mappers>
        <mapper resource="love/wangqi/dao/xml/UserMapper.xml"/>
    </mappers>
</configuration>
```

#### 使用代码方式构建

除了使用XML配置的方式创建代码外，也可以使用Java编码来实现，不过这种方法并不推荐，因为这种方式会导致配置与编码耦合，不利于维护。

我们看看之前的XML配置怎么样表述成代码方式：

```java
// 构建数据库连接池
PooledDataSource dataSource = new PooledDataSource();
dataSource.setDriver("com.mysql.cj.jdbc.Driver");
dataSource.setUrl("jdbc:mysql://127.0.0.1:3306/demo2");
dataSource.setUsername("root");
dataSource.setPassword("rootroot");
// 构建数据库事务方式
TransactionFactory transactionFactory = new JdbcTransactionFactory();
// 创建数据库运行环境
Environment environment = new Environment("development", transactionFactory, dataSource);
// 构建Configuration对象
Configuration configuration = new Configuration(environment);
// 加入映射器
configuration.addMappers("love.wangqi.dao");
sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
```

可以看到代码配置方式和XML方式没有本质区别。要注意的是，采用这种方式：Mapper类与Mapper的xml映射文件必须防止在同一个package下，否则mybatis会找不到xml映射文件。

### 相关定义

假设我们的user表的结构如下：

```
CREATE TABLE `user` (
  `id` int(10) NOT NULL AUTO_INCREMENT,
  `user_name` varchar(10) DEFAULT NULL,
  `age` int(10) DEFAULT NULL,
  `password` varchar(10) DEFAULT NULL,
  `sex` tinyint(1) NOT NULL DEFAULT '1',
  PRIMARY KEY (`id`)
)
```

User的POJO定义如下：

```java
public class User implements Serializable {
    private Integer id;
    private String userName;
    private Integer age;
    private String password;
    private Integer sex;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public Integer getSex() {
        return sex;
    }

    public void setSex(Integer sex) {
        this.sex = sex;
    }
}
```

映射器UserMapper定义如下：

```java
public interface UserMapper {
    User selectUser(@Param("pk") Integer pk);
    void insetUser(User user);
}
```

UserMapper的sql映射文件(UserMapper.xml)如下：

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="love.wangqi.dao.UserMapper">
    <resultMap id="usermap" type="love.wangqi.domain.User">
        <result column="id" property="id"/>
        <result column="user_name" property="userName"/>
        <result column="age" property="age"/>
        <result column="password" property="password"/>
        <result column="sex" property="sex"/>
    </resultMap>

    <cache/>

    <select id="selectUser" resultMap="usermap">
        select * from user where id = #{id}
    </select>

    <insert id="insetUser" useGeneratedKeys="true" keyProperty="id">
        insert into user (user_name, age, password, sex) values (#{userName}, #{age}, #{password}, #{sex})
    </insert>
</mapper>
```

映射器是由Java接口和XML文件(或注解)共同组成的，它的作用如下：

- 定义参数类型
- 描述缓存
- 描述SQL语句
- 定义查询结果和POJO的映射关系

### SqlSession

有了上面的定义，我们再来看看如何使用mybatis来查询数据和插入数据：

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
    UserMapper userMapper = session.getMapper(UserMapper.class);
    User user = userMapper.selectUser(pk);
}

try (SqlSession session = sqlSessionFactory.openSession()) {
    UserMapper userMapper = session.getMapper(UserMapper.class);
    userMapper.insetUser(user);
    session.commit();
}
```

可以看到，在定义好mapper和sql映射文件之后，mybatis的使用就非常便捷：

1. 首先通过SqlSessionFactory获得SqlSession的实例。
2. 通过SqlSession获得映射器实例(Mapper)
3. 执行映射器实例中的自定义方法

SqlSession接口类似于一个JDBC的Connection接口对象，我们需要保证每次用完正常关闭它。

数据库事务MyBatis是交由SqlSession去控制的，我们可以通过SqlSession提交(commit)或者回滚(rollback)。


