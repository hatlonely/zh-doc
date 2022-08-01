# 使用 helm

本指南解释了使用 helm 管理 kubernetes 集群上的包的基本用法。我们假设你已经安装了 helm 客户端。

如果你只是想要跑一些快速命令，你也许可以阅读[快速开始](快速开始.md)。这个章节讲述了一些特别的 helm 命令，并且解释了如何使用 helm

## 三个重要概念

**包（Chart）** 是 helm 的包。它包含了所有在 kubernetes 上运行应用、工具或者服务所必要的资源定义。chart 在 kubernetes 上，可以类比于 homebrew formula（mac 安装包），apt dpkg（unbuntu 安装包），yum rpm 文件（centos 安装包）

**仓库（Repository）** 是一个 helm 包可以收藏和共享的地方。它就像 perl 的 [CPAN achive](https://www.cpan.org/) 或者 [Fedora Package Database](https://src.fedoraproject.org/)，只是用于 kubernetes 的包。

**版本（Release）** 是 chart 运行在 kubernetes 集群上面的一个实例。一个 chart 通常会在一个集群被安装很多次。并且每次安装，都会产生一个新的 Release。考虑一个 MySQL 的 chart. 如果你想在你的集群上运行两个数据，你可以安装这个 chart 两次。每一次都会有产生自己的 release，并且每次 release 都有自己的名字。

有了这些概念，我们可以这样解释 helm：

helm 安装 chart 到 kubernetes，每次安装创建新的 release。你可以在 repositiry 里面搜寻新的 chart。

## `helm search`：搜寻 charts

helm 附带了一个强大的搜索命令，可以用来搜索两种不同类型的资源：

- `helm search hub`：从 [Artifact Hub](https://artifacthub.io/) 上搜索，从众多不同的 repository 中列出 helm chart。
- `helm search repo`：从已添加到你本地客户端中的 repository（使用 `helm repo add`）。这个搜索在本地完成，不需要公网。

你可以使用 `helm search hub` 找到公开可用的 charts：

```
$ helm search hub wordpress
URL                                                 CHART VERSION APP VERSION DESCRIPTION
https://hub.helm.sh/charts/bitnami/wordpress        7.6.7         5.2.4       Web publishing platform for building blogs and ...
https://hub.helm.sh/charts/presslabs/wordpress-...  v0.6.3        v0.6.3      Presslabs WordPress Operator Helm Chart
https://hub.helm.sh/charts/presslabs/wordpress-...  v0.7.1        v0.7.1      A Helm chart for deploying a WordPress site on ...
```

上例完成了再 Artifact Hub 上面搜索所有的 `wordpress` chart。

不过滤的话，`helm search hub` 会张氏所有可用的 chart。

使用 `helm search repo`，你可以找到你已经添加的 charts 的名字：

```
$ helm repo add brigade https://brigadecore.github.io/charts
"brigade" has been added to your repositories
$ helm search repo brigade
NAME                          CHART VERSION APP VERSION DESCRIPTION
brigade/brigade               1.3.2         v1.2.1      Brigade provides event-driven scripting of Kube...
brigade/brigade-github-app    0.4.1         v0.2.1      The Brigade GitHub App, an advanced gateway for...
brigade/brigade-github-oauth  0.2.0         v0.20.0     The legacy OAuth GitHub Gateway for Brigade
brigade/brigade-k8s-gateway   0.1.0                     A Helm chart for Kubernetes
brigade/brigade-project       1.0.0         v1.0.0      Create a Brigade project
brigade/kashti                0.4.0         v0.4.0      A Helm chart for Kubernetes
```

helm 搜索使用模糊匹配算法，所以你可以输入单词或者短语的部分搜索

```
$ helm search repo kash
NAME            CHART VERSION APP VERSION DESCRIPTION
brigade/kashti  0.4.0         v0.4.0      A Helm chart for Kubernetes
```

搜索是一个寻找可用包的好方法。一旦你找到了你想要安装的包，你可以使用 `helm install` 来安装它。

## `helm install`：安装一个包

为了安装一个新的包，使用 `helm install` 命令。最简单的情况下，它只有两个参数：你取的 release 名字，以及你想要安装的 chart 的名字

```
$ helm install happy-panda bitnami/wordpress
NAME: happy-panda
LAST DEPLOYED: Tue Jan 26 10:27:17 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
** Please be patient while the chart is being deployed **

Your WordPress site can be accessed through the following DNS name from within your cluster:

    happy-panda-wordpress.default.svc.cluster.local (port 80)

To access your WordPress site from outside the cluster follow the steps below:

1. Get the WordPress URL by running these commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w happy-panda-wordpress'

   export SERVICE_IP=$(kubectl get svc --namespace default happy-panda-wordpress --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
   echo "WordPress URL: http://$SERVICE_IP/"
   echo "WordPress Admin URL: http://$SERVICE_IP/admin"

2. Open a browser and access WordPress using the obtained URL.

3. Login with the following credentials below to see your blog:

  echo Username: user
  echo Password: $(kubectl get secret --namespace default happy-panda-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)
```

现在 wordpress chart 已经安装。请注意，安装 chart 会创建一个新的 release 对象。这个 release 被命名为 happy-panda（如果你想要 helm 为你自动生成一个名字，去掉 release 名字并且使用 --generate-name）。

在安装期间，helm 客户端会打印一些有用信息，包括创建了哪些资源、release 的状态是什么以及是否有额外的配置步骤需要执行。

helm 安装资源的顺序如下：

- Namespace
- NetworkPolicy
- ResourceQuota
- LimitRange
- PodSecurityPolicy
- PodDisruptionBudget
- ServiceAccount
- Secret
- SecretList
- ConfigMap
- StorageClass
- PersistentVolume
- PersistentVolumeClaim
- CustomResourceDefinition
- ClusterRole
- ClusterRoleList
- ClusterRoleBinding
- ClusterRoleBindingList
- Role
- RoleList
- RoleBinding
- RoleBindingList
- Service
- DaemonSet
- Pod
- ReplicationController
- ReplicaSet
- Deployment
- HorizontalPodAutoscaler
- StatefuleSet
- Job
- CronJob
- Ingress
- APIService

helm 不会等待所有资源运行后才退出。很多 chart 需要超过 600M 的 docker 镜像，可能会需要花费很长时间才能安装到集群

为了追踪 release 的装态，或者为了重读配置信息，你可以用使用 `helm status` 命令

```
$ helm status happy-panda
NAME: happy-panda
LAST DEPLOYED: Tue Jan 26 10:27:17 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
** Please be patient while the chart is being deployed **

Your WordPress site can be accessed through the following DNS name from within your cluster:

    happy-panda-wordpress.default.svc.cluster.local (port 80)

To access your WordPress site from outside the cluster follow the steps below:

1. Get the WordPress URL by running these commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w happy-panda-wordpress'

   export SERVICE_IP=$(kubectl get svc --namespace default happy-panda-wordpress --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
   echo "WordPress URL: http://$SERVICE_IP/"
   echo "WordPress Admin URL: http://$SERVICE_IP/admin"

2. Open a browser and access WordPress using the obtained URL.

3. Login with the following credentials below to see your blog:

  echo Username: user
  echo Password: $(kubectl get secret --namespace default happy-panda-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)
```

上面展示了 release 当前的状态

### 安装前自定义 chart

我们之前的安装方式只会使用 chart 默认的配置。大多数情概况下，你会想要自定义 chart 的首选配置。

为了查看 chart 有哪些选项可以配置，可以使用 `helm show values`：

```
$ helm show values bitnami/wordpress
## Global Docker image parameters
## Please, note that this will override the image parameters, including dependencies, configured to use the global value
## Current available global Docker image parameters: imageRegistry and imagePullSecrets
##
# global:
#   imageRegistry: myRegistryName
#   imagePullSecrets:
#     - myRegistryKeySecretName
#   storageClass: myStorageClass

## Bitnami WordPress image version
## ref: https://hub.docker.com/r/bitnami/wordpress/tags/
##
image:
  registry: docker.io
  repository: bitnami/wordpress
  tag: 5.6.0-debian-10-r35
  [..]
```

你可以在一个 yaml 格式的文件中去覆盖这些设置，然后将这个文件传给安装命令（译者注：使用 `-f` 参数）。

```
$ echo '{mariadb.auth.database: user0db, mariadb.auth.username: user0}' > values.yaml
$ helm install -f values.yaml bitnami/wordpress --generate-name
```

上例将创建一个名为 user0 的默认的 MariaDB 用户，并且授予这个用户访问新创建的数据库 `user0db`，但是其余剩余所有的配置都是 chart 默认的。

在安装过程中有两种方法传递配置信息：

- `--values`（或者是 `-f`）：指定一个覆盖项的 yaml 文件。这个参数可以被指定多次，并且最右边的文件将会优先
- `--set`：在命令中指定覆盖信息

如果同时使用，`--set` 中的值将被合并到 `--values` 中，并切优先级更高。`--set` 指定的覆盖将会持久化到 `ConfigMap` 中。被 `--set` 设置的值可以通过 `helm get values <release-name>` 命令查看，可以通过运行 `helm upgrade` 指定 `--reset-values` 选项来清理。

### --set 选项的格式和限制

`--set` 选项接收0个或者更多的键值对。最简单的说，它的使用就像这样：`--set name=value`。等效的 yaml 是这样：

```yaml
name: value
```

多个值用 `,` 分开。所以 `--set a=b,c=d` 变成了：

```yaml
a: b
c: d
```

分复杂的表达式也是支持的。比如，`--set outer.inner=value` 会被翻译成这样：

```yaml
outer:
  inner: value
```

列表可以用 `{}` 将值括起来。比如，`--set name={a, b, c}` 会被翻译成：

```yaml
name:
  - a
  - b
  - c
```

特定的键可以被设置成 `null` 或者设置成空的数组 `[]`。比如，`--set name=[],a=null` 会将：

```yaml
name:
  - a
  - b
  - c
a: b
```

翻译成

```yaml
name: []
a: null
```

从 helm 2.5.0 以后，可以使用数组索引语法访问列表项。比如，`--set servers[0].port=80` 变成了：

```yaml
servers:
  - port: 80
```

多个值也可以通过这种方式设置。`--set servers[0].port=80,servers[0].host=example` 变成了：

```yaml
servers:
  - port: 80
  - host: example
```

有时候你需要在 `--set` 里面使用特殊的字符。你可以使用反斜杠来转义这些字符。`--set name=value\,value2` 会变成：

```yaml
name: "value1,value2"
```

同样的，你可以转义点（`.`），当 chart 使用 `toYaml` 方法来解析 annotation（注解），label（标签），node selector（节点选择器）可能会碰到。这个语法 `--set nodeSelector."kubernetes\.io/role"=master` 会变成：

```yaml
nodeSelector:
  kubernetes.io/role: master
```

深度嵌套的数据结构很难用 `--set` 来表达。鼓励 chart 的设计者在设计 `values.yaml` 文件格式的时候考虑 `--set` 用法（获取更多关于[values 文件](https://helm.sh/docs/chart_template_guide/values_files/)的信息）。

### 更多安装方法

helm install 命令可以安装多种资源：

- 一个 chart repository（就像我们上面看到的那样）
- 一个本地的 chart 压缩包（`helm install foo foo-0.0.1.tgz`）
- 一个未压缩 chart 目录（`helm install foo path/to/foo`）
- 一个全路径 URL（`helm install foo https://example.com/charts/foo-1.2.3.tgz`）

## `helm upgrade` 和 `helm rollback`：更新 release 和故障恢复

当有一个新版本需要发布，或者你想要改变一些配置项的时候，可以用使用 `helm upgrade` 命令。

一次更新会使用已有的发布并根据你提供的信息更新。因为 kubernetes chart 可能会很大很复杂，helm 尽量执行最少的侵入性更新。它将只更新和上次发布有变化的部分

```
$ helm upgrade -f panda.yaml happy-panda bitnami/wordpress
```

上例中，happly-panda 这个 release 会使用同样的 chart 和新的 yaml 文件更新：

```yaml
mariadb.auth.username: user1
```

我们可以使用 `helm get values` 来查看新的配置是否生效。

```
$ helm get values happy-panda
mariadb:
  auth:
    username: user1
```

`helm get` 是一个非常有用的工具来查看集群中的 release。正如我们上面看到的那样，`panda.yaml` 中新的值被部署到了集群中。

现在，如果有时候发布的时候不符合预期，使用 `helm rollback [RELEASE] [REVISION]` 很容易回滚到上一个版本。

```
$ helm rollback happy-panda 1
```

上例回滚了我们的 happy-panda 到它最初的那个发布版本。发布版本是一个递增的修订号。每一次安装，更新或者回滚发生的时候，修订号都会增加 1。第一个修订号为 1。我们可以使用 `helm history [RELEASE]` 来查看特定 release 的修订号

### 安装/升级/回滚的有用选项

这里有几个其他在你安装/升级/回滚时自定义 helm 的行为时非常有帮助的选项。注意这些不是全部的客户端选项。要查看所有的选项描述，只需要运行 `helm <command> --help` 即可。

- `--timeout`：一个 Go 时间段值用于等待 kubernetes 命令完成。默认是 `5m0s`（5分钟）。
- `--wait`：在标记 release 为成功之前，等待所有的 pod 都处于 `ready`（就绪）状态，PVC 都已经绑定，部署（deployment）都有最小就绪状态 pod 数量（期望数量 - 最大不可用数量），并且服务有 ip 地址（如果是负载均衡器则是 ingress）。这将等待 `--timeout` 设置的超时时间。如果达到超时时间，这个 release 会被标记为 `FAILED`。注意：有些场景下，depolyment 的滚动更新策略（rolling update strategy）设置了 副本数量（replica）为 1 并且最大不可用数量（maxUnavailable） 未设置成 0，`--wait` 将直接返回就绪状态（ready），因为它已经满足最小 pod 就绪状态的条件。
- `--no-hooks`：跳过运行钩子
- `--repcreate-pods`（只在升级和回滚中可用）：这个选项将导致所有的 pods 都被重建（部署（depoloyment）中的 pod 除外）。（在 helm 3 中废弃）

### `helm uninstall`：卸载 release

当需要从集群中卸载一个 release 时，使用 `helm uninstall` 命令：

```
$ helm uninstall happy-panda
```

这将从集群中移除 release。你可以使用 `helm list` 命令查看你当前已经部署的 release

```
$ helm list
NAME            VERSION UPDATED                         STATUS          CHART
inky-cat        1       Wed Sep 28 12:59:46 2016        DEPLOYED        alpine-0.1.0
```

从上面的输出中，我们可以看出 `happy-panda` release 已经被卸载了

早期的 helm 版本中，当一个 release 被删除的时候，会保留一条它的删除记录。在 helm 3 中，删除同时也移除了 release 记录。如果你想要保留 release 记录，可以使用 `helm uninstall --key-history`。使用 `helm list --uninstalled` 将只展示使用 `--key-history` 选项卸载的 release。

`helm list --all` 选项会展示所有 helm 保存的 release 记录，包括失败和已经删除的记录（如果 `--keep-history` 选项被指定）：

```
$  helm list --all
NAME            VERSION UPDATED                         STATUS          CHART
happy-panda     2       Wed Sep 28 12:47:54 2016        UNINSTALLED     wordpress-10.4.5.6.0
inky-cat        1       Wed Sep 28 12:59:46 2016        DEPLOYED        alpine-0.1.0
kindred-angelf  2       Tue Sep 27 16:16:10 2016        UNINSTALLED     alpine-0.1.0
```

注意因为 release 默认情况下已经被删除了，所以不可能回滚一个已经卸载的资源

## `helm repo`：使用 repository

helm 3 不再附带一个默认的 chart repostiry。`helm repo` 命令族提供命令来添加，列出和移除 repository。

你可以使用 `helm repo list` 看到哪些 repoistory 被配置：

```
$ helm repo list
NAME            URL
stable          https://charts.helm.sh/stable
mumoshu         https://mumoshu.github.io/charts
```

使用 `helm repo add` 添加一些新的 repository

```
$ helm repo add dev https://example.com/dev-charts
```

因为 chart repository 变化非常频繁，任何时候你可以运行 `helm repo update` 来保证你的 helm 客户端是最新的。

repository 可以使用 `helm repo remove` 移除。

## 创建你自己的 charts

[chart 开发指南](https://helm.sh/docs/topics/charts/) 解释了如何开发你自己的 chart。你可以使用 `helm create` 命令快速开始。

```
$ helm create deis-workflow
Creating deis-workflow
```

现在这里有一个 chart 在 `./deis-workflow` 目录下。你可以编辑它并创建你自己的模板（template）

编辑的同时，你可以运行 `helm list` 来检查格式是否正确。

当需要打包 chart 来分发的时候。你可以运行 `helm package` 命令：

```
$ helm package deis-workflow
deis-workflow-0.1.0.tgz
```

现在这个 chart 可以很容易地使用 `helm install` 安装了

```
$ helm install deis-workflow ./deis-workflow-0.1.0.tgz
```

打包好的 chart 可以被加载到 chart repository 中。查看文档[helm chart repository](https://helm.sh/docs/topics/chart_repository/)获取更多信息

## 结论

这个章节覆盖了基本的 helm 客户端的基本使用方式，包括查找，安装，升级和卸载。也涵盖了一些有用的命令集，比如 `helm status`，`helm get` 和 `helm repo`

要了解更多关于这些命令的信息，可以查看 helm 内置的帮助：`helm help`。

在下一个章节，我们将介绍一下 chart 的开发过程。

## 链接

- Using Helm: <https://helm.sh/docs/intro/using_helm/>

