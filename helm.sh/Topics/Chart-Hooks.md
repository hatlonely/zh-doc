# Chart 钩子

helm 提供了一个钩子（hook）机制来允许 chart 开发者在 release 生命周期的某个时间点干预。比如，你可以使用钩子来：

- 安装期间，在任何其他 chart 加载之前，加载一个 configmap 或者 secret。
- 在安装一个新的 chart 之前运行一个备份数据库的任务，然后运行第二个任务在更新完成之后回复数据。
- 在删除 release 之前，运行一个任务来优雅地让服务退出轮转。

钩子就像普通模板一样工作，但是它们有特殊的注解（annotation），使 helm 以不同的方式使用它们。在这个章节，我们会介绍钩子的基本使用模式。

## 可用的钩子

定义了如下的钩子：

| 注解 | 描述 |
|---|---|
| pre-install | 模板渲染之后，任何资源创建创建之前执行  |
| post-install | 所有资源加载到 kubernetes 之后执行 |
| pre-delete | 在删除请求中，任何资源从 kubernetes 中被删除之前执行 |
| post-delete | 在删除请求中，在所有的资源都从 kubernetes 中被删除之后执行 |
| pre-upgrade | 在更新请求中，模板渲染之后，任何资源更新之前执行 |
| post-upgrade | 在更新请求中，在所有资源都更新之后执行 |
| pre-rollback | 在回滚请求中，模板渲染之后，任何资源回滚之前执行 |
| post-rollback | 在回滚请求中，在所有资源都更新之后执行 |
| test |  在 helm 测试命令调用的时候执行 |

注意 `crd-install` 钩子已经被移除了，因为 helm 3 中有 `crds/` 目录。

## 钩子和 release 生命周期

钩子允许你，chart 开发者，有机会在 release 生命周期的关键点去执行一些操作。比如，考虑 `helm install` 生命周期。默认情况下，生命周期是这样的：

1. 用户运行 `helm install foo`
2. helm 库安装 api 被调用
3. 在一些校验之后，这个库渲染了 `foo` 的模板
4. 这个库加载这些资源到 kubernetes
5. 这个库返回 release 对象（和其他数据）给客户端
6. 客户端退出

helm 为 `install` 定义了两个钩子：`pre-install` 和 `post-install`。

如果 `foo` chart 的开发者实现了两个钩子，生命周期会发生如下变化：

1. 用户运行 `helm install foo`
2. helm 库安装 api 被调用
3. `crds/` 目录中的 CRDs 被安装
4. 在一些检验之后，这个库绚烂了 `foo` 的模板
5. 这个库准备执行 `pre-install` 钩子（加载钩子资源到 kubernetes）
6. 这个库按权重对钩子排序（默认权重为 0），按资源类型和名字升序
7. 这个库首先加载权重最低的钩子（负到正）
8. 这个库等待钩子就绪（CRDs 除外）
9. 这个库加载这些资源到 kubernetes。注意如果设置了 `--wait` 参数，这个库会等待直到所有的资源都处于就绪状态，并且不会运行 `post-install` 直到他们都处于就绪状态
10. 这个库执行 `post-install` 钩子（加载钩子资源）
11. 这个库等待钩子就绪
12. 这个库返回 release 对象（和其他数据）给客户端
13. 客户端退出

等待直到钩子继续是什么意思？这取决于钩子中声明的资源。如果资源是一个 `Job` 或者 `Pod` 类型，helm 会等待直到它成功地运行到完成。如果钩子失败，release 也会失败。这是一个阻塞操作，所以 helm 客户端会在任务运行时暂停。

对于所有其他的类型，只要 kubernetes 标记资源为已加载（已新增或者已更新），这个资源被认为是”就绪“了。如果钩子中声明了很多资源，这些资源会串行执行。如果他们有权重（看下面），他们会按权重顺序执行。从 helm 3.2.0 开始，有相同权重的钩子资源和普通非钩子资源会以相同的顺序安装。否则，顺序是不保证的（在 helm 2.3.0 以后，他们按照字母顺序排序。不过，这种行为没有考虑到绑定，未来可能会有变化）。增加一个钩子权重被认为是一种好的实践，如果权重不重要，把它设置成 0。

### 钩子资源不在 release 中管理

钩子创建的资源目前不作为 release 的一部分去追踪和管理。一旦 helm 检查一个 hook 到达了就绪状态，它就不会再管这些资源。在 release 删除时候，钩子资源的垃圾回收功能可能会在未来 helm 3 中新增，所以任何不能删除的钩子资源应该添加注解 `helm.sh/resource-policy: keep`。

特别的，这意味着你在钩子中创建了资源，你不能依赖 `helm uninstall` 来删除这些资源。要删除这些资源，你可以为钩子模板文件添加自定义的注解 `helm.sh/hool-delete-policy`，或者[设置 Job 资源的存活时间字段（TTL）](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)。

## 编写钩子

钩子只是在 kubernetes 清单文件的 `metadata` 中增加一个特殊的注解。因为他们是模板文件，你可以使用所有正常模板的特性，包括获取 `.Values`，`.Release`，`.Template`。

举个例子，这个模板，保存在 `templates/post-install-job.yaml`，声明了一个在 `post-install` 运行的 job：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}"
  labels:
    app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
    app.kubernetes.io/instance: {{ .Release.Name | quote }}
    app.kubernetes.io/version: {{ .Chart.AppVersion }}
    helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
  annotations:
    # This is what defines this resource as a hook. Without this line, the
    # job is considered part of the release.
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    metadata:
      name: "{{ .Release.Name }}"
      labels:
        app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
        app.kubernetes.io/instance: {{ .Release.Name | quote }}
        helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    spec:
      restartPolicy: Never
      containers:
      - name: post-install-job
        image: "alpine:3.3"
        command: ["/bin/sleep","{{ default "10" .Values.sleepyTime }}"]
```

让这个模板变成一个钩子的是这个注解：

```yaml
annotations:
  "helm.sh/hook": post-install
```

一个资源可以实现多个钩子：

```yaml
annotations:
  "helm.sh/hook": post-install,post-upgrade
```

类似的，对于一个钩子的实现，这里没有不同资源在数量上的限制。比如，你可以声明一个 secret 和 configmap 作为 `pre-install` 钩子。

当子 chart 声明了钩子，这些也会被评估。顶层的 chart 没有办法禁用子 chart 声明的钩子。

定义钩子的权重有助于构建一个确定的执行顺序。权重的定义使用下面的注解：

```yaml
annotations:
  "helm.sh/hook-weight": "5"
```

钩子权重可以是正数或者负数，但是必须用字符串来表示。当 helm 开始某个特定钩子的执行周期，它会按照升序排序这些钩子。

### 钩子删除策略

可以定义决定合适删除响应的钩子资源的策略。钩子删除策略的定义使用如下注解：

```yaml
annotations:
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

你可以选择一个或者多个定义的注解的值：

| 注解 | 描述 |
|--|--|
| before-hook-creation | 在新的钩子启动之前，删除前面的资源（默认）|
| hook-succeeded | 钩子成功执行后删除 |
| hook-failed | 钩子执行时失败后删除 |

如果没有指定钩子删除策略注解，`before-hook-creation` 行为是默认的。

## 链接

- Chart Hooks: <https://helm.sh/docs/topics/charts_hooks/>
