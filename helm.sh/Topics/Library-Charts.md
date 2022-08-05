# Chart 库

chart 库是一种 helm chart 的类型，它定义了 chart 基本类型和定义，可以被其他的 chart 复用 helm 模板。这允许用户分享他们可以复用的的代码片段，避免重复和保持 chart 的 [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)

chart 库在 helm 3 中引入，用于正式识别公共和辅助 chart，在 helm 2 中已经被 chart 维护者使用。作为一种 chart 类型，它提供了：

- 一种含义来显示区分公共和应用 chart
- 逻辑上公共 chart 不会安装
- 公共库中没有模板渲染，即使可能包含要发布的东西

chart 的维护者可以定义一个公共 chart 作为一个库 chart，并且现在可以放心这样做，helm 会以标准一致的方式处理 chart。这也意味着，应用 chart 中的定义也可以在修改 chart 类型之后被复用。

## 创建一个简单的 chart 库

就像前面提到的那样，chart 库是 helm chart 的一种类型。这意味着你可以从穿件一个 chart 脚手架开始：

```
$ helm create mylibchart
Crateing mylibchart
```

首先移除所有 `templates` 目录下的文件，因为我们会在这个例子中创建我们自己的模板定义。

```
$ rm -rf mylibchart/template/*
```

`values.yaml` 文件也是不需要的。

在我们跳到创建公共代码之前，让我们来快速回顾一下一些相关的 helm 概念。一个命名的模板（有时候也叫组件或者子模板）仅仅是一个文件中定义的模板，并且给定了名字。在 `templates/` 目录中，任何以下划线（`_`）开头的文件都不应该输出到 kubernetes 清单文件中。所以，习惯上，辅助模板和组件都被放到了 `_*.tpl` 或者 `_*.yaml` 文件中。

在这个例子中，我们会编写一个 configmap，这会创建一个空的 configmap 资源。我们会定义一个空的 configmap 文件 `mylibchart/templates/_configmap.yaml` 如下：

```yaml
{{- define "mylibchart.configmap.tpl" -}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name | printf "%s-%s" .Chart.Name }}
data: {}
{{- end -}}
{{- define "mylibchart.configmap" -}}
{{- inclue "mylibchart.util.merge" (append . "mylibchart.configmap.tpl") -}}
{{- end -}}
```

configmap 的结构定义在命名模板 `mylibchart.configmap.tpl` 中。它是一个没有资源的简单 configmap。在这个文件里面，有另一个命名模板叫做 `mylibchart.configmap`。这个命名模板包含了另一个模板 `mylibchart.util.merge`，它使用两个命名模板作为参数，分别是 `mylibchart.configmap` 和 `mylibchart.configmap.tpl`。

辅助函数 `mylibchart.util.merge` 是一个在 `mylibchart/template/_util.yaml` 中的命名模板。它是一个在[公共 helm 辅助 chart](https://helm.sh/docs/topics/library_charts/#the-common-helm-helper-chart)中的方面的工具，因为它合并了两个模板，并且覆盖了两者的公共部分：

```yaml
{{- /*
mylibchart.util.merge will merge two YAML templates and output the result.
This takes an array of three values:
- the top context
- the template name of the overrides (destination)
- the template name of the base (source)
*/}}
{{- define "mylibchart.util.merge" -}}
{{- $top := first . -}}
{{- $overrides := fromYaml (include (index . 1) $top) | default (dict ) -}}
{{- $tpl := fromYaml (include (index . 2) $top) | default (dict ) -}}
{{- toYaml (merge $overrides $tpl) -}}
{{- end -}}
```

当一个 chart 想要使用公共代码来自定义它的配置项的时候很重要。

最后，让我们改变 chart 的类型为 `library`。这需要编辑 `mylibchart/chart.yaml` 如下：

```yaml
apiVersion: v2
name: mylibchart
description: A Helm chart for Kubernetes

# A chart can be either an 'application' or a 'library' chart.
#
# Application charts are a collection of templates that can be packaged into versioned archives
# to be deployed.
#
# Library charts provide useful utilities or functions for the chart developer. They're included as
# a dependency of application charts to inject those utilities and functions into the rendering
# pipeline. Library charts do not define any templates and therefore cannot be deployed.
# type: application
type: library

# This is the chart version. This version number should be incremented each time you make changes
# to the chart and its templates, including the app version.
version: 0.1.0

# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application and it is recommended to use it with quotes.
appVersion: "1.16.0"
```

这个 chart 库现在可以用来分享了，它的 configmap 定义可以被复用了。

继续之前，需要检查一下 helm 是否识别这个 chart 作为 chart 库：

```
$ helm install mylibchart mylibchart/
Error: library charts are not installable
```

## 使用 chart 库

是时候使用 chart 库了。这意味着再次创建一个 chart 脚手架：

```
$ helm create mychart
Creating mychart
```

让我们再次清理模板文件，因为我们只想要创建一个 configmap：

```
$ rm -rf mychart/templates/*
```

当我们想要在 helm 模板中创建一个简单的 configmap，它看起来类似下面这样：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name | printf "%s-%s" .Chart.Name }}
data:
  myvalue: "Hello World"
