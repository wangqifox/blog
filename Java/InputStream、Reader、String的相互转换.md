---
title: InputStream、Reader、String的相互转换
date: 2018/10/15 20:23:00
---

InputStream、Reader、String的相互转换是我们经常会遇到的情况，在此记录一下以备之后查阅。

<!-- more -->

## String -> InputStream

```java
String s = "Hello, World!!";
InputStream input = new ByteArrayInputStream(s.getBytes());
```

## InputStream -> String

### InputStream.read

```java
StringBuilder out = new StringBuilder();
byte[] b = new byte[4096];
for (int n; (n = inputStream.read(b)) != -1;) {
    out.append(new String(b, 0, n));
}
String s = out.toString();
```

### Stream API

```java
BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
String s = bufferedReader.lines().collect(Collectors.joining());
```

### Scanner

```java
Scanner scanner = new Scanner(inputStream).useDelimiter("\\A");
String s = scanner.hasNext() ? scanner.next() : "";
```

### IOUtils (Apache Utils)

```java
String s = IOUtils.toString(inputStream, StandardCharsets.UTF_8);
```

### CharStreams (Guava)

```java
String s = CharStreams.toString(new InputStreamReader(inputStream, Charsets.UTF_8));
```

## InputStream -> Reader

```java
Reader reader = new InputStreamReader(inputStream);
```

## String -> Reader

```java
String s = "Hello, World!!";
Reader reader = new StringReader(s);
```

## Reader -> String

```java
BufferedReader bufferedReader = new BufferedReader(reader);
StringBuilder stringBuilder = new StringBuilder();
String line;
while ((line = bufferedReader.readLine()) != null) {
    stringBuilder.append(line);
}
String s = stringBuilder.toString();
```

```java
BufferedReader bufferedReader = new BufferedReader(reader);
String s = bufferedReader.lines().collect(Collectors.joining());
```














> https://hype.codes/how-convert-inputstream-string-java

