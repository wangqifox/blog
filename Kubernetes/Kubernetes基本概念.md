---
title: Kubernetes基本概念
date: 2018/07/08 10:48:25
---

Kubernetes中的大部分概念如Node、Pod、Replication Controller、Service等都可以看做一种"资源对象"，几乎所有的资源对象都可以通过Kubernetes提供的kubectl工具（或者API编程调用）执行增、删、改、查等操作并将其保存在etcd中持久化存储。从这个角度来看，Kubernetes其实是一个高度自动化的资源控制系统，它通过跟踪对比etcd库里保存的"资源期望状态"与当前环境中的"实际资源状态"的差异来实现自动控制和自动纠错的高级功能。
<!-- more -->
# Master

Kubernetes里的Master指的是集群控制节点，每个Kubernetes集群里需要有一个Master节点来负责整个集群的管理和控制，基本上Kubernetes所有的控制命令都是发给它，它来负责具体的执行过程，我们后面所有执行的命令基本上都是在Master节点上运行的。Master节点通常会占据一个独立的的X86服务器（或者一个虚拟机），一个主要的原因是它太重要了，它是整个集群的"首脑"，如果它宕机或者不可用，那么我们所有的控制命令都将失效。

Master节点上运行着以下一组关键进程。

- Kubernetes API Server（kube-apiserver），提供了HTTP Rest接口的关键服务进程，是Kubernetes里所有资源的增、删、改、查等操作的唯一入口，也是集群控制的入口进程。
- Kubernetes Controller Manager（kube-controller-manager），Kubernetes里所有资源对象的自动化控制中心，可以理解为资源对象的"大总管"。
- Kubernetes Scheduler（kube-scheduler），负责资源调度（Pod调度）的进程，相当于公交公司的"调度室"。

其实Master节点上往往还启动了一个etcd Server进程，因为Kubernetes里的所有资源对象的数据全部是保存在etcd中的。

# Node

除了Master，Kubernetes集群中的其他机器被称为Node节点，在较早的版本中也被称为Minion。与Master一样，Node节点可以是一台物理主机，也可以是一台虚拟机。Node节点才是Kubernetes集群中的工作负载节点，每个Node都会被Master分配一些工作负载（Docker容器），当某个Node宕机时，其上的工作负载会被Master自动转移到其他节点上去。

每个Node节点上都运行着以下一组关键进程。

- kubelet：负责Pod对应的容器的创建、启停等任务，同时与Master节点密切协作，实现集群管理的基本功能。
- kube-proxy：实现Kubernetes Service的通信与负载均衡机制的重要组件
- Docker Engine（docker）：Docker引擎，负责本机的容器创建和管理工作。

Node节点可以在运行期间动态增加到Kubernetes集群中，前提是这个节点上已经正确安装、配置和启动了上述关键进程，在默认情况下kubelet会向Master注册自己，这也是Kubernetes推荐的Node管理方式。一旦Node被纳入集群管理范围，kubelet进程就会定时向Master节点汇报自身的情报，例如操作系统、Docker版本、机器的CPU和内存情况，以及之前有哪些Pod在运行等，这样Master可以获知每个Node的资源使用情况，并实现高效均衡的资源调度策略。而某个Node超过指定时间不上报信息时，会被Master判定为"失联"，Node的状态被标记为不可用（Not Ready），随后Master会触发"工作负载大转移"的自动流程。

# Pod

Pod是Kubernetes的最重要也是最基本的概念。每个Pod都有一个特殊的被称为"根容器"的Pause容器。Pause容器对应的镜像属于Kubernetes平台的一部分，除了Pause容器，每个Pod还包含一个或多个紧密相关的用户业务容器。

为什么Kubernetes会设计出一个全新的Pod的概念并且Pod有这样特殊的组成结构？

