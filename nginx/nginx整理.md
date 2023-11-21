---
title: nginx整理
date: 2017/12/22 11:17:00
updated: 2020/12/03 19:48:00
---

## 内置预定义变量

- `$args`：这个变量等于GET请求中的参数。例如foo=123&bar=456，这个变量只能被修改
- `$host`：请求中的主机头(HOST)字段，如果请求中的主机头不可用或者空，则为处理请求的server名称(处理请求的server的server_name指令的值)。值为小写，不包含端口
- `$http_host`: 原始请求的host
- `$http_port`: 原始请求的port
- `$remote_addr`：客户端的IP地址
- `$remote_port`：客户端的端口
- `$request_uri`：这个变量等于包含一些客户端请求参数的原始URI，它无法修改，请查看$uri更改或重写URI
- `$scheme`：所用的协议，比如http或者是https，比如`rewrite ^(.+)$ $scheme://example.com$1 redirect;`
- `$server_addr`：服务器地址，在完成一次系统调用后可以确定这个值，如果要绕开系统调用，则必须在listen中指定地址并且使用bind参数
- `$server_name`：服务器名称
- `$server_port`：请求到达服务器的端口号
- `$server_protocol`：请求使用的协议，通常是HTTP/1.0或HTTP/1.1
- `$uri`：请求中的当前URI(不带请求参数，参数位于`$args`)，不同于浏览器传递的`$request_uri`的值，它可以通过内部重定向，或者使用index指令进行修改。不包含协议和主机名，例如`/foo/bar.html`

<!--more-->

## `proxy_set_header`设置Host为`$proxy_host`,`$host`,`$local_host`的区别

`proxy_set_header`允许重新定义或者添加发往后端服务器的请求头。当且仅当当前配置级别中没有定义`proxy_set_header`指令时，会从上面的级别继承配置。默认情况下，只有两个请求头会被重新定义：

```
proxy_set_header	Host		$proxy_host;
proxy_set_header	Connection	close;
```

nginx对于upstream默认使用的是基于IP的转发，因此对于以下配置：

```
upstream backend {  
    server 127.0.0.1:8080;  
}  
upstream crmtest {  
    server crmtest.aty.sohuno.com;  
}  
server {  
        listen       80;  
        server_name  chuan.aty.sohuno.com;  
        proxy_set_header Host $http_host;  
        proxy_set_header x-forwarded-for  $remote_addr;  
        proxy_buffer_size         64k;  
        proxy_buffers             32 64k;  
        charset utf-8;  
  
        access_log  logs/host.access.log  main;  
        location = /50x.html {  
            root   html;  
        }  
    location / {  
        proxy_pass backend ;  
    }  
          
    location = /customer/straightcustomer/download {  
        proxy_pass http://crmtest;  
        proxy_set_header Host $proxy_host;  
    }  
}  
```

当匹配到`/customer/straightcustomer/download`时，使用crmtest处理，到upstream就匹配到`crmtest.aty.sohuno.com`，这里直接转换成IP进行转发了。假如`crmtest.aty.sohuno.com`是在另一台nginx下配置的，ip为`10.22.10.116`，则`$proxy_host`则对应为`10.22.10.116`。此时相当于设置了Host为`10.22.10.116`。如果想让Host是`crmtest.aty.sohuno.com`，则进行如下设置:

`proxy_set_header Host crmtest.aty.sohuno.com;`

如果不想改变请求头"Host"的值，可以这样来设置：

`proxy_set_header Host $http_host;`

但是，如果客户端请求头中没有携带这个头部，那么传递到后端服务器的请求也不含这个头部。这种情况下，更好的方式是使用`$host`变量——它的值在请求包含"Host"请求头时为"Host"字段的值，在请求未携带"Host"请求头时为虚拟主机的主域名:

`proxy_set_header Host $host;`

此外，服务器名可以和后端服务器的端口一起传送：

`proxy_set_header Host $host:$proxy_port;`

如果某个请求头的值为空，那么这个请求头将不会传送给后端服务器：

`proxy_set_header Accept-Encoding "";`

> http://liuluo129.iteye.com/blog/1943311

## `proxy_set_header`其他设置

### `proxy_set_header X-real-ip $remote_addr;`

经过反向代理后，由于在客户端和web服务器之间增加了中间层，因此web服务器无法直接拿到客户端的ip，通过`$remote_addr`变量拿到的将是反向代理服务器的ip地址。但是，nginx是可以获得用户的真实ip的，也就是说nginx使用`$remote_addr`变量时获得的是用户的真实ip，如果我们想要在web端获得用户的真实ip，就必须在nginx这里作一个赋值操作，如下：
	
```
proxy_set_header X-real-ip $remote_addr;
```
	
其中这个`X-real-ip`是一个自定义的变量名，名字可以随意取，这样做完之后，用户的真实ip就被放在`X-real-ip`这个变量里了，然后在web端可以这样获取：
	
```
request.getAttribute("X-real-ip")
```

### `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`

