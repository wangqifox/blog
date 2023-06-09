---
title: 构建多架构Docker镜像
date: 2022/05/18 08:16:00
---

程序代码开发完成之后，为了运行在不同的CPU架构中，需要将代码编译成对应架构的可执行文件。

<!-- more -->

## 编译不同架构的应用

### 1 直接在目标硬件上构建

可以直接在目标硬件上编译可执行文件，比如在树莓派上运行的程序可以在树莓派中直接编译。

### 2 模拟目标硬件

QEMU是一个自由且开源的模拟器，支持许多通用架构，包括：ARM、Power-PC和RISC-V。通过运行一个全功能模拟器，我们可以启动一个可以运行Linux操作系统的通用ARM虚拟机，然后在虚拟机中设置开发环境，编译应用程序。

但是，一个全功能虚拟机有一些浪费资源。在该模式下，QEMU会模拟整个系统，包括诸如定时器、内存控制器、SPI和I2C总线控制器等硬件。但是大部分情况下，我们编译应用程序不会关心以上所提到的硬件特性。

### 3 通过binfmt_misc模拟目标架构的用户空间

在Linux系统上，QEMU有另外一种操作模式，可以通过用户模式模拟器来运行非本地架构的二进制程序。该模式下，QEMU会跳过方法2中描述的对整个目标系统硬件的模拟，取而代之的是通过binfmt_misc在Linux内核注册一个二进制格式处理程序，将陌生二进制代码拦截并转换后再执行，同时将系统调用按需从目标系统转换成当前系统。最终对于用户来说，他们会发现可以在本机运行这些异构二进制程序。

通过用户态模拟器和QEMU，我们可以通过轻量级虚拟化来安装其他Linux发行版，并像在本地一样编译我们需要的异构二进制程序。

这将会是构建多架构Docker镜像的可选方式。

### 4 使用交叉编译器

交叉编译器是一个特殊的编译器，它运行在主机架构上，但是可以为不同的目标架构生成二进制程序。

从性能上考虑，这种方式拥有和直接在目标硬件上构建相同的效率，因为它没有运行在模拟器上。但是教程编译的变数取决于使用的编程语言。

## 构建多架构Docker镜像

不仅是应用程序，Docker镜像也是区分架构的。当我们引入Docker镜像的时候，不仅仅是关于构建单独的二进制文件，而是构建一整个异构容器镜像。

所幸，Docker可以利用Docker扩展：buildx来构建多架构镜像。

buildx能够使用由Moby BuildKit提供的构建镜像额外特性，它能够创建多个builder实例，在多个节点并行地执行构建任务，以及跨平台构建。

### 步骤1：开启buildx

确认Docker运行环境是19.03之后的版本。新版本中，buildx事实上已经默认和Docker捆绑在一起，但是需要通过设置环境变量DOCKER_CLI_EXPERIMENTAL来开启。

```Bash
export DOCKER_CLI_EXPERIMENTAL=enabled

```

或者编辑`~/.docker/config.json`文件，加入以下内容：

```Bash
"experimental": "enabled"
```

通过检查版本来验证目前我们已经可以使用buildx

```Bash
docker buildx version
```

![docker多架构-1](media/docker%E5%A4%9A%E6%9E%B6%E6%9E%84-1.png)

### 步骤2：开启binfmt_misc来运行非本地架构的Docker镜像

如果使用的是Mac或者Windows版本的Docker桌面版，可以跳过这个步骤，因为binfmt_misc默认开启。

如果使用Linux系统，需要设置binfmt_misc。现在可以通过运行一个特权Docker容器来更方便的设置：

```Bash
docker run --rm --privileged docker/binfmt:66f9012c56a8316f9244ffd7622d7c21c1f6f28d

```

通过检查QEMU处理程序来验证binfmt_misc设置是否正确：

```Bash
ls -al /proc/sys/fs/binfmt_misc/
```

![docker多架构-2](media/docker%E5%A4%9A%E6%9E%B6%E6%9E%84-2.png)

然后验证指定架构处理程序已经启用，例如：

```Bash
cat /proc/sys/fs/binfmt_misc/qemu-aarch64
```

![docker多架构-3](media/docker%E5%A4%9A%E6%9E%B6%E6%9E%84-3.png)

### 步骤3：将默认Docker镜像builder切换成多架构builder

`docker buildx`通过builder实例对象来管理构建配置和节点，命令行将构建任务发送至builder实例，再由builder指派给符合条件的节点执行。我们可以基于同一个docker服务程序创建多个builder实例，提供给不同的项目使用以隔离各个项目的配置，也可以为一组远程docker节点创建一个builder实例组成构建阵列，并在不同阵列之间快速切换。

使用`docker buildx create`命令可以创建builer实例，这将以当前使用的docker服务为节点创建一个新的builder实例。

```Bash
docker buildx create --use --name mybuilder
```

> `docker buildx ls`命令查询多架构构建器。`docker buildx rm`命令删除某个多架构构建器。

验证新的构建器已经生效：

```Bash
docker buildx ls
```

![docker多架构-4](media/docker%E5%A4%9A%E6%9E%B6%E6%9E%84-4.png)

