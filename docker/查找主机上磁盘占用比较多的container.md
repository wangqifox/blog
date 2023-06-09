---
title: 查找主机上磁盘占用比较多的container
date: 2022/03/31 08:16:00
---

很多时候在kubernetes运行过程中， 存在某个pod因为一些原因造成磁盘的使用率比较高， 我们应该如何找到pod，并进行磁盘的清理呢？

因为kubernetes没有收集磁盘相关的数据， 我们只能ssh到宿主机上查找对应的container

<!-- more -->


## 查看正在运行容器的大小

使用docker系统df命令，我们可以得到一个docker使用的总结信息，包括以下内容:

- 所有Images的总大小
- 所有Containers的总大小
- 本地卷大小
- 和缓存



```Bash
$ docker system df
TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              95                  92                  12.29GB             2.514GB (20%)
Containers          188                 184                 54.79GB             0B (0%)
Local Volumes       0                   0                   0B                  0B
Build Cache         0                   0                   0B                  0B
```

通过上文的输出可以看出， 目前Image的存储使用了12G， containers的存储使用了54G。

默认情况下，如果运行`docker Image`，只能得到每个Image的大小，我们可以运行`docker ps`加上`--size`获得到正在运行容器的大小。



```Bash
$ docker ps --size
315be0878bba        nginx       "/bin/sh -c 'echo \"j…"   24 hours ago        Up 24 hours                             k8s_nginx-df8b4b58d-lx7w6_pre_84f02139-1970-4663-9bb8-dfd613717d80_0                   173MB (virtual 1.07GB)
......................................

```




> https://www.jianshu.com/p/86dc44de906e