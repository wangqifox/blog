---
title: maven整理
date: 2018/05/10 08:05:00
---

Maven命令整理
<!-- more -->

# 使用Maven运行Java main的3中方式

> https://blog.csdn.net/qbg19881206/article/details/19850857

## 从命令行运行

1. 运行前先编译代码，`exec:java`不会自动编译代码，你需要手动执行mvn compile来完成编译

    `mvn compile`

2. 编译完成后，执行`exec`运行main方法

    不需要传递参数
    
    `mvn exec:java -Dexec.mainClass="com.vineetmanohar.module.Main"`

    需要传递参数
    
    `mvn exec:java -Dexec.mainClass="com.vineetmanohar.module.Main" -Dexec.args="arg0 arg1 arg2"`

    指定对classpath的运行时依赖
    
    `mvn exec:java -Dexec.mainClass="com.vineetmanohar.module.Main" -Dexec.classpathScope=runtime`

## 在pom.xml中指定某个阶段执行

```xml
<build>  
 <plugins>  
  <plugin>  
   <groupId>org.codehaus.mojo</groupId>  
   <artifactId>exec-maven-plugin</artifactId>  
   <version>1.1.1</version>  
   <executions>  
    <execution>  
     <phase>test</phase>  
     <goals>  
      <goal>java</goal>  
     </goals>  
     <configuration>  
      <mainClass>com.vineetmanohar.module.CodeGenerator</mainClass>  
      <arguments>  
       <argument>arg0</argument>  
       <argument>arg1</argument>  
      </arguments>  
     </configuration>  
    </execution>  
   </executions>  
  </plugin>  
 </plugins>  
</build>
```

将CodeGenerator.main()方法的执行绑定到maven的test阶段，通过下面的命令可以执行main方法：

```
mvn test
```

## 在pom.xml中指定某个配置来执行

```
<profiles>  
 <profile>  
  <id>code-generator</id>  
  <build>  
   <plugins>  
    <plugin>  
     <groupId>org.codehaus.mojo</groupId>  
     <artifactId>exec-maven-plugin</artifactId>  
     <version>1.1.1</version>  
     <executions>  
      <execution>  
       <phase>test</phase>  
       <goals>  
        <goal>java</goal>  
       </goals>  
       <configuration>  
        <mainClass>com.vineetmanohar.module.CodeGenerator</mainClass>  
        <arguments>  
         <argument>arg0</argument>  
         <argument>arg1</argument>  
        </arguments>  
       </configuration>  
      </execution>  
     </executions>  
    </plugin>  
   </plugins>  
  </build>  
 </profile>  
</profiles>
```

将2中的配置用`<profile>`标签包裹后就能通过指定该配置文件来执行main方法，如下：

```
mvn test -Pcode-generator
```

通过一下命令可以获取`mvn exec`的其他配置参数说明

```
mvn exec:help -Ddetail=true -Dgoal=java
```

# spring-boot插件

## 传入参数

```
mvn spring-boot:run -Dspring-boot.run.jvmArguments="-Dserver.port=8081"
```

## 选择profile

```
mvn spring-boot:run -Dspring-boot.run.profiles=foo,bar
```

# 执行单元测试

- 运行整个单元测试

    `mvn test`

- 运行单个测试

    `mvn -Dtest=TestApp2 test`


