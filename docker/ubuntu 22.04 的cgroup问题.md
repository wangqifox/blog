---
title: ubuntu 22.04 的cgroup问题
date: 2022/12/06 22:16:00
---

最近在ubuntu 22.04上安装docker 19.03之后发现docker启动容器会报错：`cgroups: cgroup mountpoint does not exist: unknown.`

<!-- more -->

经过网上查询：

1. [https://github.com/docker/for-linux/issues/219](https://github.com/docker/for-linux/issues/219)
2. [https://www.php5.idv.tw/wordpress/?p=1143](https://www.php5.idv.tw/wordpress/?p=1143)

可以执行一下命令来解决问题：

```Bash
mkdir /sys/fs/cgroup/systemd
mount -t cgroup -o none,name=systemd cgroup /sys/fs/cgroup/systemd

```



其实该问题是根本原因是ubuntu 22.04上的cgroup的版本改变了。

ubuntu从21.10开始默认使用cgroup v2（[https://www.oschina.net/news/155959/ubuntu-21-10-willsupport-cgroupsv2](https://www.oschina.net/news/155959/ubuntu-21-10-willsupport-cgroupsv2)）。

而docker从20.10版本才开始支持cgroup v2（[https://cloud-atlas.readthedocs.io/zh_CN/latest/docker/moby/docker_20.10.html](https://cloud-atlas.readthedocs.io/zh_CN/latest/docker/moby/docker_20.10.html)），因此docker19.03在ubuntu 22.04工作不正常。



解决办法要么升级docker版本，要么将系统的cgroup版本切换到v1：

1. 调整grub linux内核引导参数：

```Bash
vim /etc/default/grub
```
2. 在GRUB_CMDLINE_LINUX一行添加：

```Bash
systemd.unified_cgroup_hierarchy=0
```
3. 更新grub

```Bash
update-grub
reboot
```

