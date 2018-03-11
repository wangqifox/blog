---
title: 树莓派搭建aria2下载机
date: 2018/03/11 14:40:00
---

aria2是一款开源免费跨平台且不限速的多线程下载软件，其优点是速度超级快、体积轻盈、性能强劲、资源占用少；支持HTTP/FTP/BT/Magnet磁力链接等类型文件的下载。

## 使用`apt-get`来安装：

```
sudo apt-get install aria2
```

## 配置aria2

创建`/home/pi/.aria2/aria2.conf`和`/home/pi/.aria2/aria2.session`两个文件。

在`aria2.conf`加入以下配置：

```
# 基本配置
# 下载目录
dir=/mnt/disk
# 下载从这个文件中找到的 urls, 需自己建立这个文件
# touch /home/pi/.aria2/aria2.session
input-file=/home/pi/.aria2/aria2.session
# 最大同时下载任务数，默认 5
max-concurrent-downloads=3
# 断点续传，只适用于 HTTP(S)/FTP
continue=true

# HTTP/FTP 配置
# 关闭连接如果下载速度等于或低于这个值，默认 0
lowest-speed-limit=0
# 对于每个下载在同一个服务器上的连接数，默认 5
max-connection-per-server=5
# 每个文件最小分片大小，例如文件 20M，设置 size 为 10M, 则用2个连接下载，默认 20M
min-split-size=10M
# 下载一个文件的连接数，默认 5
split=5

# BT 特殊配置
# 启用本地节点查找，默认 false
bt-enable-lpd=true
# 指定最大文件数对于每个 bt 下载，默认 100
bt-max-open-files=100
# 单种子最大连接数，默认 55
bt-max-peers=55
# 设置最低的加密级别，可选全连接加密 arc4，默认是头加密 plain
bt-min-crypto-level=plain
# 总是使用 obfuscation handshake，防迅雷必备，默认 false
bt-require-crypto=true
# 如果下载的是种子文件则自动解析并下载，默认 true
follow-torrent=true
# 为 BT 下载设置 TCP 端口号，确保开放这些端口，默认 6881-6999
listen-port=6881-6999
# 整体上传速度限制，0 表示不限制，默认 0
max-overall-upload-limit=0
# 每个下载上传速度限制，默认 0
max-upload-limit=0
# 种子分享率大于1, 则停止做种，默认 1.0
seed-ratio=1
# 做种时间大于2小时，则停止做种
seed-time=120

# RPC 配置
# 开启 JSON-RPC/XML-RPC 服务，默认 false
enable-rpc=true
# 允许所有来源，web 界面跨域权限需要，默认 false
rpc-allow-origin-all=true
# 允许外部访问，默认 false
rpc-listen-all=true
# rpc 端口，默认 6800
rpc-listen-port=6800
# 设置最大的 JSON-RPC/XML-RPC 请求大小，默认 2M
rpc-max-request-size=2M
# rpc 密码，可不设置
rpc-passwd=
# rpc 用户名，可不设置
rpc-user=

# 高级配置
# This is useful if you have to use broken DNS and
# want to avoid terribly slow AAAA record lookup.
# 默认 false
disable-ipv6=true
# 指定文件分配方法，预分配能有效降低文件碎片，提高磁盘性能，缺点是预分配时间稍长
# 如果使用新的文件系统，例如 ext4 (with extents support), btrfs, xfs or NTFS(MinGW build only), falloc 是最好的选择
# 如果设置为 none，那么不预先分配文件空间，默认 prealloc
file-allocation=falloc
# 整体下载速度限制，默认 0
max-overall-download-limit=0
# 每个下载下载速度限制，默认 0
max-download-limit=0
# 保存错误或者未完成的下载到这个文件
# 和基本配置中的 input-file 一起使用，那么重启后仍可继续下载
save-session=/home/pi/.aria2/aria2.session
# 每5分钟自动保存错误或未完成的下载，如果为 0, 只有 aria2 正常退出才回保存，默认 0
save-session-interval=300

# 若要用于 PT 下载，需另外的配置，这里没写
```

## 测试配置是否有错误

```
aria2c --conf-path=/home/pi/.aria2/aria2.conf
```

## 添加aria2脚本

新建`sudo vim /etc/init.d/aria2`文件，输入以下内容

```
#!/bin/sh
### BEGIN INIT INFO
# Provides:          aria2
# Required-Start:    $remote_fs $network
# Required-Stop:     $remote_fs $network
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Aria2 Downloader
### END INIT INFO

USER=pi
CONF=/home/pi/.aria2/aria2.conf

case "$1" in
start)
    echo "Start aria2c"
    umask 0002
    su - $USER -c "aria2c --conf-path=$CONF -D"
    ;;
stop)
    echo "Stopping aria2c, please wait..."
    killall -w aria2c
    ;;
restart)
    echo "Stopping aria2c, please wait..."
    killall -w aria2c
    echo "Start aria2c"
    umask 0002
    su - $USER -c "aria2c --conf-path=$CONF -D"
    ;;
*)
    echo "$0 {start|stop|restart|status}"
    ;;
esac
exit
```

执行`sudo chmod 755 /etc/init.d/aria2`给aria2脚本赋予可执行权限

## 安装yaaw

aria2只能使用命令操作，使用不太方便。可以考虑安装一个管理aria2的前端页面`yaaw`。

`yaaw`的github地址：[https://github.com/binux/yaaw](https://github.com/binux/yaaw)

使用很简单，将代码clone下来，将nginx配置为静态页面的服务

```
server {
    listen      8888;
    server_name localhost;

    location / {
        root   /home/pi/yaaw;
        index  index.html index.htm;
    }
}
```

访问`http://ip-of-raspberrypi:8888`即可看到yaaw的界面