`X-Forwarded-For`变量用于识别通过HTTP代理或负载平衡器原始IP这个连接到web服务器的客户机地址的非rfc标准，如果有做`X-Forwarded-For`设置的话，每次经过proxy转发都会有记录，格式就是`client1,proxy1,proxy2`，以逗号隔开各个地址，由于它是非rfc标准，所以默认是没有的，需要强制添加，在默认情况下经过proxy转发的请求，在后端看来远程地址都是proxy端的ip。也就是说在默认情况下我们使用`request.getAttribute("X-Forwarded-For")`获取不到用户的ip，如果我们想要通过这个变量获得用户的ip，我们需要自己在nginx添加如下配置：
	
```
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```
	
意思是增加一个`$proxy_add_x_forwarded_for`到`X-Forwarded-For`里去，注意是增加，而不是覆盖，当然由于默认的`X-Forwarded-For`值是空的，所以我们总感觉到`X-Forwarded-For`的值等于`$proxy_add_x_forwarded_for`的值，实际上当你搭建两台nginx在不同ip上，并且都使用了这段配置，那你会发现在web服务器端通过`request.getAttribute("X-Forwarded-For")`获得的将会是客户端ip和第一台nginx的ip。
	
#### `$proxy_add_x_forwarded_for`

`$proxy_add_x_forwarded_for`变量包含客户端请求头中的`X-Forwarded-For`与`$remote_addr`两部分，他们之间用逗号分开。
	
举个例子，有一个web应用，在它之前通过了两个nginx转发，即用户访问该web通过两台nginx。
	
在第一台`nginx`中，使用`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`。现在的`$proxy_add_x_forwarded_for`变量的`X-Forwarded-For`部分是空的，所以只有`$remote_addr`，而`$remote_addr`的值是用户的ip，于是赋值以后，`X-Forwarded-For`变量的值就是用户的真实ip地址了。
	
到了第二台nginx，使用`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`。现在的`$proxy_add_x_forwarded_for`变量，`X-Forwarded-For`部分包含的是用户的真实ip，`$remote_addr`部分的值是上一台nginx的ip地址，于是通过这个赋值以后现在的`X-Forwarded-For`的值就变成了"用户的真实ip，第一台nginx的ip"。
	
#### `$http_x_forwarded_for`

这个变量就是`X-Forwarded-For`，由于之前我们说了，默认的这个`X-Forwarded-For`是为空的，所以当我们直接使用`proxy_set_header X-Forwarded-For $http_x_forwarded_for`时会发现，web服务器端使用`request.getAttribute("X-Forwarded-For")`获得的值是null。如果想要通过`request.getAttribute("X-Forwarded-For")`获得用户ip，就必须先使用`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`，这样就可以获得用户真实ip

> http://gong1208.iteye.com/blog/1559835

## location表达式的优先级

location的语法规则如下：

```
location [ = | ~ | ~* | ^~ ] uri { ... }
location @name { ... }
```

语法规则很简单，一个`location`关键字，后面跟着可选的修饰符，后面是要匹配的字符，花括号中是要执行的操作。

### 修饰符以及匹配优先级

1. `=`：表示精确匹配。只有请求的url路径与后面的字符串完全相等时，才会命中。
2. `^~`：表示普通字符串匹配。使用前缀匹配。如果匹配成功，则不再匹配其他location。
3. `~*`：表示该规则是使用正则定义的，不区分大小写。
4. `~`：表示该规则是使用正则定义的，区分大小写。
5. 无修饰符：表示普通路径的前缀匹配。

### 匹配过程

首先对请求的url序列化。例如，对`%xx`等字符进行解码，去除url中多个相连的`/`，解析url中的`.`，`..`等。这一步是匹配的前置工作。

location有两种表示形式，一种是使用前缀字符，一种是使用正则。如果是正则的话，前面有`~`或`~*`修饰符。

具体的匹配过程如下：

1. 首先先检查使用前缀字符定义的location，选择最长匹配的项并记录下来。
2. 如果找到了精确匹配的location，也就是使用了`=`修饰符的location，结束查找，使用它的配置。
3. 然后按顺序查找使用正则定义的location，如果匹配则停止查找，使用它定义的配置。
4. 如果没有匹配的正则location，则使用前面记录的最长匹配前缀字符location

基于以上的匹配过程，我们可以知道：

1. 使用正则定义的location在配置文件中出现的顺序很重要。因为找到第一个匹配的正则后，查找就停止了，后面定义的正则就是再匹配也没有机会了。
2. 使用精确匹配可以提高查找的速度。例如经常请求`/`的话，可以使用`=`来定义location。

### 示例

假设有下面这段配置文件：

```
location = / {
    [ configuration A ]
}

location / {
    [ configuration B ]
}

location /user/ {
    [ configuration C ]
}

location ~* /images/.* {
    [ configuration D ]
}

location ~* \.(git|jpg|jpeg)$ {
    [ configuration E ]
}
```

