---
title: Java注释
date: 2020/02/28 14:43:00
---

Java注释学习整理


<!-- more -->


## 注解的本质

所有的注解都继承自`Annotation`接口。比如`Override`：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {

}
```

它的本质就是:

```java
public interface Override extends Annotation {

}
```

因此，注解的本质就是一个继承了`Annotation`接口的接口。一个注解准确意义上来说，只不过是一种特殊的注释而已，如果没有解析它的代码，它可能连注释都不如。

而解析一个类或者方法的注解往往有两种形式，一种是编译期直接扫描，一种是运行期反射。编译期扫描指的是编译器在对java代码编译字节码的过程中检测到某个类或者方法被一些注解修饰，这时它就会对这些注解进行某些处理。

典型的就是注解`@Override`，一旦编译器检测到某个方法被修饰了`@Override`注解，编译器就会检查当前方法的方法签名是否真正重写了父类的某个方法，也就是比较父类中是否具有一个同样的方法签名。

这一种情况只适用于那些编译器已经熟知的注解类，比如JDK内置的几个注解，而你自定义的注解，编译器是不知道你这个注解的作用的，当然也不知道该如何处理，往往只是会根据该注解的作用范围来选择是否编译进字节码文件，仅此而已。


## 元注解

`元注解`是用于修饰注解的注解，通常用于在注解的定义上，一般用于指定某个注解生命周期以及作用目标等信息。

Java中有以下几个`元注解`：

- `@Target`：注解的作用目标
- `@Retention`：注解的声明周期
- `@Documented`：注解是否应当被包含在Java Doc文档中
- `@Inherited`：是否允许子类继承该注解

`@Target`用于指明被修饰的注解最终可以作用的目标是谁，也就是指明你的注解到底是用来修饰方法、修饰类、修饰字段属性的？

`@Target`的value是`ElementType`类型，有以下几个枚举值：

- `ElementType.TYPE`：允许被修饰的注解作用在类、接口和枚举上
- `ElementType.FIELD`：允许作用在属性字段上
- `ElementType.METHOD`：允许作用在方法上
- `ElementType.PARAMETER`：允许作用在方法参数上
- `ElementType.CONSTRUCTOR`：允许作用在构造器上
- `ElementType.LOCAL_VARIABLE`：允许作用在本地局部变量上
- `ElementType.ANNOTATION_TYPE`：允许作用在注解上
- `ElementType.PACKAGE`：允许作用在包上

`@Retention`用于指明当前注解的生命周期。它的value是`RetentionPolicy`类型，有以下几个枚举值：

- `RetentionPolicy.SOURCE`：当前注解编译器可见，不会写入class文件
- `RetentionPolicy.CLASS`：类加载阶段丢弃，会写入class文件
- `RetentionPolicy.RUNTIME`：永久保存，可以反射获取

## Java内置的三大注解

### `@Override`

它的定义如下：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```

没有任何属性，索引并不能存储任何其他信息。它只能作用于方法上，编译结束后将被丢弃。

###`@Deprecated`：

它的定义如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(value={CONSTRUCTOR, FIELD, LOCAL_VARIABLE, METHOD, PACKAGE, PARAMETER, TYPE})
public @interface Deprecated {
}
```

依然是一种`标记式注解`，永久存在，可以修饰所有的类型。作用是：标记当前的类或者方法或者字段等已经不再被推荐使用了，可能下一次的JDK版本就会删除。

当然，编译器并不会强制要求你做什么，只是告诉你JDK已经不再推荐使用当前的方法或者类了，建议你使用某个替代者。

### `@SuppressWarnings`

`@SuppressWarnings`主要用来压制java的警告。它的定义如下：

```java
@Target({TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE})
@Retention(RetentionPolicy.SOURCE)
public @interface SuppressWarnings {
    String[] value();
}
```

它有一个`value`属性需要你主动的传值，这个`value`代表的就是需要被压制的警告类型。













































> https://juejin.im/post/5b45bd715188251b3a1db54f#heading-2