---
title: 复制bean属性兼谈本地缓存的作用
date: 2019/02/12 15:15:00
---

在我们后端开发的过程中，对于数据的存储和处理一般采用这样的一个套路：首先有一个Bean和数据库中的数据相对应，另外有多个对应的DTO和前端数据进行通信。这样可以保证数据库的字段保持稳定，和前端的数据通信格式可以灵活改变。
<!-- more -->
一般来说DTO中的字段是Bean中字段的一部分，以达到不同场景下隐藏一些敏感数据的目的。

举例说明：

```java
public class Comment {
    /**
     * id
     */
    private String id;
    /**
     * 稿件标题
     */
    private String articleTitle;
    /**
     * 稿件id
     */
    private String articleId;
    /**
     * 稿件url
     */
    private String articleUrl;
    /**
     * 评论内容
     */
    private String content;
    /**
     * 分类id
     */
    private String categoryId;
    /**
     * 分类名称
     */
    private String categoryName;
    /**
     * 评论用户id
     */
    private String commentUserId;
    /**
     * 评论用户名称
     */
    private String commentUserName;
    /**
     * 点赞数
     */
    private Integer likeCount;
    /**
     * 楼层号
     */
    private Integer floorNum;
    /**
     * 评论创建日期
     */
    private LocalDateTime createdAt;
    /**
     * 评论更新日期
     */
    private LocalDateTime updatedAt;
    
    // 省略setter和getter操作...
}


public class CommentDTO {
    /**
     * id
     */
    private String id;
    /**
     * 评论内容
     */
    private String content;
    /**
     * 点赞数
     */
    private Integer likeCount;
    /**
     * 评论创建日期
     */
    private LocalDateTime createdAt;
    
    // 省略setter和getter操作...
}
```

`Comment`是我们评论对应的Bean，`CommentDTO`是`Comment`对应的DTO。可以看到，`CommentDTO`中的字段是`Comment`字段的子集。

这种情况下，将Bean中的数据复制到DTO中或者将DTO中的数据复制到Bean中是相对机械的操作，我们当然可以将这个操作写成固定的代码用以调用。

## 初始代码

```java
public class CopyDirect {

    public static void copy(Object source, Object dest) throws Exception {
        // 使用Introspector获取Bean信息
        BeanInfo sourceBean = Introspector.getBeanInfo(source.getClass(), Object.class);
        // 获取PropertyDescriptor数组，PropertyDescriptor中包含各个字段的名称、类型、读方法、写方法等信息
        PropertyDescriptor[] sourcePropertyDescriptors = sourceBean.getPropertyDescriptors();

        BeanInfo destBean = Introspector.getBeanInfo(dest.getClass(), Object.class);
        PropertyDescriptor[] destPropertyDescriptors = destBean.getPropertyDescriptors();

        // 将PropertyDescriptor数组转化成字段名称和PropertyDescriptor对应的Map，便于遍历
        Map<String, PropertyDescriptor> sourcePropertyMap = Arrays.stream(sourcePropertyDescriptors).collect(Collectors.toMap(FeatureDescriptor::getName, p -> p));
        Map<String, PropertyDescriptor> destPropertyMap = Arrays.stream(destPropertyDescriptors).collect(Collectors.toMap(FeatureDescriptor::getName, p -> p));

        for (Map.Entry<String, PropertyDescriptor> entry : destPropertyMap.entrySet()) {
            String desProperty = entry.getKey();
            PropertyDescriptor desPropertyDescriptor = entry.getValue();
            // 从source对象中获取相应字段对应的PropertyDescriptor
            PropertyDescriptor sourcePropertyDescriptor = sourcePropertyMap.get(desProperty);
            if (sourcePropertyDescriptor != null) {
                Object sourceValue = sourcePropertyDescriptor.getReadMethod().invoke(source);
                // 如果source中的字段可以赋值给dest中的字段，并且source中的字段不为null。调用dest中字段的写方法设置该字段的值。
                if (desPropertyDescriptor.getPropertyType().isAssignableFrom(sourcePropertyDescriptor.getPropertyType()) && sourceValue != null) {
                    desPropertyDescriptor.getWriteMethod().invoke(dest, sourceValue);
                }
            }
        }
    }

}
```

