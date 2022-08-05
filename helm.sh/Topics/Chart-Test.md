# Chart 测试

chart 包含了很多一起工作的 kubernetes 资源和组件。作为一个 chart 作者，你可能想要写一些测试来验证你的 chart 在安装的时候运行符合预期。这些测试也帮助 chart 使用者明白你的 chart 应该是什么样子的。

helm chart 里面的测试在 `templates/` 目录下，并且是指定容器运行给定命令的任务（job）定义。这个容器应该要成功退出（退出码是0），这样测试才认为是成功的。这个任务定义必须包含 helm 的测试钩子注解 `helm.sh/hook: test`。

注意，helm v3 之前，这个任务定义需要包含下面这些 helm 测试钩子注解的中一个：`helm.sh/hook: test-success` 或者 `helm.sh/hook: test-failure`。`helm.sh/hook: test-success` 现在仍然是可用的，作为向后兼容的可选方案：`helm.sh/hook: test`

测试例子：

- 检查 `values.yaml` 中的配置都被正确的注入了
    - 确保你的用户名密码都正常工作
    - 确保一个不正确的用户名密码不能正常工作
- 检查你的服务都已经启动，并且负载均衡正确
- 等等

你可以用 `helm test <RELEASE_NAME>` 命令来运行一个预定义的 helm 测试。对于 chart 使用者，这是一个好方法来检查他们的 release 工作正常。

## 测试例子

`helm create` 命令会自动创建一些目录和文件。先创建一个 helm chart 样例来试试 helm 的测试功能。

```
$ helm create demo
```

你会在你的 helm chart 样例中看到下面这个目录结构。

```
demo/
    Chart.yaml
    values.yaml
    charts/
    templates/
    templates/tests/test-connection.yaml
```

在 `demo/templates/tests/test-connection.yaml` 中你可以看到一个你可以试试的测试。你可以看到 helm 测试 pod 的定义如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "demo.fullname" . }}-test-connection"
  labels:
    {{- include "demo.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "demo.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

## 在 release 中运行测试步骤

首先，安装这个 chart 到你的集群中来创建一个 release。你可能需要等待所有的 pod 都处于活跃状态；如果在安装之后立马测试，很可能会失败，你可能会重新测试。

```
$ helm install demo demo --namespace default
$ helm test demo
NAME: demo
LAST DEPLOYED: Mon Feb 14 20:03:16 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE:     demo-test-connection
Last Started:   Mon Feb 14 20:35:19 2022
Last Completed: Mon Feb 14 20:35:23 2022
Phase:          Succeeded
[...]
```

## 注意

- 你可以在一个单独的 yaml 文件中定义很多测试，也可以分开几个 yaml 文件到 `templates/` 目录中。
- 欢迎内嵌你的测试到 `tests/` 目录下，就像 `<chart-name>/templates/tests/` 以更好地隔离。
- 测试是一个 helm 钩子，所以有时候像 `helm.sh/hook-weight` 和 `helm.sh/hook-delete-policy` 有可能会在测试资源中使用

## 链接

- Chart Tests: <https://helm.sh/docs/topics/chart_tests/>