```

然而我们将要复用已经在 `mylibchart` 中创建的公共代码。configmap 会在文件 `mychart/tempaltes/configmap.yaml` 中创建，如下：

```yaml
{{- include "mylibchart.configmap" (list . "mychart.configmap") -}}
{{- define "mychart.configmap" -}}
data:
  myvalue: "Hello World"
{{- end -}}
```

你可以看到，继承了公共的 configmap 定义之后在 configmap 中增加标准的属性，我们要做的事情很简单。在我们的模板中增加配置，这个例子中，`data` 中的键 `myvalue` 以及它的值。这个配置覆盖了公共 configmap 中空的资源。这是可行的，因为我们前面提到的辅助方法 `mylibchart.util.merge`。

为了复用公共代码，我们需要添加 `mylibchart` 作为依赖。增加如下内容到文件的最后 `mychart/Chart.yaml`：

```yaml
# My common code in my library chart
dependencies:
- name: mylibchart
  version: 0.1.0
  repository: file://../mylibchart
```

这包含了 chart 库作为一个来自文件系统的动态的依赖，和我们的 chart 应用在同一个父目录中。因为我们以动态依赖的方式包含了这个 chart 库，我们需要运行 `helm dependency update`。它会复制这个 chart 库到你的 `charts/` 目录。

```
$ helm dependency update mychart/
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Deleting outdated charts
```

我们现在准备发布我们的 chart 了。在安装之前，首先需要检查渲染的模板。

```
$ helm install mydemo mychart/ --debug --dry-run
install.go:159: [debug] Original chart version: ""
install.go:176: [debug] CHART PATH: /root/test/helm-charts/mychart

NAME: mydemo
LAST DEPLOYED: Tue Mar  3 17:48:47 2020
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
affinity: {}
fullnameOverride: ""
image:
  pullPolicy: IfNotPresent
  repository: nginx
imagePullSecrets: []
ingress:
  annotations: {}
  enabled: false
  hosts:
  - host: chart-example.local
    paths: []
  tls: []
mylibchart:
  global: {}
nameOverride: ""
nodeSelector: {}
podSecurityContext: {}
replicaCount: 1
resources: {}
securityContext: {}
service:
  port: 80
  type: ClusterIP
serviceAccount:
  annotations: {}
  create: true
  name: null
tolerations: []

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
data:
  myvalue: Hello World
kind: ConfigMap
metadata:
  labels:
    app: mychart
    chart: mychart-0.1.0
    release: mydemo
  name: mychart-mydemo
```

看起来 configmap 我们想要用 `myvalue: Hello World` 覆盖 data。让我们安装它：

```
$ helm install mydemo mychart/
NAME: mydemo
LAST DEPLOYED: Tue Mar  3 17:52:40 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

我们可以获取这个 release 来看真正加载的模板。

```
$ helm get manifest mydemo
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
data:
  myvalue: Hello World
kind: ConfigMap
metadata:
  labels:
    app: mychart
    chart: mychart-0.1.0
    release: mydemo
  name: mychart-mydemo
```

## 公共 Helm 辅助 chart

> 注意：公共 Helm 辅助模板仓库在 github 上已经不维护了，并且这个仓库已经废弃和存档了。

这个 chart 是最开始的公共 chart 模式。它提供了反映 kubernetes chart 开发最佳实践的工具集。最重要的是，他可以在你开发 chart 的时候立即使用，并且方便你的代码分享。

这是一个快速使用它的方式。更多信息，查看[README](https://github.com/helm/charts/blob/master/incubator/common/README.md)。

再次创建一个 chart 脚手架：

```
$ helm create demo
Creating demo
```

让我们使用辅助 chart 的公共代码。首先，编辑 deployment `demo/templates/deployments.yaml` 如下：

```yaml
{{- template "common.deployment" (list . "demo.deployment") -}}
{{- define "demo.deployment" -}}
## Define overrides for your Deployment resource here, e.g.
apiVersion: apps/v1
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "demo.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "demo.selectorLabels" . | nindent 8 }}

{{- end -}}
```

然后 service 文件，`demo/templates/service.yaml` 如下：

```yaml
{{- template "common.service" (list . "demo.service") -}}
{{- define "demo.service" -}}
## Define overrides for your Service resource here, e.g.
# metadata:
#   labels:
#     custom: label
# spec:
#   ports:
#   - port: 8080
{{- end -}}
```

这些模板展示了如何从辅助 chart 继承公共代码来简化配置和自定义资源的代码。

为了使用公共代码，你需要添加 `common` 作为一个依赖。在 `demo/Chart.yaml` 文件尾部添加如下内容：

```yaml
dependencies:
- name: common
  version: "^0.0.5"
  repository: "https://charts.helm.sh/incubator/"
```

注意：你需要添加 `incubator` 仓库到你的 helm repository 列表中（`helm repo add`）。

因为我们以动态依赖的方式包含了 chart，我们需要运行 `helm dependency update`。他会复制辅助 chart 到 `charts/` 目录。

因为辅助 chart 使用了一些 helm 2 的结构，你需要添加如下内容到 `demo/values.yaml` 来启用 `nginx` 镜像的加载，因为这在 helm 3 的脚手架 chart 中被更新了。

```yaml
image:
  tag: 1.16.0
```

你可以在部署之前测试这个 chart 模板是否正确，使用 `helm lint` 和 `helm tempalte` 命令。

如果没问题，使用 `helm install` 部署！

## 链接

- Library Charts: <https://helm.sh/docs/topics/library_charts/>
