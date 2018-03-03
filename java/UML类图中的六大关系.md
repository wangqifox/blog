---
title: UML类图中的六大关系：关联、聚合、组合、依赖、继承、实现
date: 2017/12/14 09:24:00
---

## 简介

在UML类图中，类之间的关系可以分成：关联(association)、聚合(aggregation)、组合(composition)、依赖(dependency)、泛化(generalization)/继承(inheritance)、实现(realization)。

![](http://static.oschina.net/uploads/space/2014/0420/102330_lYoU_941605.png)

上面的关系可以解读如下：

- (关联)Association：A类和B类有逻辑上的连接
- (聚合)Aggregation：A类有一个B类
- (组合)Composition：A类拥有一个B类
- (依赖)Dependency：A类使用了B类
- (继承)Inheritance：B类是一个A类（或者B类扩展A类）
- (实现)Realization：B类实现了接口A

<!--more-->

## 关联(association)

关联描述两个类之间行为的一般二元关系。例如，一个学生选修一门特定的课程是学生类Student和课程类Course之间的一个关联，而一个教师教授一门课程是师资类Faculty和课程类Course之间的一个关联。Java代码中，关联可以用属性和方法来实现。

```java
public class Student {
	private Course[] courses;
	public void addCourse(Course s) {
		...
	}
}

public class Course {
	private Student[] students;
	private Faculty faculty;
	
	public void addStudent(Student s) {
		...
	}
	
	public void setFaculty(Faculty faculty) {
		this.faculty = faculty;
	}
}

public class Faculty {
	private Course[] courses;
	
	public void addCourse(Course s) {
		...
	}
}
```

## 聚合(Aggregation)

聚合是一种特殊的关联(Association)形式，表示两个对象之间的所属(has-a)关系。所有者对象称为聚合对象，它的类称为聚合类；从属对象称为被聚合对象，它的类称为被聚合类。例如，一个公司有很多员工就是公司类Company和员工类Employee之间的一种聚合关系。被聚合对象和聚合对象有着各自的生命周期，即如果公司倒闭并不影响员工的存在。

```java
public class Company {
	private List<Employee> employee;
}

public class Employee {
	private String name;
}
```

## 组合(Composition)

聚合是一种较弱形式的对象包含(一个对象包含另一个对象)关系。较强形式是组合(Composition)。在组合关系中，包含对象负责被包含对象的创建以及生命周期，即当包含对象被销毁时被包含对象也会不复存在。例如一辆汽车拥有一个引擎是汽车类Car与引擎类Engine的组合关系。

1. 通过成员变量初始化

	```java
	public class Car {
		private final Engine engine = new Engine();
	}
	
	class Engine {
		private String type;
	}
	```

2. 通过构造函数初始化

	```java
	public class Car {
		private final Engine engine;
		
		public Car() {
			engine = new Engine();
		}
	}
	
	public class Engine {
		private String type;
	}
	```

3. 通过延迟初始化

	```java
	public class Car {
		private final Engine engine;
		public Engine getEngine() {
			if (null == engine) {
				engine = new Engine();
			}
			return engine;
		}
	}
	
	public class Engine {
		private String type;
	}
	```
	
## 依赖(Dependency)

依赖(Dependency)描述的是一个类的引用用作另一个类的方法的参数。例如，可以使用Calendar类中的setTime(Date date)方法设置日历，所以Calendar和Date之间的关系可以用依赖描述。

```java
public abstract class Calendar implements Serializable, Cloneable, Comparable<Calendar> {
	public final void setTime(Date date) {
		setTimeInMillis(date.getTime());
	}
}
```

在依赖关系中，类之间是松耦合的。

## 继承(Inheritance)

继承(Inheritance)模拟两个类之间的is-a关系。强是(strong is-a)关系描述两个类之间的直接继承关系。弱是(weak is-a)关系描述一个类具有某个属性。强是关系可以用类的继承表示。例如，Spring的ApplicationEvent是一个EventObject，ApplicationEvent和EventObject间就是一种强是关系，可以用继承描述。

```java
public abstract class ApplicationEvent extends EventObject {
	...
}
```

## 实现(Realization)

实现(Realization)描述的是一个类实现了接口（可以是多个）。上面描述的弱是(weak is-a)关系就可以用接口表示。例如字符串是可以被序列化的，这就可以用实现来描述

```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
	...
}
```

引用：

> https://my.oschina.net/jackieyeah/blog/224265

