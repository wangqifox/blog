---
title: Java常用操作总结
date: 2018/03/12 18:10:00
---

Java常用操作总结，便于查询（时常更新）

<!-- more -->

## 将多个list合并成一个list

假设有一个列表，其中的元素还是列表，将其中的列表合并成一个大的列表：

```java
List<List<Integer>> list = ...
//list2是一个合并了list中所有列表的大列表
List<Integer> list2 = list.stream().flatMap(Collection::stream).collect(Collectors.toList()); 
```

## 将Stream转成Array

```java
Stream<String> stringStream = ...;
String[] stringArray = streamString.toArray(String[]::new);
```

## 将String转成InputStream

1. 不使用第三方库

```java
String initialString = "text";
InputStream targetStream = new ByteArrayInputStream(initialString.getBytes());
```

2. 使用Guava

```java
String initialString = "text";
InputStream targetStream = new ReaderInputStream(CharSource.wrap(initialString).openStream());
```

3. 使用Commons IO

```java
String initialString = "text";
InputStream targetStream = IOUtils.toInputStream(initialString);
```

## 按行读文件内容

```java
Files.readAllLines()
```

## stream代替fori

```java
IntStream.range(1, 4)
```

## stream sorted()

```java
stream.sorted(Comparator.naturalOrder())
stream.sorted(Comparator.reverseOrder())
stream.sorted(Comparator.comparingDouble())
stream.sorted(Comparator.comparingLong())
stream.sorted(Comparator.comparingInt())
stream.sorted((f1, f2) -> Long.compare(f1, f2));
```

## list to map 

```java
books.stream().collect(Collectors.toMap(Book::getIsbn, Book::getName));
```

## List<Double> 转换成 double[]

```java
List<Double> list;
list.stream().mapToDouble(Double::doubleValue).toArray()
```

## double[] 转换成 List<Double>

```java
double[] array;
DoubleStream.of(array).boxed().collect(Collectors.toList());
```






















