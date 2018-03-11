---
title: 树莓派上配置samba
date: 2018/03/10 21:40:00
---

最近多买了一块硬盘，为了能更好地利用这块硬盘。我翻出了早已积灰的树莓派。配上一个带电源的硬盘底座，用来作为家庭存储。

## 准备硬盘

将硬盘通电插入树莓派的USB口。可以看到`/dev`目录下多了一个sda的设备。

- 首先将格式化硬盘:

```
sudo mkfs.ext4 /dev/sda
```

- 然后挂载硬盘

输入

```
sudo mount /dev/sda /mnt/disk
```

将硬盘挂载在/mnt/disk目录下。输入`df -h`命令可以看到刚刚挂载的硬盘

树莓派每次重启都需要对硬盘进行重新挂载，比较麻烦，因此需要自动挂载。修改`/etc/fstab`文件。在后面加入一行：

```
/dev/sda        /mnt/disk       ext4    defaults,noatime    0   0
```

表格各列的含义如下：

1. 第一列：磁盘分区名/卷标
2. 第二列：挂载点，我们这里把`/dev/sda`挂到`/mnt/disk`上
3. 第三列：该设备上的文件系统，这里是ext4
4. 第四列：文件系统的挂载选项。    

    - default：使用默认选项，rw,suid,dev,exec,auto,nouser,and async
    - noauto：当启动时给出`mount -a`命令时并不挂载
    - user：允许用户挂载
    - owner：允许设备自己挂载
    - comment：供fstab维护程序使用
    - nofail：如果这个设备不存在，不报告错误信息
    - noatime：默认linux会把文件访问的时间atime做记录，文件系统在文件被访问、创建、修改等的时候记录下文件的一些时间戳，比如文件创建时间、最近一次修改时间、最近一次访问时间，如果能减少一些动作将会显著提高磁盘IO的效率、提升文件系统的性能。noatime表示禁止记录最近一次访问时间戳

5. 第5列：0表示不做dump备份，1表示每天做dump备份，2表示不定日期进行dump操作
6. 第6列：该字段由fsck程序用于确定在重新启动文件系统检查完成的顺序，启动用的文件系统需要制定为1，其他文件系统需要指定2，如果没有此域或设置为0表示不检查。

## 安装samba

安装相关软件：

```
sudo apt-get install samba samba-common-bin
```

### 配置

修改`/etc/samba/smb.conf`文件，在`[global]`下增加：

```
load printers = no
printing = bsd
printcap name = /dev/null
disable spoolss = yes

bind interfaces only = yes
interfaces = eth0, lo

# 让samba支持软连接
follow symlinks = yes
wide links = yes
unix extensions = no
```

在尾部增加：

```
[share]
path = /mnt/disk
public = yes
writable = yes
valid users = wangqi
create mask = 0644
force create mode = 0644
directory mask = 0755
force directory mode = 0755
available = yes
```

然后执行`sudo /etc/init.d/samba restart`重启samba服务即可

### 增加samba用户

执行：

```
sudo smbpasswd -a wangqi
```

输入密码之后就在samba中增加了一个名为`wangqi`的用户

## 验证

如果是mac系统，在finder中选择`前往`->`连接服务器`，然后输入`smb://ip-of-raspberrypi`，输入用户名密码就可以看到共享的网络存储了


