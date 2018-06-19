---
title: InputStream转成Byte数组
date: 2018/03/12 18:13:00
---

本文介绍三种将InputStream转成Byte数组的方法：

## 只使用JDK

### 固定大小的stream

```java
InputStream initialStream = new ByteArrayInputStream(new byte[] {0, 1, 2});
byte[] targetArray = new byte[initialStream.available()];
initialStream.read(targetArray);
```

### 不固定大小的stream

```java
InputStream is = new ByteArrayInputStream(new byte[] {0, 1, 2});
ByteArrayOutputStream buffer = new ByteArrayOutputStream();
int nRead;
byte[] data = new byte[1024];
while ((nRead = is.read(data, 0, data.length)) != -1) {
    buffer.write(data, 0, nRead);
}
buffer.flush();
byte[] byteArray = buffer.toByteArray();
```

## 使用Guava

```java
InputStream initialStream = ByteSource.wrap(new byte[] {0, 1, 2}).openStream();
byte[] targetArray = ByteStreams.toByteArray(initialStream);
```

## 使用Commons IO

```java
ByteArrayInputStream initialStream = new ByteArrayInputStream(new byte[] {0, 1, 2});
byte[] targetArray = IOUtils.toByteArray(initialStream);
```

> http://www.baeldung.com/convert-input-stream-to-array-of-bytes

