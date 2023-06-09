---
title: Helm
date: 2022/09/26 10:48:25
---

Helm 是 Kubernetes 的包管理器，在 Kubernetes 下能够非常方便的完成应用的安装、卸载、升级等，是查看、分享和使用软件构建 Kubernetes 的最优方式，被广泛的使用。

<!-- more -->

# 背景

在 Kubernetes 环境中部署一个应用，需要 Kubernetes 原生资源 YAML 文件，如 Deployment、Service 或 Pod 等。对于一个应用而言，这些 Kubernetes 资源 YAML 文件都是十分分散的，不方便进行管理，通常直接通过 kubectl 命令来管理一个应用，你会发现这十分繁琐。

面对一个复杂的应用，会有很多类似的资源 YAML 文件，如果有更新或回滚应用的需求，可能要修改和维护所涉及的大量资源 YAML 文件，且由于缺少对发布过的应用版本管理和控制，使 Kubernetes 上的应用维护和更新等面临诸多的挑战，而 Helm 的出现可以帮助我们解决这些问题：

- 如何统一管理、配置和更新这些分散的 Kubernetes 的资源 YAML 文件
- 如何分发和复用一套应用模板
- 如何将应用的一系列资源 YAML 文件当做一个软件包管理

# 安装

通过脚本安装

```Bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

```

通过apt安装

```Bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

# 基本命令

## 添加chart存储库

```Bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```



## 查看helm安装的存储库

```Bash
helm repo list
```



## 搜索chart存储库

```Bash
helm search repo bitnami
```

### --versions

默认只展示最新的chart，可以使用`--versions`参数查看所有版本：

```Bash
helm search repo bitnami --versions
```



## 安装程序包

```Bash
helm install mysite bitnami/drupal
```

安装时需要指定installation名和chart名。该代码将创建bitnami/drupal chart的一个实例，并将该实例命名为mysite。

### --namespace

通过—namespace来指定chart安装的命名空间：

```Bash
helm install mysite bitnami/drupal --namespace first
```



### --values

通过`--values`标志，指向具有覆盖值的YAML文件：

```Bash
helm install mysite bitnami/drupal --values values.yaml
```



### --set

`--set`标志直接接收一个或多个值。他们不需要存储在YAML文件中：

```Bash
helm install mysite bitnami/drupal --set drupalUsername=admin
```



## 列出安装

默认命名空间的安装：

```Bash
helm list
```



列出所有命名空间的安装：

```Bash
helm list --all-namespaces
```



## 刷新存储库中的软件包

```Bash
helm repo update
```



## 升级安装

### --version

升级到某个特定的版本

```Bash
helm upgrade mysite bitnami/drupal --version 6.2.22
```



使用配置文件升级

```Bash
helm upgrade mysite bitnami/drupal --values values.yaml
```



## 卸载安装

删除某个helm安装

```Bash
helm uninstall mysite
```

提供—namespace标志来指定要从特定命名空间中删除安装：

```Bash
helm uninstall mysite --namespace first
```



## --dry-run试运行

helm install和helm upgrade等命令提供了一个名为`--dry-run`的标志。helm在命令行中输出渲染之后的模板，而不是发送到Kubernetes中。

```Bash
helm install mysite bitnami/drupal --dry-run
```



## helm template命令

与`--dry-run`不同，template命令不联系远程的Kubernetes服务器，需要联系Kubernetes服务器的模板函数和指令返回默认数据。

```Bash
helm template mysite bitnami/drupal
```



## --generate-name和--name-template标志

使用--generate-name标志，不再需要提供名称作为helm install的第一个参数，helm根据chart名称和时间戳的组合生成名称。

```Bash
helm install bitnami/wordpress --generate-name
```



使用`--name-template`将使用helm模板引擎生成一个名称。

```Bash
helm install bitnami/wordpress --generate-name --name-template "foo-{{ randAlpha 9 | lower }}"
```



## --create-namespace标志

`--create-namespace`标志向helm表明在安装前创建一个namespace

```Bash
helm install drupal bitnami/drupal --namespace mynamespace --create-namespace
```



## 安装或升级

`helm upgrade --install`命令将安装一个尚不存在的版本，或者升级一个以该名称命名的版本。

```Bash
helm update --install wordpress bitnami/wordpress
```



## --wait和--atomic标志

使用`--wait`进行安装时，helm将一直跟踪下发到kubernetes中的对象，直到它们创建的pod被标记为由Kubernetes运行为止。使用`--wait`标志时可以使用`--timeout`。



`--atomic`标志在安装失败时，将自动回滚到最后一个成功的版本，而不是将该版本标记为失败并退出。



## --force和--cleanup-on-fail标志

`--force`标志将删除并重新创建Deployment，而不是修改Deployment。这迫使Kubernetes删除旧Pod并创建新的Pod。



`--cleanup-on-fail`标志，如果升级失败，将请求删除升级期间新创建的每个对象。



## 创建chart

```Bash
helm create anvil
```

创建一个名为anvil的新chart作为当前目录的子目录。

可以通过以下命令来安装这个新创建的chart：

```Bash
helm install myapp anvil
```

最后一个anvil参数是chart所在的目录。

## 打包chart

```Bash
helm package anvil
```

anvil是anvil chart源所在位置的路径。



## 校验chart代码

```Bash
helm lint anvil
```



## 更新chart的依赖项

在指定范围内解析依赖项的最新版本并检索它：

```Bash
helm dependency update .
```