- 请求`/`精准匹配A，不再往下查找。
- 请求`/index.html`匹配B。首先查找匹配的前缀字符，找到最长匹配是配置B，接着又按照顺序查找匹配的正则。结果没有找到，因此使用先前标记的最长匹配，即配置B。
- 请求`/user/index.html`匹配C。首先找到最长匹配C，由于后面没有匹配的正则，所以使用最长匹配C。
- 请求`/user/1.jpg`匹配E。首先进行前缀字符的查找，找到最长匹配项C，继续进行正则查找，找到匹配项E，因此使用E。
- 请求`/images/1.jpg`匹配D。首先找到最长匹配B，接着进行正则的匹配，由于D的顺序在E之前，匹配到D之后就结束，因此使用D。
- 请求`/test`匹配B。因此B表示任何以`/`开头的url都匹配。其他规则都不匹配，因此使用B。


如果在配置文件中添加一个F规则：

```
location = / {
    [ configuration A ]
}

location / {
    [ configuration B ]
}

location /user/ {
    [ configuration C ]
}

location ~* /images/.* {
    [ configuration D ]
}

location ~* \.(git|jpg|jpeg)$ {
    [ configuration E ]
}

location ^~ /images/ {
    [ configuration F ]
}
```

- 请求`/images/1.jpg`匹配F。因为`^~`修饰符的优先级高于正则，匹配到F之后不再匹配其他规则。

### `@name`的用法

`@`用来定义一个命名location。主要用于内部重定向，不能用来处理正常的请求。其用法如下：

```
location / {
    try_files $uri $uri/ @custom
}
location @custom {
    # ...do something
}
```

上例中，当尝试访问url找不到对应的文件就重定向到我们自定义的命名location（此处为custom）。

值的注意的是，命名location中不能再嵌套其他的命名location。


> https://www.cnblogs.com/zjfjava/p/10760157.html
> https://segmentfault.com/a/1190000013267839
> https://blog.csdn.net/qq_31772441/article/details/107072564
> https://segmentfault.com/a/1190000012829367

## nginx配置proxy_pass代理转发

在nginx配置`proxy_pass`代理转发时，如果在`proxy_pass`后面的url加/，表示相对路径，把匹配的路径部分也给代理走。

假设下面四种情况分别用`http://192.168.1.1/proxy/test.html`进行访问。

1. 第一种

	```
	location /proxy/ {
		proxy_pass http://127.0.0.1/;
	}
	```
	代理到URL：`http://127.0.0.1/test.html`
	
2. 第二种

	```
	location /proxy/ {
		proxy_pass http://127.0.0.1;
	}
	```
	代理到URL：`http://127.0.0.1/proxy/test.html`

3. 第三种

	```
	location /proxy/ {
		proxy_pass http://127.0.0.1/aaa/;
	}
	```
	代理到URL：`http://127.0.0.1/aaa/test.html`

4. 第四种

	```
	location /proxy/ {
		proxy_pass http://127.0.0.1/aaa;
	}
	```
	代理到URL：`http://127.0.0.1/aaatest.html`

> http://blog.csdn.net/PHPService/article/details/48803235

## alias与root区别

```
location /i/ {
	root /data/w3;
}

location /i/ {
	alias /data/w3/images/;
}
```

当访问/i/top.gif时，root是去/data/w3/i/top.git请求文件，alias是去/data/w3/images/top.gif请求文件

- root响应的路径：配置的路径+完整访问路径(完整的location配置路径+静态文件)
- alias响应的路径：配置的路径+静态文件(去除location中配置的路径)

## proxy_redirect

`proxy_redirect http:// $scheme://` 表示在程序中有redirect跳转时，将采用原有传输协议方式跳转，即如果是以https请求，在跳转后依然是https


## 配置WebSocket反向代理

nginx的配置默认情况下不支持WebSocket，需要额外配置才能支持WebSocket。

当客户端发起WebSocket请求时，会首先尝试建立连接，此时使用的请求地址并不是普通的`http://`或`https://`开头的地址，而是以`ws://`（未经TLS加密，与HTTP对应）或`wss://`（经TLS加密，与HTTPS对应）开头的地址。

以`ws://example.com/websocket`为例，请求头如下：


```
GET /websocket HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Version: 13
```


该请求头与普通的HTTP请求头非常类似，除了多几个字段：

- Upgrade：必须为`websocket`，表示需要升级协议为WebSocket进行通讯
- Connection：必须为`Upgrade`，表示需要升级连接
- Sec-WebSocket-Key：必须为随机字符串，用于握手验证，服务器也会返回一个类似的字符串响应头：


```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```


经过这样的握手，双方就可以建立WebSocket连接，进行实时双向通讯了。

nginx反向代理WebSocket的话，需要明确地添加`Upgrade`和`Connection`头：


```
# 如果没有Upgrade头，则$connection_upgrade为close，否则为upgrade
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    ...
    location /websocket {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        # 下面这两行是关键
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }
}
```


通过以上配置，nginx就可以正常代理WebSocket请求了。

如果有多个后端服务器，则可以使用`upstream`定义多个后端服务器，并在`location`中使用`proxy_pass`指定后端服务器即可：


```
upstream backend {
    192.168.3.1:3000;
    192.168.3.2:300;
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    ...
    location /websocket {
        proxy_pass http://upstream;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        # 下面这两行是关键
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }
}
```








> https://tutorials.tinkink.net/zh-hans/nginx/nginx-websocket-reverse-proxy.html



