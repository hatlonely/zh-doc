# 基于角色的权限控制

在 kubernetes 中，授权角色给一个用户或者特定应用服务账号是最佳实践，可以保证你的应用操作在你指定的作用域内。阅读更多关于服务账号权限在 [kubernetes 官方文档](https://kubernetes.io/docs/admin/authorization/rbac/#service-account-permissions)中。

从 kubernetes 1.6 之后，基于角色的权限控制默认启用。RBAC 允许你指定通过用户和他们在组织中的角色来决定哪种动作是被允许的。

通过 RBAC，你可以

- 授权特权操作（创建集群范围内的资源，比如新的角色）给超级管理员
- 限制用户在特定的命名空间或者集群范围作用域（资源配额，角色，用户资源定义）内创建资源的能力（pod，persistent volumes，deployments）
- 限制用户在特定的命名空间或者集群范围作用域内查看资源的权限

这个指南针对想要限制控制用户和 kubernetes API 交互作用域的超级管理员。

## 管理用户账号

所有的 kubernetes 集群有两类用户：kubernetes 管理的服务账号，以及普通用户。

普通用户被认为在外部独立的服务中管理。超级管理员分发私钥，用户的存储，像 keystone 或者 google 账号，设置一个有用户名密码列表的文件。在这方面，kubernetes 没有代表普通用户的对象。普通用户不能通过 api 调用被添加到集群中。

相反，服务账号是 kubenetes api 管理的用户。他们绑定在特定的命名空间，并且有 api 服务自动创建或者手动通过 api 调用创建。服务账号绑定了一组凭证存储在 secret 中，挂载在 pod 中允许集群内进程和 kubernetes api 交互。

api 请求被绑定到普通用户或者服务账号或者被当做匿名请求。这意味着每一个集群内或者集群外的程序，从一个人类用户在工作台键入 `kubectl`，到节点的 kubelet，到控制面板成员，必须在请求 api server 的时候认证，或者被当做一个匿名用户。

## 角色，集群角色，角色绑定以及集群角色绑定

在 kubenetes 中，用户账号和服务账号只能查看或者编辑他们被授权访问的资源。这个权限通过用户角色和角色绑定授予。用户角色和角色绑定都绑定在特定的命名空间，授予用户命名空间内角色提供的权限，使得他们有能力查看或者编辑资源。

在集群作用域，他们被叫做集群角色以及集群角色绑定。授权用户集群角色，授权他们权限查看或者编辑整个集群的资源。它也需要查看或者编辑集群作用域内的资源（命名空间，资源配额，节点）。

集群角色可以通过引用角色绑定绑定到特定的命名空间。`admin`，`edit`，`view` 默认集群角色通常像这样使用。

这是一些默认在 kubernetes 中可用的集群绑定。他们旨在成为面向用户的角色。他们包括超级用户角色（`cluster-admin`），以及更多细粒度权限的角色（`admin`，`edit`，`view`）。

| 默认集群角色 | 默认集群角色绑定 | 描述 |
|--|--|--|
| `cluster-admin` | `system:masters` 组 | 允许超级用户在任何资源上执行任何动作的权限，在使用集群角色绑定的时候，它授予了集群中所有命名空间的所有资源的完全控制。当使用一个角色绑定，它授予了角色绑定的命名空间包括命名空间本身在内的所有资源的完全控制。|
| `admin` | `None` | 允许管理员权限，旨在命名空间中使用角色绑定授权。如果使用在角色绑定中，允许大多数命名空间内资源的读写，包括在命名空间中创建角色和角色绑定的能力。它不允许资源配额和命名空间本身的写权限。|
| `edit` | `None` | 允许命名空间内读写大多数对象。它不允许查看和修改角色以及角色绑定。 |
| `view` | `None` | 允许命名空间内只读大多数对象。它不允许查看角色或者角色绑定。它不允许查看 secret，因为他们是需要升级的。 |

## 使用 RBAC 限制用户账号

现在我们明白了基本的基于角色的访问控制，让我们讨论超级管理员如何限制用户访问范围。

### 例子：授权特定命名空间的用户读写权限

为了限制用户特定的命名空间的权限，我们可以使用 `edit` 或者 `admin` 角色。如果你的 chart 创建或者和角色以及角色绑定交互，你要使用 `admin` 集群角色。

此外，你也可以创建 `cluster-admin` 权限的角色绑定。授予用户 `cluster-admin` 命名空间内的权限，提供命名空间内所有资源的完全控制，包括命名空间自己。

比如这个例子，我们创建一个带有 `edit` 角色的用户。首先，创建命名空间：

```
$ kubectl create namespace foo
```

现在，在命名空间中创建角色绑定，授予用户 `edit` 角色。

```
$ kubectl create rolebinding sam-edit
    --clusterrole edit \​
    --user sam \​
    --namespace foo
```

### 例子：在集群范围内授予用户读写权限

如果一个用户想要安装集群范围内资源（命名空间，角色，自定义资源定义，等等。）的 chart，他们需要集群范围写权限。

为了做到这个，授予用户 `admin` 或者 `cluster-admin` 权限。

授予用户 `cluster-admin` 权限就是授予他们 kubernetes 中每一个可用资源的绝对访问权限，包括使用 `kubectl drain` 的节点权限，以及其他管理任务。强烈建议提供用户 `admin` 权限来代替，或者创建一个自定义的集群角色定制他们的需求。

```
$ kubectl create clusterrolebinding sam-view
    --clusterrole view \​
    --user sam

$ kubectl create clusterrolebinding sam-secret-reader
    --clusterrole secret-reader \​
    --user sam
```

### 例子：授予用户特定命名空间只读权限

你也许已经注意到这里没有查看秘钥的集群角色。因为升级的问题，`view` 集群角色没有授权用户读取 secret 的权限。helm 默认保存 release 作为 secret。

为了能让用户运行 `helm list`，他们需要能够读取这些 secret。为了做到这个，我们会创建一个特殊的集群角色 `secret-reader`。

创建一个文件 `cluster-role-secret-reader.yaml` 并写入如下内容：

```yaml
apiVersion: rbac.authorization.k8s.io/v1​
kind: ClusterRole​
metadata:​
  name: secret-reader​
rules:​
- apiGroups: [""]​
  resources: ["secrets"]​
  verbs: ["get", "watch", "list"]
```

然后使用如下命令创建集群角色

```
$ kubectl create -f clusterrole-secret-reader.yaml​
```

做完之后，我们可以授予用户读取大部分资源的权限，然后授予他们访问 secret。

```
$ kubectl create namespace foo

$ kubectl create rolebinding sam-view
    --clusterrole view \​
    --user sam \​
    --namespace foo

$ kubectl create rolebinding sam-secret-reader
    --clusterrole secret-reader \​
    --user sam \​
    --namespace foo
```

### 授予用户集群范围内的只读访问

在特定的场景中，授予用户集群范围权限是有好处的。比如，如果用户想要运行 `helm list --all-namespaces`，api 需要用户用集群范围的读权限。

为了做到这个，同时授予用户和上面上面描述的一样的 `view` 和 `secret-reader` 权限，但是使用集群角色绑定。

```
$ kubectl create clusterrolebinding sam-view
    --clusterrole view \​
    --user sam

$ kubectl create clusterrolebinding sam-secret-reader
    --clusterrole secret-reader \​
    --user sam
```

## 额外的想法

上面例子展示了 kubernetes 默认提供的集群角色的用法。对于更多细粒度关于用户被授予哪些资源权限的控制，创建你自己自定义的角色和集群角色，查看 [kubernetes 文档](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)。

## 链接

- Role-based Access Control: <https://helm.sh/docs/topics/rbac/>
