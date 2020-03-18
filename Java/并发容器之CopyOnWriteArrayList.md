---
title: 并发容器之CopyOnWriteArrayList
date: 2020/03/12 14:06:00
---

本文是对并发容器`CopyOnWriteArrayList`的学习。

<!-- more -->

`ArrayList`是我们很常用的容器，但它并不是线程安全的容器，多个线程同时操作一个`ArrayList`时会发生错误或者抛出异常。

举个例子，以下几种情况会发生异常：

1. 两个线程同时调用`add`方法向同一个`ArrayList`插入数据，大概率会产生`ArrayIndexOutOfBoundsException`异常
2. 一个线程调用`add`方法插入数据，另一个线程使用`Iterator`遍历列表，大概率会产生`ConcurrentModificationException`异常

如果想要在多线程环境中使用，一种办法是使用`Collections.synchronizedList()`方法将`ArrayList`包装成一个线程安全的类，它在所有的方法中都加入了`synchronized`关键字，利用独占式锁来保证同一时间只有一个线程可以操作`ArrayList`。特别需要注意的是，`Collections.synchronizedList()`方法没有为`ArrayList`的`Iterator`和`ListIterator`提供线程安全的保证，如果要使用迭代器操作`ArrayList`需要用户手动对容器进行保护。

使用`synchronized`独占锁会导致读与读操作互相影响，从而降低读操作的性能。考虑到大部分场景都是读多写少的，因此这种降低读操作性能的方式是比较不合理的。

我们还可以使用`ReentrantReadWriteLock`来对`ArrayList`加锁，使得读与读操作之间不会阻塞，可以大大提升读的性能。不过因为`ReentrantReadWriteLock`的特性，当写操作获得写锁之后，读操作仍然会被阻塞。

`CopyOnWriteArrayList`就是为了解决这个问题而被创造的。它的思路是当我们往一个容器添加元素时，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后往新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样一来，对`CopyOnWriteArrayList`进行并发读的时候就不需要加锁，因为当前容器不会添加任何元素。`CopyOnWriteArrayList`就是通过`Copy-On-Write`即写时复制的思想来通过延时更新的策略来实现数据的最终一致性，并且能够保证读线程间不阻塞。

## CopyOnWriteArrayList的实现

`CopyOnWriteArrayList`内部维护的是一个数据：

```java
private transient volatile Object[] array;
```

`array`数据被一个`volatile`修饰，它能够保证`array`变量的可见性。

`get`方法：

```java
public E get(int index) {
    return get(getArray(), index);
}

final Object[] getArray() {
    return array;
}

private E get(Object[] a, int index) {
    return (E) a[index];
}
```

可以看到，`get`方法非常简单，没有添加任何线程安全的的操作。原因是`array`保存的数据只会读取，不会进行修改。

`add`方法：

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}

final void setArray(Object[] a) {
    array = a;
}
```

可以看到，`add`方法使用了`ReentrantLock`来保证同一时刻只有一个写线程在执行，否则会出现多个拷贝数据，引发数据不一致。

主要的步骤就是拷贝一个新的、长度加1的数组，在新数组的末尾插入数据。最后将旧数据的引用指向新的数组，根据`volatile`的`happens-before`规则，写线程对数组引用的修改对读线程是可见的。

由于在写数据的时候，是在新的数组中插入数据的，从而保证读写在两个不同的数据容器中进行操作。








































> https://juejin.im/post/5aeeb55f5188256715478c21