如上所示，这个版本的属性复制功能是我参考了网上的代码写的。

基本逻辑很简单：

1. 使用`Introspector`获取bean信息
2. 遍历`dest`中的字段，在`source`中寻找同名的字段，将`source`中字段的值赋值给`dest`中的字段

使用如上的代码是可以完成属性复制功能的。但是在实际的项目中，对接口进行压力测试总是不尽如人意、经过定位没想到居然是这样一个简单的属性复制的代码成为了性能的瓶颈。我们需要对其进行优化。

## Spring中的属性复制

Spring中其实也提供了一个属性复制的方法：

```java
BeanUtils.copyProperties(Object source, Object target)
```

为了说明其功能，简化一下Spring的`copyProperties`代码：

```java
public class CopyCache {

    public static void copy(Object source, Object target) throws Exception {
        Class<?> actualEditable = target.getClass();

        PropertyDescriptor[] targetPds = getPropertyDescriptors(actualEditable);

        for (PropertyDescriptor targetPd : targetPds) {
            Method writeMethod = targetPd.getWriteMethod();
            if (writeMethod != null) {
                PropertyDescriptor sourcePd = getPropertyDescriptor(source.getClass(), targetPd.getName());
                if (sourcePd != null) {
                    Method readMethod = sourcePd.getReadMethod();
                    if (readMethod != null && isAssignable(writeMethod.getParameterTypes()[0], readMethod.getReturnType())) {
                        try {
                            if (!Modifier.isPublic(readMethod.getDeclaringClass().getModifiers())) {
                                readMethod.setAccessible(true);
                            }
                            Object value = readMethod.invoke(source);
                            if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers())) {
                                writeMethod.setAccessible(true);
                            }
                            writeMethod.invoke(target, value);
                        }
                        catch (Throwable ex) {
                            ex.printStackTrace();
                        }
                    }
                }
            }
        }
    }

    public static boolean isAssignable(Class<?> lhsType, Class<?> rhsType) {
        return lhsType.isAssignableFrom(rhsType);
    }

    public static PropertyDescriptor[] getPropertyDescriptors(Class<?> clazz) throws Exception {
        CachedIntrospectionResults cr = CachedIntrospectionResults.forClass(clazz);
        return cr.getPropertyDescriptors();
    }

    public static PropertyDescriptor getPropertyDescriptor(Class<?> clazz, String propertyName) throws Exception {
        CachedIntrospectionResults cr = CachedIntrospectionResults.forClass(clazz);
        return cr.getPropertyDescriptor(propertyName);
    }
}

public class CachedIntrospectionResults {
    /**
     * 类信息的缓存
     */
    static final ConcurrentMap<Class<?>, CachedIntrospectionResults> classCache = new ConcurrentHashMap<>(64);
    /**
     * 类字段信息的缓存
     */
    private final Map<String, PropertyDescriptor> propertyDescriptorCache;

    private static BeanInfo getBeanInfo(Class<?> beanClass) throws IntrospectionException {
        return Introspector.getBeanInfo(beanClass);
    }

    private CachedIntrospectionResults(Class<?> beanClass) throws Exception {
        BeanInfo beanInfo = getBeanInfo(beanClass);
        PropertyDescriptor[] pds = beanInfo.getPropertyDescriptors();

        this.propertyDescriptorCache = new LinkedHashMap<>();
        for (PropertyDescriptor pd : pds) {
            this.propertyDescriptorCache.put(pd.getName(), pd);
        }
    }

    static CachedIntrospectionResults forClass(Class<?> beanClass) throws Exception {
        CachedIntrospectionResults results = classCache.get(beanClass);
        if (results != null) {
            return results;
        }

        results = new CachedIntrospectionResults(beanClass);
        CachedIntrospectionResults existing = classCache.putIfAbsent(beanClass, results);
        return (existing != null ? existing : results);
    }

    PropertyDescriptor getPropertyDescriptor(String name) {
        return this.propertyDescriptorCache.get(name);
    }

    PropertyDescriptor[] getPropertyDescriptors() {
        PropertyDescriptor[] pds = new PropertyDescriptor[this.propertyDescriptorCache.size()];
        int i = 0;
        for (PropertyDescriptor pd : this.propertyDescriptorCache.values()) {
            pds[i] = pd;
            i++;
        }
        return pds;
    }
}
```

