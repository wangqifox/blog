---
title: Netplan学习
date: 2019/06/26 16:47:00
---

Netplan是ubuntu 17.10中引入的一种新的命令行配置程序。用于在ubuntu系统中管理和配置网络设置。它使用`YAML`格式的描述文件来抽象化定义网络接口的相关信息。ubuntu 18.04之后无法再通过原来的`ifupdown`工具包在`/etc/network/interfaces`文件里配置管理网络接口。

Netplan使用`NetworkManager`或`Systemd-networkd`的网络守护程序来作为内核的接口。默认描述文件在`/etc/netplan/*.yaml`里。

Netplan根据描述文件中定义的内容自动生成其对应的后端网络守护程序所需要的配置信息，后端网络守护程序再根据其配置信息通过Linux内核管理对应的网络设备。

<!-- more -->

## 使用Networkd配置网络

`Systemd-networkd`是一个管理网络设备的系统守护程序，它能检测并配置网络设备的状态和创建虚拟网络设备。

创建的配置项：

1. enp0s5：指定需配置网络接口的名称
2. dhcp4：是否打开ipv4的dhcp
3. dhcp6：是否打开ipv6的dhcp
4. addresses：定义网络接口的静态IP地址
5. gateway4：指定默认网关的ipv4地址
6. nameservers：指定域名服务器的IP地址

### 使用Networkd配置动态IP地址

修改Netplan的描述文件：

```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s5:
      dhcp4: yes
      dhcp6: yes
```

运行下面的命令使其生效：

```
sudo netplan apply
```

### 使用Networkd配置静态IP地址

```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s5:
      addresses:
      - 192.168.100.211/23
      gateway4: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
        search: []
    optional: true
```

如果要增加一个ipv6地址，可以在addresses行增加。多个地址间用逗号分隔：

```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s5:
      addresses: [192.168.100.211/23, 'fe80:0:0:0:0:0:c0a8:64d3']
      gateway4: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
        search: []
    optional: true
```

### 使用Networkd同时配置多张网卡

```
# 第一张网卡enp0s3配置为动态IP，第二张网卡enp0s5配置为静态IP
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: yes
      dhcp6: no
    enp0s5:
      dhcp4: no
      dhcp6: no
      addresses: [192.168.100.211/23]
      gateway4: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

## 使用NetworkManager配置网络

NetworkManager主要用于在桌面系统上管理网络设备。如果你使用NetworkManager作为网络设备管理的系统守护程序，将会使用NetworkManager的图形程序来管理网络接口。

修改Netplan的描述文件：

```
network:
  version: 2
  renderer: NetworkManager
```

## netplan generate

根据Netplan的描述文件手动创建网络守护程序的配置信息：

```
sudo netplan generate
```

执行后会使用`/etc/netplan/*.yaml`生成对应网络守护程序的配置信息。


































> https://www.hi-linux.com/posts/49513.html