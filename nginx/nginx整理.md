---
title: nginx整理
date: 2017/12/22 11:17:00
---

## 内置预定义变量

- `$args`：这个变量等于GET请求中的参数。例如foo=123&bar=456，这个变量只能被修改
- `$host`：请求中的主机头(HOST)字段，如果请求中的主机头不可用或者空，则为处理请求的server名称(处理请求的server的server_name指令的值)。值为小写，不包含端口
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

## `proxy_set_header`

- `proxy_set_header X-real-ip $remote_addr;`

	经过反向代理后，由于在客户端和web服务器之间增加了中间层，因此web服务器无法直接拿到客户端的ip，通过`$remote_addr`变量拿到的将是反向代理服务器的ip地址。但是，nginx是可以获得用户的真实ip的，也就是说nginx使用`$remote_addr`变量时获得的是用户的真实ip，如果我们想要在web端获得用户的真实ip，就必须在nginx这里作一个赋值操作，如下：
	
	```
	proxy_set_header X-real-ip $remote_addr;
	```
	
	其中这个`X-real-ip`是一个自定义的变量名，名字可以随意取，这样做完之后，用户的真实ip就被放在`X-real-ip`这个变量里了，然后在web端可以这样获取：
	
	```
	request.getAttribute("X-real-ip")
	```

- `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`

	`X-Forwarded-For`变量用于识别通过HTTP代理或负载平衡器原始IP这个连接到web服务器的客户机地址的非rfc标准，如果有做`X-Forwarded-For`设置的话，每次经过proxy转发都会有记录，格式就是`client1,proxy1,proxy2`，以逗号隔开各个地址，由于它是非rfc标准，所以默认是没有的，需要强制添加，在默认情况下经过proxy转发的请求，在后端看来远程地址都是proxy端的ip。也就是说在默认情况下我们使用`request.getAttribute("X-Forwarded-For")`获取不到用户的ip，如果我们想要通过这个变量获得用户的ip，我们需要自己在nginx添加如下配置：
	
	```
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	```
	
	意思是增加一个`$proxy_add_x_forwarded_for`到`X-Forwarded-For`里去，注意是增加，而不是覆盖，当然由于默认的`X-Forwarded-For`值是空的，所以我们总感觉到`X-Forwarded-For`的值等于`$proxy_add_x_forwarded_for`的值，实际上当你搭建两台nginx在不同ip上，并且都使用了这段配置，那你会发现在web服务器端通过`request.getAttribute("X-Forwarded-For")`获得的将会是客户端ip和第一台nginx的ip。
	
	- `$proxy_add_x_forwarded_for`

		`$proxy_add_x_forwarded_for`变量包含客户端请求头中的`X-Forwarded-For`与`$remote_addr`两部分，他们之间用逗号分开。
		
		举个例子，有一个web应用，在它之前通过了两个nginx转发，即用户访问该web通过两台nginx。
		
		在第一台`nginx`中，使用`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`。现在的`$proxy_add_x_forwarded_for`变量的`X-Forwarded-For`部分是空的，所以只有`$remote_addr`，而`$remote_addr`的值是用户的ip，于是赋值以后，`X-Forwarded-For`变量的值就是用户的真实ip地址了。
		
		到了第二台nginx，使用`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`。现在的`$proxy_add_x_forwarded_for`变量，`X-Forwarded-For`部分包含的是用户的真实ip，`$remote_addr`部分的值是上一台nginx的ip地址，于是通过这个赋值以后现在的`X-Forwarded-For`的值就变成了"用户的真实ip，第一台nginx的ip"。
		
	- `$http_x_forwarded_for`变量，这个变量就是`X-Forwarded-For`，由于之前我们说了，默认的这个`X-Forwarded-For`是为空的，所以当我们直接使用`proxy_set_header X-Forwarded-For $http_x_forwarded_for`时会发现，web服务器端使用`request.getAttribute("X-Forwarded-For")`获得的值是null。如果想要通过`request.getAttribute("X-Forwarded-For")`获得用户ip，就必须先使用`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`，这样就可以获得用户真实ip
	
> http://gong1208.iteye.com/blog/1559835

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