原因之一：在一组容器作为一个单元的情况下，我们难以对"整体"简单地进行判断及有效地进行行动。比如，一个容器死亡了，此时算是整体死亡么？是N/M的死亡率么？引入业务无关并且不易死亡的Pause容器作为Pod的根容器，以它的状态代表整个容器组的状态，就简单、巧妙地解决了这个难题。

原因之二：Pod里的多个业务容器共享Pause容器的IP，共享Pause容器挂接的Volumne，这样既简化了密切关联的业务容器之间的通信问题，也很好地解决了它们之间的文件共享问题。

Kubernetes为每个Pod都分配了唯一的IP地址，称之为Pod IP，一个Pod里的多个容器共享Pod IP地址。Kubernetes要求底层网络支持集群内任意两个Pod之间的TCP/IP直接通信，这通常采用虚拟二层网络技术来实现，例如Flannel、Openvswitch等，因此我们需要牢记一点：在Kubernetes里，一个Pod里的容器与另外主机上的Pod容器能够直接通信。

Pod其实有两种类型：普通的Pod及静态Pod（static Pod），后者比较特殊，它并不存放在Kubernetes的etcd存储里，而是存放在某个具体的Node上的一个具体文件中，并且只在此Node上启动运行。而普通的Pod一旦被创建，就会被放入到etcd中存储，随后会被Kubernetes Master调度到某个具体的Node上并进行绑定（Binding），随后该Pod被对应的Node上的kubelet进程实例化成一组相关的Docker容器并启动起来。在默认情况下，当Pod里的某个容器停止时，Kubernetes会自动检测到这个问题并且重新启动这个Pod（重启Pod里的所有容器），如果Pod所在的Node宕机，则会将这个Node上的所有Pod重新调度到其他节点上。

# Label

Label是Kubernetes系统中另外一个核心概念。一个Label是一个key=value的键值对，其中key与value由用户自己指定。Label可以附加到各种资源对象上，例如Node、Pod、Service、RC等，一个资源对象可以定义任意数量的Label，同一个Label也可以被添加到任意数量的资源对象上去，Label通常在资源对象定义时确定，也可以在对象创建后动态添加或者删除。

我们可以通过给指定的资源对象捆绑一个或多个不同的Label来实现多维度的资源分组管理功能，以便于灵活、方便地进行资源分配、调度、配置、部署等管理工作。

Label相当于我们熟悉的"标签"，给某个资源对象定义一个Label，就相当于给它打了一个标签，随后可以通过Label Selector（标签选择器）查询和筛选拥有某些Label的资源对象，Kubernetes通过这种方式实现了类似SQL的简单又通用的对象查询机制。

Label Selector在Kubernetes中的重要使用场景有以下几处。

- kube-controller进程通过资源对象RC上定义的Label Selector来筛选要监控的Pod副本的数量，从而实现Pod副本的数量始终符合预期设定的全自动控制流程。
- kube-proxy进程通过Service的Label Selector来选择对应的Pod，自动建立起每个Service到对应Pod的请求转发路由表，从而实现Service的智能负载均衡机制
- 通过对某些Node定义特定的Label，并且在Pod定义文件中使用NodeSelector这种标签调度策略，kube-scheduler进程可以实现Pod"定向调度"的特性。

# Replication Controller（RC）

RC是Kubernetes系统中的核心概念之一，简单来说，它其实是定义了一个期望的场景，即声明某种Pod的副本数量在任意时刻都符合某个预期值，所以RC的定义包括如下几个部分。

- Pod期待的副本数（replicas）
- 用于筛选目标Pod的Label Selector
- 当Pod的副本数量小于预期数量的时候，用于创建新Pod的Pod模板（template）

当我们定义了一个RC并提交到Kubernetes集群中以后，Master节点上的Controller Manager组件就得到通知，定期巡检系统中当前存活的目标Pod，并确保目标Pod实例的数量刚好等于此RC的期望值，如果有过多的Pod副本在运行，系统就会停掉一些Pod，否则系统就会再自动创建一些Pod。可以说，通过RC，Kubernetes实现了用户应用集群的高可用性，并且大大减少了系统管理员在传统IT环境中需要完成的许多手动运维工作。

