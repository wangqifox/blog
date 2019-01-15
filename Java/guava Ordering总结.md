---
title: guava Ordering总结
date: 2018/12/25 20:23:00
---

学习Ordering的背景是这样的：去年我在负责搜索业务开发的时候，将所有需要搜索以及需要展示的字段通通插入到ElasticSearch的索引中保存下来。这样做的好处是当Es搜索出数据之后可以直接将数据组织起来返回给前端，不再需要进行数据库的查询，减少了数据库的压力；坏处也显而易见，当数据库中的表或者需要展示的数据有变动则需要重新进行数据的索引。综合来看这样做的弊大于利。所以现在改成Es中只索引需要搜索的数据，搜索返回后用结果的id去数据库中再进行一次查询。因为id在数据库中一般都是主键或者是有唯一索引的字段，因此这样的一次查询耗费的时间是很少的，这样避免了Es中的数据频繁的要重新索引。

这样做唯一需要注意的是：数据库查询后的数据顺序要和Es中搜索出来的数据顺序保持一致，这样才能保证相关性高的数据在前面展示。

<!-- more -->

# order by field

mysql提供了一种`order by field`的方法来进行自定义排序。

打个比方，在Es中搜索评论数据，返回数据的id顺序为`2, 4, 6, 5, 3, 1`，我们可以使用语句

```
select * from comment where auto_pk in (2, 4, 6, 5, 3, 1) order by field(auto_pk, 2, 4, 6, 5, 3, 1)
```

以特定的顺序来查询评论数据。

这种方式倒也不复杂，但是将特定的排序操作放在数据库中来操作使得sql语句冗余，不利于维护。

于是我们直接在代码中来进行排序。guava的`Ordering`类的子类`ExplicitOrdering`正好满足我们的需要。在此总结一下`Ordering`类的使用。

# Ordering

`Ordering`是guava中用于增强排序功能的类。提供了许多静态方法来简化我们的排序代码。

需要说明的是对于java 8的用户来说，里面的绝大多数功能都可以使用`Stream`和`Comparator`来代替。下文我也会说明如何来进行这种代替。

## allEqual()

`allEqual()`方法将列表中所有的元素都视作相同的元素，即不改变列表中元素原来的顺序。

使用方式如下：

```java
Ordering<Object> objectOrdering = Ordering.allEqual();
numbers.sort(objectOrdering);
System.out.println(numbers);
```

如果使用Java 8，可以使用以下的代码来代替：

```java
numbers.sort((a, b) -> 0);
System.out.println(numbers);
```

## arbitrary()

`arbitrary()`将列表中所有元素随意排列。注意它并不是随机排列，而是依赖每个元素的`identityHashCode`值来进行比较。

使用方式如下：

```java
Ordering<Object> arbitrary = Ordering.arbitrary();
numbers.sort(arbitrary);
System.out.println(numbers);
```

## compare(T left, T right)

`compare(T left, T right)`按某个顺序对`left`和`right`元素比较大小。如果`left` < `right`，返回负整数；如果`left` = `right`，返回0；如果`left` > `right`，返回正整数。

## compound(Comparator<? super U> secondaryComparator)

将当前的`Ordering`和`secondaryComparator`组合成一个新的`Ordering`。

假设有一个学生列表，每个学生分别有分数(`score`)和标识(`id`)。现在要先对学生的分数进行排序，如果分数相同再对id进行排序，其代码如下所示：

```java
Ordering<Student> scoreOrder = Ordering.natural().onResultOf(Student::getScore);
Ordering<Student> idOrder = Ordering.natural().reverse().onResultOf(Student::getId);
Ordering<Student> compoundOrder = scoreOrder.compound(idOrder);
studentList.sort(compoundOrder);
```

如果使用Java 8，可以使用以下的代码来代替：

```java
thisComparator.thenComparing(secondaryComparator)
```

## compound(Iterable<? extends Comparator<? super T>> comparators)

将`comparators`中的所有比较器组合成一个新的`Ordering`。

还是按上面需求：先对学生的分数进行排序，如果分数相同再对id进行排序，其代码如下所示：

```java
Ordering<Student> scoreOrder = Ordering.natural().onResultOf(Student::getScore);
Ordering<Student> idOrder = Ordering.natural().reverse().onResultOf(Student::getId);

Ordering<Student> compoundOrder = Ordering.compound(Arrays.asList(scoreOrder, idOrder));
studentList.sort(compoundOrder);
```

如果使用Java 8，可以使用以下的代码来代替：

```java
Comparator.thenComparing(Comparator)
```

或者

```java
comparatorCollection.stream().reduce(Comparator::thenComparing).get()
```

## explicit(List<T> valuesInOrder)

`explicit(List<T> valuesInOrder)`方法按照`valuesInOrder`中的顺序对列表进行排序。

假设我们要对学生列表按照id的特定顺序进行排序，可以使用如下代码：

```java
List<Integer> idOrder = new ArrayList<>();
for (int i = 0; i < 100; i++) {
    idOrder.add(i);
}
idOrder.sort(Ordering.arbitrary());
System.out.println(idOrder);
Ordering<Student> studentOrdering = Ordering.explicit(idOrder).onResultOf(Student::getId);
ImmutableList<Student> students = studentOrdering.immutableSortedCopy(studentList);
```

## explicit(T leastValue, T... remainingValuesInOrder)

