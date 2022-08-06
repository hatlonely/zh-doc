# helm 高级技术

这个章节介绍一些使用 helm 的高级特性和技术。这个章节信息面向希望在他们的 chart 和 release 上做一些高级定制和操作的 helm 的高级用户。这些高级特性，每一条都有他们自己的权衡和注意事项，所以每一条都需要小心使用，以及需要对 helm 需要很深的了解。换句话说，记住[彼得帕克原则](https://en.wikipedia.org/wiki/With_great_power_comes_great_responsibility)（能力越大，责任越大）

## 后置渲染

后置渲染让安装者可以在他们被安装之前手动操作，配置和验证渲染过得清单。这允许有高级需求的用户能够使用类似 [kustomize](https://kustomize.io/) 工具来应用配置变更，而无需 fork 公开得 chart，也无需 chart 维护者为应用指定每一个最新的配置选项。也有一些用法，在企业的环境中注入公共的工具和 sidecar，或者在部署之前分析清单。

### 先决条件

- helm 3.1+

### 用法

后置渲染器可以是任意可执行文件，从标准输入接受渲染后的 kubernetes 清单文件，然后在标准输出返回有效的 kubernetes 清单。它需要在失败的时候返回一个非 0 的退出码。这是两个组件之间唯一的 api。它赋予了你在后置渲染过程中能做的事情的极大地灵活性。

后置渲染器可以用在 `install`，`upgrade` 和 `template` 中。为了使用后置渲染器，使用 `--post-renderer` 参数，指定你想要的渲染器可执行程序的路径来使用：

```
$ helm install mychart stable/wordpress --post-renderer ./path/to/executable
```

如果路径中没有包含任何分隔符，它会从 `$PATH` 中搜索，否则，它会把相对路径转换为全路径。

如果你想要使用多个后置渲染器，在一个脚本中调用他们，或者一起放到你构建的随便什么二进制工具中。在 bash 中，这很容易，`renderer1 | renderer2 | renderer3`

你可以看下使用 `kustomize` 作为一个后置渲染器的例子，[这里](https://github.com/thomastaylor312/advanced-helm-demos/tree/master/post-render)


## 警告

在使用后置渲染器的时候，有些需要牢记于心的重要事情。最重要的是在使用后置渲染器的时候，所有修改 release 的人必须使用同一个渲染器来保证可以重复的构建。这个特性的目的是为了允许用户切换他们使用的渲染器以及停止使用一个渲染器，但是这应该谨慎地执行，以避免意外的数据修改和数据丢失。

另一个重要的注意事项是安全。如果你正在使用后置渲染器，你应该确保它来自于可信赖的源（和其他任意的可执行程序一样）。使用不受信任和未经验证的渲染器是不推荐的，因为它们对渲染过得模板有完全的权限，其中包括了密码数据。

## 自定义后置渲染器

在 Go SDK 中使用后置渲染步骤，提供了更大的灵活性。任何后置渲染器只需要实现下面的 Go 接口接口：

```go
type PostRenderer interface {
    // Run expects a single buffer filled with Helm rendered manifests. It
    // expects the modified results to be returned on a separate buffer or an
    // error if there was an issue or failure while running the post render step
    Run(renderedManifests *bytes.Buffer) (modifiedManifests *bytes.Buffer, err error)
}
```

更多使用 Go SDK 的信息，查阅 [Go SDK](./Advanced-Helm-Techniques.md#go-sdk) 章节

## Go SDK

helm 3 首次发布了完全重组的 Go SDK，为了在构建使用 helm 的应用和工具获得更好地体验。完整的文档可以在 <https://pkg.go.dev/helm.sh/helm/v3> 找到，最常用的包的简要的概览和简单的例子如下。

### 包概览

这是最常用的包列表，包括每个包的简单的解释：

- `pkg/action`：包含主要执行 helm 动作的”客户端“。这也是客户端内部正在使用的包。如果你只是需要在另一个 Go 程序中执行基本的 helm 命令，这个包是为你准备的。
- `pkg/{chart,chartutil}`：用来加载和操作 chart 的方法和辅助工具
- `pkg/cli` 和他的子包：包含所有在标准 helm 环境变量中的处理器，和他的子包包含输出和 values 文件的处理。
- `pkg/release`：定义 release 对象和状态

显然除此之外这里有更多的包，所以查阅文档了解更多信息！

### 简单例子

这是一个简单例子，使用 Go SDK 来做 `helm lint`：

```go
package main

import (
    "log"
    "os"

    "helm.sh/helm/v3/pkg/action"
    "helm.sh/helm/v3/pkg/cli"
)

func main() {
    settings := cli.New()

    actionConfig := new(action.Configuration)
    // You can pass an empty string instead of settings.Namespace() to list
    // all namespaces
    if err := actionConfig.Init(settings.RESTClientGetter(), settings.Namespace(), os.Getenv("HELM_DRIVER"), log.Printf); err != nil {
        log.Printf("%+v", err)
        os.Exit(1)
    }

    client := action.NewList(actionConfig)
    // Only list deployed
    client.Deployed = true
    results, err := client.Run()
    if err != nil {
        log.Printf("%+v", err)
        os.Exit(1)
    }

    for _, rel := range results {
        log.Printf("%+v", rel)
    }
}
```

## 存储后端

helm 3 改变了默认的 release 信息存储到 release 命名空间下的 secret 中。helm 默认存储在命名空间下的 Tiller 实例的 configmap 中。这个子章节接下来会展示如何配置不同的后端。这个配置项是基于 `HELM_DRIVER` 环境变量的。它可以被设置成下面的值：`[configmap, secret, sql]`

### configmap 存储后端

要启用 configmap 后端，你需要设置环境变量 `HELM_DRIVER` 为 `configmap`。

你可以在 shell 中进行如下设置：

```shell
export HELM_DRIVER=configmap
```

如果你想要切换默认的后端到 configmap 后端，你需要自己迁移。可以通过如下命令找到 release 信息：

```shell
kubectl get secret --all-namespaces -l "owner=helm"
```

产品说明：release 信息包括 chart 内容和 values 文件，因此可能会包含一些敏感数据（比如密码，私钥，和其他凭证），这些需要防止未授权的访问。在管理 kubernetes 授权的时候，比如 [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)，有可能授予更宽泛的 configmap 资源的访问权限，但更严格的 secret 资源的访问权限。举个例子，默认的 [user-facing role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles) 授予大多数资源查看权限，但是没有 secret。此外，secret 数据可以配置成[加密存储](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)。请记住这一点，如果你觉得切换到 configmap 后端，因为它可能会暴露你的应用的敏感数据。

### SQL 存储后端

这里有一个 beta 版本的 SQL 存储后端来存储 release 信息到 SQL 数据库中。

使用这样一个存储后端是非常有用的，如果你的发布信息超过 1MB（这种情况下，它不能存储在 configmap 以及 secret 中，因为 kubernetes 内部使用 etcd 的键值存储的限制）。

要开启 sql 后端，你需要部署一个 sql 数据库，并且设置环境变量 `HELM_DRIVER` 为 `sql`。DB 的详细信息在环境变量 `HELM_DRIVER_SQL_CONNECTION_STRING` 中设置。

你可以在 shell 中进行如下设置：

```shell
export HELM_DRIVER=sql
export HELM_DRIVER_SQL_CONNECTION_STRING=postgresql://helm-postgres:5432/helm?user=helm&password=changeme
```

注意：当前只支持 PostgreSQL

产品说明：推荐：

- 准备好数据库产品，对于 PostgreSQL，参考 [服务器管理](https://www.postgresql.org/docs/12/admin.html) 文档获取更新细节
- 为 release 信息在 kubernetes RBAC 中启用[权限管理](https://helm.sh/docs/permissions_sql_storage_backend/)

如果你想要切换默认后端到 SQL 后端，你需要自己迁移。你可以通过下面的命令搜索 release 信息：

```shell
kubectl get secret --all-namespaces -l "owner=helm"
```

## 链接

- Advanced Helm Techniques: <https://helm.sh/docs/topics/advanced/>