当我们的应用升级时，通常会通过build一个新的Docker镜像，并用新的镜像版本来替代旧的版本的方式达到目的。在系统升级的过程中，我们希望是平滑的方式，比如当前系统中10个对应的旧版本的Pod，最佳的方式是旧版本的Pod每次停止一个，同时创建一个新版本的Pod，在整个升级过程中，此消彼长，而运行中的Pod数量始终是10个，几分钟以后，当所有的Pod都已经是新版本的时候，升级过程完成。通过RC的机制，Kubernetes很容易就实现了这种高级实用的特性，被称为"滚动升级"。

Replica Set与Deployment这两个重要资源对象逐步替换了之前的RC的作用，是Kubernetes 1.3里Pod自动扩容（伸缩）这个高级功能实现的基础，也将继续在Kubernetes未来的版本中发挥重要作用。

# Deployment

Deployment是Kubernetes 1.2引入的新概念，引入的目的是为了更好地解决Pod的编排问题。为此，Deployment在内部使用了Replica Set来实现目的，无论从Deployment的作用于目的、它的yaml定义，还是从它具体命令行操作来看，我们都可以把它看做RC的一次升级，两者的相似度超过90%。

Deployment相对于RC的一个最大升级是我们可以随时知道当前Pod"部署"的进度。实际上由于一个Pod的创建、调度、绑定节点及在目标Node上启动对应的容器这一完整过程需要一定的时间，所以我们期待系统启动N个Pod副本的目标状态，实际上是一个连续变化的"部署过程"导致的最终状态。

Deployment的典型使用场景有以下几个：

- 创建一个Deployment对象来生成对应的Replica Set并完成Pod副本的创建过程
- 检查Deployment的状态来看部署动作是否完成（Pod副本的数量是否达到预期的值）
- 更新Deployment以创建新的Pod（比如镜像升级）
- 如果当前Deployment不稳定，则回滚到一个早先的Deployment版本
- 挂起或者恢复一个Deployment

# HPA

Horizontal Pod Autoscaling，Pod横向自动扩容，简称HPA。与之前的RC、Deployment一样，也属于一种Kubernetes资源对象。通过追踪分析RC控制的所有目标Pod的负载变化情况，来确定是否需要针对性地调整目标Pod的副本数，这是HPA的实现原理。当前HPA可以有以下两种方式作为Pod负载的度量指标：

- CPU Utilization Percentage
- 应用程序自定义的度量指标，比如服务在每秒内的相应的请求数(TPS或QPS)

# StatefulSet

在Kubernetes系统中，Pod的管理对象RC、Deployment、DaomonSet和Job都是面向无状态的服务。但在现实中有很多服务是有状态的，特别是一些复杂的中间件集群，例如Mysql集群、MongoDB集群、Akka集群、Zookeeper集群等，这些应用集群有以下一些共同点。

- 每个节点都有固定的身份ID，通过这个ID，集群中的成员可以相互发现并通信。
- 集群的规模是比较固定的，集群规模不能随意变动
- 集群里的每个节点都是有状态的，通常会持久化数据到永久存储中
- 如果磁盘损坏，则集群里的某个节点无法正常运行，集群功能受损

如果用RC/Deployment控制Pod副本数的方式来实现上述有状态的集群，则我们会发现第1点是无法满足的，因为Pod的名字是随机产生的，Pod的IP地址也是运行期才确定且可能有变动的，我们事先无法为每个Pod确定唯一不变的ID。另外，为了能够在其他节点上恢复某个失败的节点，这种集群中的Pod需要挂接某种共享存储，为了解决这个问题，Kubernetes引入了StatefulSet，它从本质上可以看做是Deployment/RC的一个特殊变种，它有如下一些特性。