构建器创建完成后，会启动一个buildkit容器：

![docker多架构-5](media/docker%E5%A4%9A%E6%9E%B6%E6%9E%84-5.png)

现在Docker会使用新的构建器，支持构建多架构镜像。


### 步骤4：构建多架构镜像

假设在ubuntu镜像的基础上构建包含JDK的镜像，由于JDK支持x86以及arm架构的版本，因此根据不同架构的JDK来构建不同架构的镜像。

文件如下：

![docker多架构-6](media/docker%E5%A4%9A%E6%9E%B6%E6%9E%84-6.png)

Dockerfile如下：

```Bash
FROM ubuntu:18.04
ARG TARGETARCH

WORKDIR /usr/local/java/
RUN apt-get update && apt-get install -y locales && rm -rf /var/lib/apt/lists/* \
    && localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
ENV LANG en_US.utf8
ADD jdk-8u331-linux-${TARGETARCH}.tar.gz /usr/local/java/
ENV JAVA_HOME=/usr/local/java/jdk1.8.0_331
ENV CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV PATH=$JAVA_HOME/bin:$PATH
```

通过buildx来构建一个支持arm64、amd64架构的多架构镜像，并一次性推送到Docker Hub:

```Bash
docker buildx build -t wangqifox/jdk:8u331 --platform amd64,arm64 . --push
```

![docker多架构-7](media/docker%E5%A4%9A%E6%9E%B6%E6%9E%84-7.png)

执行构建命令时，除了指定镜像名称，另外两个重要的选项是指定目标平台和输出格式。







现在Docker Hub上就有了支持amd64和arm64两个架构的多架构Docker镜像。当我们运行`docker pull wangqifox/jdk:8u331`时，Docker会根据机器的架构来获取匹配的镜像。

![docker多架构-8](media/docker%E5%A4%9A%E6%9E%B6%E6%9E%84-8.png)

在命令的背后，buildx使用QEMU和binfmt_misc创建了两个Docker镜像（arm64和amd64架构每个创建一个）。当构建完成后，Docker会创建一个清单，其中包含这三个镜像以及他们对应的架构。换句话说，“多架构镜像”实际上是一个清单，列举了每个架构对应的镜像。

### 步骤5：测试多架构镜像

列出每个镜像的散列值：

```Bash
docker buildx imagetools inspect wangqifox/jdk:8u331
```

![docker多架构-9](media/docker%E5%A4%9A%E6%9E%B6%E6%9E%84-9.png)

通过这些散列治，我们可以逐一运行镜像：
![docker多架构-10](media/docker%E5%A4%9A%E6%9E%B6%E6%9E%84-10.png)

### 架构相关变量

`Dockerfile`支持如下架构相关的变量

- **`TARGETPLATFORM`**

构建镜像的目标平台，例如 `linux/amd64`, `linux/arm/v7`, `windows/amd64`。

- **`TARGETOS`**

`TARGETPLATFORM` 的 OS 类型，例如 `linux`, `windows`

- **`TARGETARCH`**

`TARGETPLATFORM` 的架构类型，例如 `amd64`, `arm`

- **`TARGETVARIANT`**

`TARGETPLATFORM` 的变种，该变量可能为空，例如 `v7`

- **`BUILDPLATFORM`**

构建镜像主机平台，例如 `linux/amd64`

- **`BUILDOS`**

`BUILDPLATFORM` 的 OS 类型，例如 `linux`

- **`BUILDARCH`**

`BUILDPLATFORM` 的架构类型，例如 `amd64`

- **`BUILDVARIANT`**

`BUILDPLATFORM` 的变种，该变量可能为空，例如 `v7`





## 将多架构Docker镜像推送到私服

buildx默认必须使用https来推送镜像到私服，无法使用http。需要在创建buildkit时执行一个配置文件：

```Bash
debug = true
# root is where all buildkit state is stored.
root = "/var/lib/buildkit"
# insecure-entitlements allows insecure entitlements, disabled by default.
insecure-entitlements = [ "network.host", "security.insecure" ]

# registry configures a new Docker register used for cache import or output.
[registry."docker.io"]
  mirrors = ["harbor-service:8888"]
  http = true
  insecure = true

# optionally mirror configuration can be done by defining it as a registry.
[registry."harbor-service:8888"]
  http = true

```

在create buildx时指定配置文件

```Bash
docker buildx create --config /root/buildx/buildkitd.toml --use --name mybuilder
```



buildkit无法使用宿主机中配置的hosts，需要修改buildkit的hosts：
![docker多架构-11](media/docker%E5%A4%9A%E6%9E%B6%E6%9E%84-11.png)
执行`docker exec -it 84e4ad0d7e50 sh`。在/etc/hosts中添加`124.221.194.47 harbor-service`。









> https://www.infoq.cn/article/v9qj0fjj6hsgyq0lphxg
> https://yeasy.gitbook.io/docker_practice/buildx/multi-arch-images
> https://www.waynerv.com/posts/building-multi-architecture-images-with-docker-buildx/
> https://yeasy.gitbook.io/docker_practice/image/manifest