`explicit(T leastValue, T... remainingValuesInOrder)`也是将列表按特定的顺序来排序，唯一不同的是它将`leastValue`作为最小值，其他的值按`remainingValuesInOrder`的顺序来排序。示例如下：

```java
List<Integer> numbers = Arrays.asList(5, 1, 10, 9, 20);
Ordering<Integer> explicitOrder = Ordering.explicit(1, 5, 9, 10, 20);

List<Integer> list = explicitOrder.immutableSortedCopy(numbers);
System.out.println(list);
```

## from(Comparator<T> comparator)

将`comparator`包装成`Ordering`

## greatestOf(Iterable<E> iterable, int k)

按照特定的排序返回k个最大的值，示例如下：

```java
Ordering<Comparable> natural = Ordering.natural();
List<Integer> integers = natural.greatestOf(numbers, 3);
```

如果使用Java 8，可以使用以下的代码来代替：

```java
Streams.stream(iterable).collect(Comparators.greatest(k, thisComparator))
```

## immutableSortedCopy(Iterable<E> elements)

将`elements`排序之后返回一个不可变（`immutable`）的列表，`elements`不发生改变。示例如下：

```java
Ordering<Comparable> natural = Ordering.natural();
ImmutableList<Integer> integers = natural.immutableSortedCopy(numbers);
```

## isOrdered(Iterable<? extends T> iterable)

`isOrdered`方法判断`iterable`按照特定的顺序是否是有序的，及后面的元素要大于等于前面的元素。示例如下：

```java
List<Integer> num = Arrays.asList(5, 1, 1, 2, 9, 4);
Ordering<Comparable> natural = Ordering.natural();
num.sort(natural);
System.out.println(num);
System.out.println(natural.isOrdered(num));
System.out.println(natural.isStrictlyOrdered(num));
```

如果使用Java 8，可以使用以下的代码来代替：

```java
Comparators.isInOrder(Iterable, Comparator)
```

## isStrictlyOrdered(Iterable<? extends T> iterable)

`isStrictlyOrdered`方法判断`iterable`按照特定的顺序是否是严格有序的，及后面的元素要大于前面的元素。

如果使用Java 8，可以使用以下的代码来代替：

```java
Comparators.isInStrictOrder(Iterable, Comparator)
```

## leastOf(Iterable<E> iterable, int k)

按照特定的排序返回k个最小的值，示例如下：

```java
Ordering<Comparable> natural = Ordering.natural();
List<Integer> integers = natural.leastOf(numbers, 3);
```

如果使用Java 8，可以使用以下的代码来代替：

```java
Streams.stream(iterable).collect(Comparators.least(k, thisComparator))
```

## lexicographical()

将列表元素按字典序进行排序。一个典型的字典序如下所示：`[] < [1] < [1, 1] < [1, 2] < [2]`。

```java
Ordering<Iterable<Integer>> lexicographical = Ordering.natural().lexicographical();
List<List<Integer>> lists = new ArrayList<>();
lists.add(Arrays.asList());
lists.add(Arrays.asList(1));
lists.add(Arrays.asList(1, 1));
lists.add(Arrays.asList(1, 2));
lists.add(Arrays.asList(1, 2, 3));
lists.add(Arrays.asList(2));

List<List<Integer>> es = lexicographical.immutableSortedCopy(lists);
System.out.println(es);
```

如果使用Java 8，可以使用以下的代码来代替：

```java
Comparators.lexicographical(Comparator)
```

## natural()

按自然顺序对元素进行排序。

```java
Ordering<Integer> natural = Ordering.natural();
numbers.sort(natural);
System.out.println(numbers);
```

如果使用Java 8，可以使用以下的代码来代替：

```java
Comparator.naturalOrder()
```

## nullsFirst()

将所有的`null`元素都排列到列表的开头。

```java
Ordering<Comparable> nullsFirstOrdering = Ordering.natural().nullsFirst();
List<Integer> list = new ArrayList<>(numbers);
list.add(null);
list.add(null);
list.add(null);

list.sort(nullsFirstOrdering);
System.out.println(list);
```

如果使用Java 8，可以使用以下的代码来代替：

```java
Comparator.nullsFirst(thisComparator)
```

## nullsLast()

将所有的`null`元素都排列到列表的末尾。

```java
List<Integer> list = new ArrayList<>(numbers);
list.add(null);
list.add(null);
list.add(null);

Ordering<Comparable> nullsLastOrdering = Ordering.natural().nullsLast();
list.sort(nullsLastOrdering);
System.out.println(list);
```

如果使用Java 8，可以使用以下的代码来代替：

```java
Comparator.nullsLast(thisComparator)
```

## onResultOf(Function<F,? extends T> function)

将元素根据`function`返回的值进行排序。

如果使用Java 8，可以使用以下的代码来代替：

```java
Comparator.comparing(function, thisComparator)
```

## reverse()

将特定的顺序按逆序排列。

```java
Ordering<Integer> natural = Ordering.natural();
numbers.sort(natural.reverse());
System.out.println(numbers);
```

如果使用Java 8，可以使用以下的代码来代替：

```java
thisComparator.reversed() 
```

## sortedCopy(Iterable<E> elements)

将`elements`排序之后返回一个不可变（`immutable`）的列表，`elements`不发生改变。

## usingToString()

将元素按照它`toString()`方法返回的值进行排序

如果使用Java 8，可以使用以下的代码来代替：

```java
Comparator.comparing(Object::toString) 
```












> https://google.github.io/guava/releases/snapshot-jre/api/docs/