- StatefulSet里的每个Pod都有稳定、唯一的网络标识，可以用来发现集群内的其他成员。假设StatefulSet的名字叫kafka，那么第1个Pod叫kafka-0，第2个Pod叫kafka-1，以此类推。
- StatefulSet控制的Pod副本的启停顺序是受控的，操作第n个Pod时，前n-1个Pod已经是运行且准备好的状态。
- StatefulSet里的Pod采用稳定的持久化存储卷，通过PV/PVC来实现，删除Pod时默认不会删除与StatefulSet相关的存储卷（为了保证数据的安全）

StatefulSet除了要与PV卷捆绑使用以存储Pod的状态数据，还要与Headless Service配合使用，即在每个StatefulSet的定义中要声明它属于哪个Headless Service。Headless Service与普通Service的关键区别在于，它没有Cluster IP，如果解析Headless Service的DNS域名，则返回的是该Service对应的全部Pod的Endpoint列表。StatefulSet在Headless Service的基础上又为StatefulSet控制的每个Pod实例创建了一个DNS域名，这个域名的格式为：

```java
${podname}.${headless service name}
```

比如一个3节点的Kafka的StatefulSet集群，对应的Headless Service的名字为kafka，StatefulSet的名字为kafka，则StatefulSet里面的3个Pod的DNS名称分别为kafka-0.kafka、kafka-1.kafka、kafka-2.kafka，这些DNS名称可以直接在集群的配置文件中固定下来。

# Service

Service也是Kubernetes里的最核心的资源对象之一，Kubernetes里的每个Service其实就是我们经常提起的微服务架构中的一个"微服务"，之前我们所说的Pod、RC等资源对象其实都是为这节所说的"服务"——Kubernetes Service做"嫁衣"的。

Kubernetes的Service定义了一个服务的访问入口地址，前端的应用（Pod）通过这个入口地址访问其背后的一组由Pod副本组成的集群实例，Service与其后端的Pod副本集群之间则是通过Label Selector来实现"无缝对接"的。而RC的作用实际上是保证Service的服务能力和服务质量始终处于预期的标准。

通过分析、识别并建模系统中的所有服务为微服务——Kubernetes Service，最终我们的系统由多个提供不同业务能力而又彼此独立的微服务单元所组成，服务之间通过TCP/IP进行通信，从而形成了我们强大而又灵活的弹性网格，拥有了强大的分布式能力、弹性扩展能力、容错能力，与此同时，我们的程序架构也变得简单和直观许多。

既然每个Pod都会被分配一个单独的IP地址，而且每个Pod都提供了一个独立的Endpoint（Pod IP + ContainerPort）以被客户端访问，现在多个Pod副本组成了一个集群来提供服务，那么客户端如何来访问它们呢？一般的做法是部署一个负载均衡器（软件或硬件），为这组Pod开启一个对外的服务端口如8080端口，并且将这些Pod的Endpoint列表加入8080端口的转发列表中，客户端就可以通过负载均衡器的对外IP地址+服务端口来访问此服务，而客户端的请求最后会被转发到哪个Pod，则由负载均衡器的算法所决定。

Kubernetes也遵循了上述常规做法，运行在每个Node上的kube-proxy进程其实就是一个智能的软件负载均衡器，它负责把对Service的请求转发到后端的某个Pod实例上，并在内部实现服务的负载均衡与会话保持机制。但Kubernetes发明了一种很巧妙又影响深远的设计：Service不是共用一个负载均衡器的IP地址，而是每个Service分配了一个全局唯一的虚拟IP地址，这个虚拟IP被称为Cluster IP。这样一来，每个服务就变成了具备唯一IP地址的"通信节点"，服务调用就变成了最基础的TCP网络通信问题。

我们知道，Pod的Endpoint地址会随着Pod的销毁和重新创建而发生改变，因为新Pod的IP地址与之前旧Pod的不同。而Service一旦创建，Kubernetes就会自动为它分配一个可用的Cluster IP，而且在Service的整个生命周期内，它的Cluster IP不会发生改变。于是，服务发现这个棘手的问题在Kubernetes的框架里也得以轻松解决：只要用Service的Name和Service的Cluster IP地址做一个DNS域名映射即可完美解决问题。