基本的逻辑并不复杂，主要的不同在于`CachedIntrospectionResults`类。这是一个缓存类，其中有两个变量：`classCache`用于缓存类信息，`propertyDescriptorCache`用户字段与字段信息的缓存。

当缓存存在时，类的字段信息都是从缓存中获取的，这样就可以节省了从Bean中获取字段信息的时间。实践证明从Bean中获取字段信息的操作是十分耗时的。

## 性能对比

为了直观地说明缓存的作用，我们通过下面的测试代码来对比使用缓存和不使用缓存两种情况的耗时：

```java
public class PerformanceTest {
    static abstract class Run implements Runnable {
        Comment comment;
        int loop = 100;

        public Run(Comment comment) {
            this.comment = comment;
        }
    }

    static class RunDirect extends Run {

        public RunDirect(Comment comment) {
            super(comment);
        }

        @Override
        public void run() {
            for (int i = 0; i < loop; i++) {
                CommentDTO newComment = new CommentDTO();
                try {
                    CopyDirect.copy(comment, newComment);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }

    static class RunCache extends Run {

        public RunCache(Comment comment) {
            super(comment);
        }

        @Override
        public void run() {
            for (int i = 0; i < loop; i++) {
                CommentDTO newComment = new CommentDTO();
                try {
                    CopyCache.copy(comment, newComment);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void test(Class<? extends Run> runClass) throws Exception {
        Comment comment = new Comment();
        comment.setId("123");
        comment.setArticleTitle("articleTitle");
        comment.setArticleId("1234");
        comment.setArticleUrl("http://127.0.0.1");
        comment.setContent("评论内容");
        comment.setCategoryId("123456");
        comment.setCategoryName("新闻");
        comment.setCommentUserId("1234567");
        comment.setCommentUserName("测试");
        comment.setLikeCount(123);
        comment.setFloorNum(123);
        comment.setCreatedAt(LocalDateTime.now());
        comment.setUpdatedAt(LocalDateTime.now());

        List<Thread> threadList = new ArrayList<>();
        for (int i = 0; i < 300; i++) {
            Constructor<? extends Run> constructor = runClass.getConstructor(Comment.class);
            Run run = constructor.newInstance(comment);
            threadList.add(new Thread(run));
        }

        long start = System.currentTimeMillis();
        for (Thread thread : threadList) {
            thread.start();
        }

        for (Thread thread : threadList) {
            thread.join();
        }

        long end = System.currentTimeMillis();

        System.out.println("end - start: " + (end - start) + "ms");
        System.out.println("per copy cost: " + (end - start) / 100 * 300 * 1.0 + "ms");
    }

    public static void testDirect() throws Exception {
        test(RunDirect.class);
    }

    public static void testCache() throws Exception {
        test(RunCache.class);
    }

    public static void main(String[] args) throws Exception {
        System.out.println("testDirect");
        testDirect();
        System.out.println();
        System.out.println("testCache");
        testCache();
    }
}
```

启动300个线程，每个线程循环执行100次属性复制。执行结果如下：

```
testDirect
end - start: 27231ms
per copy cost: 0.9077ms

testCache
end - start: 129ms
per copy cost: 0.0043ms
```

可以看到，使用缓存和不使用缓存两种情况的性能差距达到上百倍。不使用缓存的情况每次属性复制要将近1ms，如果每个接口属性复制的操作执行的次数比较多，耗时还是非常可观的。

## 总结

提到缓存，我们一般想到的是`memcache`和`redis`这类的缓存服务。但是进程内本地缓存的作用在某些情况下的作用也是非常巨大的，对于提升程序的性能有时候响应巨大。




























