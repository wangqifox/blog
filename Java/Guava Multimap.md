---
title: Guava Multimap
date: 2018/06/19 16:15:00
---

有时候我们需要这样的数据类型Map<String, Collection<String>>，guava中的`Multimap`就是为了解决这类问题的。

`Multimap`提供了丰富的实现，所以你可以用它来替代程序里的Map<K, Collection<V>>，具体的实现如下：

|实现|Key实现|Value实现|
|---|---|---|
|ArrayListMultiamp|HashMap|ArrayList|
|HashMultimap|HashMap|HashSet|
|LinkedListMultimap|LinkedHashMap|LinkedList|
|LinkedHashMultimap|LinkedHashMap|LinkedHashSet|
|TreeMultimap|TreeMap|TreeSet|
|ImmutableListMultimap|ImmutableMap|ImmutableList|
|ImmutableSetMultimap|ImmutableMap|ImmutableSet|

上述`Multimap`的实现都是线程不安全的，如果想要创建线程安全的`Multimap`需要调用`Multimaps.synchronizedMultimap(Multimap<K, V> multimap)`方法将multimap包装成线程安全的版本。



> http://outofmemory.cn/java/guava/Collections/Multimaps
> 
> https://blog.csdn.net/qq_34310242/article/details/76218320