## Kubernetes的服务发现机制

任何分布式系统都会涉及"服务发现"这个基础问题，大部分分布式系统通过提供特定的API接口来实现服务发现的功能，但这样做会导致平台的侵入性比较强，也增加了开发测试的困难。Kubernetes则采用了直观朴素的思想去解决这个棘手的问题。

首先，每个Kubernetes中的Service都有一个唯一的Cluster IP以及唯一的名字，而名字是由开发者自己定义的，部署的时候也没必要改变，所以完全可以固定在配置中。接下来的问题就是如何通过Service的名字找到对应的Cluster IP。

最早的时候Kubernetes采用了Linux环境变量的方式解决这个问题，即每个Service生成一些对应的Linux环境变量，并在每个Pod容器在启动时，自动注入这些环境变量。

每个Serivce的IP地址及端口都是有标准的命名规范的，遵循这个命名规范，就可以通过代码访问系统环境变量的方式得到所需的信息，实现服务调用。

考虑到环境变量的方式获取Service的IP与端口的方式仍然不太方便，不够直观，后来Kubernetes通过Add-On增值包的方式引入了DNS系统，把服务名作为DNS域名，这样一来，程序就可以直接使用服务名来建立通信连接了。

## 外部系统访问Service的问题

为了更加深入地理解和掌握Kubernetes，我们需要弄明白Kubernetes里的"三种IP"这个关键问题，这三种IP分别如下：

- Node IP: Node节点的IP地址
- Pod IP: Pod的IP地址
- Cluster IP: Service的IP地址

首先，Node IP是Kubernetes集群中每个节点的物理网卡的IP地址，这是一个真实存在的物理网络，所有属于这个网络的服务器之间都通过这个网络直接通信，不管它们中是否有部分节点不属于这个Kubernetes集群。这也表明了Kubernetes集群之外的节点访问Kurbernetes集群之内的某个节点或者TCP/IP服务的时候，必须通过Node IP进行通信。

其次，Pod IP是每个Pod的IP地址，它是Docker Engine根据docker0网桥的IP地址段进行分配的，通常是一个虚拟的二层网络，前面我们说过，Kubernetes要求位于不同Node上的Pod上的Pod能够彼此直接通信，所以Kubernetes里一个Pod里的容器访问另外一个Pod里的容器，就是通过Pod IP所在的虚拟二层网路进行通信的，而真实的TCP/IP流量则是通过Node IP所在的物理网卡流出的。

最后，我们说说Service的Cluster IP，它也是一个虚拟的IP，但更像是一个"伪造"的IP网络，原因有以下几点。

- Cluster IP仅仅作用于Kubernetes Service这个对象，并由Kubernetes管理和分配IP地址
- Cluster IP无法被Ping，因为没有一个"实体网络对象"来响应
- Cluster IP只能结合Service Port组成一个具体的通信端口，单独的Cluster IP不具备TCP/IP通信的基础，并且它们属于Kubernetes集群这样一个封闭的空间，集群之外的节点如果要访问这个通信端口，则需要做一些额外的工作
- 在Kubernetes集群之内，Node IP网、Pod IP网与Cluster IP网之间的通信，采用的是Kubernetes自己设计的一种编程方式的特殊路由规则，与我们所熟知的IP路由有很大的不同

# Volume（存储卷）

Volume是Pod中能够被多个容器访问的共享目录。Kubernetes的Volume概念、用途和目的与Docker的Volume比较类似，但两者不能等价。首先，Kubernetes中的Volume定义在Pod上，然后被一个Pod里的多个容器挂载到具体的文件目录下；其次，Kubernetes中的Volume与Pod的生命周期相同，但与容器的生命周期不相关，当容器终止或者重启时，Volume中的数据也不会丢失。最后，Kubernetes支持多种类型的Volume，例如GlusterFS、Ceph等先进的分布式文件系统。

