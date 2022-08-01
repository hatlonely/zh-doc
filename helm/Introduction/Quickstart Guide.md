# 快速开始

本指南介绍如何快速开始使用 helm。

## 先决条件

成功且安全地使用 helm 需要如下先决条件

1. 一个 kubernetes 集群
2. 应用于安装的安全配置（如果有）
3. 安装和配置 helm

### 安装 kubernetes 或者有集群访问权限

- 你必须有一个 kubernetes 集群。对于最新的 helm 版本，我们推荐最新稳定的 kubernetes 版本，大部分情况下是第二个最新的次要版本
- 你还需要在本地安装 kubectl 工具

有关 helm 和 kubernetes 之间的版本兼容性，请参阅 [helm 版本支持策略](../官方指南/helm-版本支持策略.md)

## 安装 Helm

下载一个 helm client 的二进制版本。你可以使用类似 homebrew 的工具，或者在[官方发布页面](https://github.com/helm/helm/releases)查找

## 初始化一个 Helm Chart 仓库

安装 Helm 之后，你可以添加一个 chart 仓库。在 [Artifact Hub](https://artifacthub.io/packages/search?kind=0) 上搜寻可用的 Helm chart 仓库在

```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
```

添加成功之后，你可以列出你可以安装的 charts

```
$ helm search repo bitnami
NAME                             	CHART VERSION	APP VERSION  	DESCRIPTION
bitnami/bitnami-common           	0.0.9        	0.0.9        	DEPRECATED Chart with custom templates used in ...
bitnami/airflow                  	8.0.2        	2.0.0        	Apache Airflow is a platform to programmaticall...
bitnami/apache                   	8.2.3        	2.4.46       	Chart for Apache HTTP Server
bitnami/aspnet-core              	1.2.3        	3.1.9        	ASP.NET Core is an open-source framework create...
# ... and many more
```

## 安装一个样例 chart

你可以运行 `helm instal` 命令来安装一个 chart. helm 有好几种方式来查找和安装 chart，最简单的方式是使用 `bitnami` charts（注：应该是指用 repo add 一个远端的仓库地址后，指定仓库名字和 chart 名字直接安装）

```
$ helm repo update              # Make sure we get the latest list of charts
$ helm install bitnami/mysql --generate-name
NAME: mysql-1612624192
LAST DEPLOYED: Sat Feb  6 16:09:56 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES: ...
```

在上面这个例子中，`bitnami/mysql` chart 已经发布了，并且发布的名字为 `mysql-1612624192`

你可以非常简单地运行命令 `helm show chart bitnami/mysql` 来了解这个 MySQL chart 的功能。或者运行 `helm show all bitnami/mysql` 来了解这个 chart 的全部信息

无论什么时候安装一个 chart，都会有一个新的发布版本被创建出来。所以同一个 chart 可以在同一个集群中被安装多次（名字不同）。并且每个发布都可以独立的管理和更新

`helm install` 命令是非常强大的命令，具有许多功能。要了解更新信息，请参阅[使用 helm](使用-helm.md)

## 关于发布版本

使用 helm 可以很容易看出来发布了哪些东西

```
$ helm list
NAME            	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART      	APP VERSION
mysql-1612624192	default  	1       	2021-02-06 16:09:56.283059 +0100 CET	deployed	mysql-8.3.0	8.0.23
```

`helm list`（或者 `helm ls`）的功能是向你展示所有已经部署的版本

## 卸载一个版本

使用 `helm uninstall` 命令可以卸载一个版本

```
$ helm uninstall mysql-1612624192
release "mysql-1612624192" uninstalled
```

这个命令将从 kuberbetes 卸载 mysql-1612624192，相关的所有资源和版本历史记录都将被移除。

如果提供了 `--keep-history` 选项，版本历史记录将会被保存。你仍然可以请求和这个版本相关的信息：

```
$ helm status mysql-1612624192
Status: UNINSTALLED
...
```

因为 helm 即使在已经卸载版本之后仍然追踪它们，所以您可以审计集群的历史记录，甚至可以取消删除一个版本（使用 `helm rollback`）。

## 阅读帮助文本

要了解更多关于 helm 可用命令的信息，请使用 `helm help` 或者在一个命令后键入 `-h` 选项。

```
$ helm get -h
```

## 链接

- Quickstart Guide: <https://helm.sh/docs/intro/quickstart/>
