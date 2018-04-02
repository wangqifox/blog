---
title: ThreadLocal
date: 2018/04/01 09:48:00
---

ThreadLocal的作用是提供线程内的局部变量，这种变量在线程的生命周期内起作用，减少同一个线程内多个函数或者组件之间一些公共变量的传递的复杂度。

每个Thread的内部保存了一个类型为ThreadLocalMap的变量threadLocals。ThreadLocalMap是一个在ThreadLocal中实现的hashmap，Key是ThreadLocal，Value是保存的对象。
<!-- more -->
## get()

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

get方法的执行步骤：
1. 获得当前线程，取得当前线程中保存的ThreadLocalMap
2. 调用`map.getEntry(this)`从ThreadLocalMap中取得Entry
3. 如果Entry不为null，则从entry中获得value
4. 如果Entry为null，则调用`setInitialValue`设置初始值

## set(T value)

set的执行步骤：
1. 获得当前线程，取得当前线程中保存的ThreadLocalMap
2. 如果ThreadLocalMap不为null，调用set设置value
3. 如果ThreadLocalMap为null，调用createMap创建ThreadLocalMap

## remove()

remove方法的执行步骤
1. 首先获取当前线程的ThreadLocalMap
2. 然后调用ThreadLocalMap的remove方法将当前ThreadLocal对应的数据删除

## ThreadLocalMap的关键方法

### getEntry(ThreadLocal<?> key)

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

1. 获取key的hashcode
2. 从table中获取Entry
3. 如果entry不为null且entry保存的key和当前的key相等，返回这个entry
4. 如果在直接hash到的位置中没有找到Entry，则调用`getEntryAfterMiss(key, i, e)`

### getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e)

getEntryAfterMiss是在hash计算得到的位置中找不到数据时，寻找其他冲突位置的方法。

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

主要的流程就是循环获取下一个Entry不为null的位置的数据：

- 如果这个位置的key和给定的key相等，则返回这个位置的Entry
- 如果这个位置的key为null，则调用`expungeStaleEntry`方法将这个位置中失效的数据清除

### expungeStaleEntry(int staleSlot)

expungeStaleEntry方法擦除失效位置的数据，然后对从这个位置开始直到下一个null位置中的数据重新hash，如果在这个过程中遇到了失效的数据也会将这些失效数据清除。

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 清除失效位置的Entry
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // 重新hash直到遇到null的Entry
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 如果遇到了null的Entry则将其清除
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            // 否则根据这个Entry的hash值，将其放到另外的位置
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

### set(ThreadLocal<?> key, Object value)

set方法将key与value的组合设置到ThreadLocalMap中

```java
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

set方法的主要步骤为：

1. 首先计算key的hashcode，获取这个key的初始位置
2. 循环遍历这个key的初始位置到下一个null的位置
3. 如果某个位置的key与给定的key相等，将value设置到这个位置的Entry中
4. 如果这个位置的key为null，调用`replaceStaleEntry`方法将这个失效的位置替换给定的key和value
5. 如果遍历结束之后，仍然没有找到合适的位置：

    1. 将我们的key和value设置在这个null的位置上（位置为i）
    2. 调用`cleanSomeSlots`方法，遍历这个table，如果有失效的Entry被删除则返回`true`，否则返回`false`
    3. 如果`cleanSomeSlots`方法返回`false`，且当前table中存储的数据大于threshold，调用`rehash`方法，重新对table进行hash

### replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot)

`replaceStaleEntry`方法的功能是将失效位置的数据替换为我们给定的key和value

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // 从给定的失效位置往前检查是否还有失效的位置，将之前失效的位置赋值给slotToExpunge。因为垃圾回收机制有可能在这之前回收掉一些Entry
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    // 从给定的staleSlot开始遍历，直到下一个null位置
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // 如果找到了key，我们将它和失效的entry进行交换，以维持hash表的顺序。新的失效位置则调用expungeStaleEntry来删除或者重新计算entry的hash
        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // 如果这次循环中找不到失效的entry
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // 如果找不到key，将新的entry放到失效的位置
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // 如果有其他失效的entry，将它们擦除
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

### rehash()

```java
private void rehash() {
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)
        resize();
}
```

首先调用`expungeStaleEntries`方法擦除失效的数据，如果table中存储的数据个数大于等于`(threshold - threshold / 4)`，调用`resize`方法扩容

### expungeStaleEntries()

```java
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

`expungeStaleEntries`方法遍历table的每个位置，如果发现这个位置的数据失效，则调用`expungeStaleEntry`方法清除这个位置的数据

### resize()

`resize`方法的功能就是将table的容量扩大为原先的两倍，然后将原先的数据重新hash。

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

### remove(ThreadLocal<?> key)

```java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

remove方法的主要步骤：
1. 遍历table直到遇见一个null的entry
2. 如果找到了这个Entry，将其删除，然后擦除这个位置的Entry


## 总结

ThreadLocal的设计思路：
每个Thread维护一个ThreadLocalMap映射表，这个映射表的key是ThreadLocal实例本身，value是真正需要存储的Object。优势：

- 这样设计之后每个Map的Entry数量变少了：之前是Thread的数量，现在是ThreadLocal的数量，能提高性能
- 当Thread销毁之后对应的ThreadLocalMap也就随之销毁了，能减少内存使用量