Volume的使用也比较简单，在大多数情况下，我们先在Pod上声明一个Volume，然后在容器里引用该Volume并Mount到容器里的某个目录上。

除了可以让一个Pod里的多个容器共享文件、让容器的数据写到宿主机的磁盘上或者写文件到网络存储中，Kubernetes的Volume还扩展出一种非常有实用价值的功能，即容器配置文件集中化定义与管理，这是通过ConfigMap这个新的资源对象来实现的。

Kubernetes提供了非常丰富的Volume类型。

## emptyDir

一个emptyDir Volume是在Pod分配到Node时创建的。从它的名字就可以看出，它的初始内容为空，并且无须指定宿主机上对应的目录文件，因为这是Kubernetes自动分配的一个目录，当Pod从Node上移除时，emptyDir中的数据也会被永久删除。emptyDir的一些用途如下：

- 临时空间，例如用于某些应用程序运行时所需的临时目录，且无须永久保留
- 长时间任务的中间过程CheckPoint的临时保存目录
- 一个容器需要从另一个容器中获取数据的目录（多容器共享目录）

## hostPath

hostPath为在Pod上挂载宿主机上的文件或目录，它通常可以用于以下几方面：

- 容器应用程序生成的日志文件需要永久保存时，可以使用宿主机的高速文件系统进行存储
- 需要访问宿主机上Docker引擎内部数据结构的容器应用时，可以通过定义hostPath为宿主机/var/lib/docker目录，使容器内部应用可以直接访问Docker的文件系统

在使用这种类型的Volume时，需要注意以下几点：

- 在不同的Node上具有相同配置的Pod可能会因为宿主机上的目录和文件不同而导致对Volume上目录和文件的访问结果不一致。
- 如果使用了资源配额管理，则Kubernetes无法将hostPath在宿主机上使用的资源纳入管理

## gcePersistentDisk

使用这种类型的Volume表示使用谷歌公有云提供的永久磁盘（Persistent Disk，PD）存放Volume的数据，它与EmptyDir不同，PD上的内容会被永久保存，当Pod被删除时，PD只是被卸载(Unmount)，但不会被删除。

## awsElasticBlockStore

与GCE类似，该类型的Volume使用亚马逊公有云提供的EBS Volume存储数据。

## NFS

使用NFS网络文件系统提供的共享目录存储数据时，我们需要在系统中部署一个NFS Server。

# Persistent Volume

之前我们提到的Volume是定义在Pod上的，属于"计算资源"的一部分，而实际上，"网络存储"是相对独立于"计算资源"而存在的一种实体资源。比如在使用虚拟机的情况下，我们通常会先定义一个网络存储，然后从中划出一个"网盘"并挂接到虚机上。Persistent Volume（简称PV）和与之相关联的Persistent Volume Claim也起到了类似的作用。

# Namespace

Namespace是Kubernetes系统中的另一个非常重要的概念，Namespace在很多情况下用于实现多租户的资源隔离。Namespace通过将集群内部的资源对象"分配"到不同的Namespace中，形成逻辑上分组的不同项目、小组或用户组，便于不同的分组再共享使用整个集群的资源的同时还能被分别管理。

# Annotation

Annotation与Label类似，也使用key/value键值对的形式进行定义。不同的是Label具有严格的命名规则，它定义的是Kubernetes对象的元数据，并且用于Label Selector。而Annotation则是用户任意定义的"附加"信息，以便于外部工具进行查找，很多时候，Kubernetes的模块自身会通过Annotation的方式标记资源对象的一些特殊信息。

通常来说，用Annotation来记录的信息如下：

- build信息、release信息、Docker镜像信息等，例如时间戳、release id号、PR号、镜像hash值、docker registry地址等。
- 日志库、监控库、分析库等资源库的地址信息
- 程序调试工具信息，例如工具名称、版本号等
- 团队的联系信息，例如电话号码、负责人名称、网址等。




> Kubernetes实践指南

