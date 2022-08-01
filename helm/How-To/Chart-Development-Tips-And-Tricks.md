# Chart 开发的指导和技巧

本指南覆盖了一些开发者在构建产品级别 chart 时需要学习的指导和技巧

## 了解模板方法

helm 使用[go 模板](https://godoc.org/text/template)来渲染你的资源文件。go 本身仅仅附带几个内置的方法，我们添加了很多其他的。

首先，我们添加了所有的 [Sprig 库](https://masterminds.github.io/sprig/)的方法，因为安全原因，`env` 和 `expandenv` 除外

我们也添加了两个特别的模板方法：`include` 和 `required`。`include` 方法允许你传递一个模板的结果到到其他的模板方法中。

比如，这个模板片段包含了一个名为 `mytpl` 的模板，然后将结果小写之后包装在双引号之中。

```
value: {{ include "mytpl" . | lower | quote }}
```

`required` 方法允许你声明一个特别的值入口，表示在模板渲染时必填。如果这个值是空，模板渲染会失败，并显示一个用户定义的错误信息。

下面这个关于 `required` 方法的例子，声明了一个 `.Values.who` 的入口是必要的，并且当这个入口缺失时会打印一条错误信息：

```yaml
value: {{ required "A valid .Values.who entry required!" .Values.who }}
```

## 给字符串加引号，不要给整型加引号

当你使用字符串的时候，用引号引起来总会比不引起来要安全：

```yaml
name: {{ .Values.MyName | quote }}
```

但是使用数字的时候千万不要加引号。这将在大多数情况下导致 kubernetes 内部的解析错误

```yaml
port: {{ .Values.Port }}
```

此备注不使用于环境变量，因为本来就需要字符串，即使他们本身代表的是整型。

```yaml
env:
  - name: HOST
    value: "http://host"
  - name: PORT
    value: "1234"
```

## 使用 `include` 函数

go 提供了一种包含另一个模板的方式，直接使用内置的 `template`。然而这个内置的方法不能用在 go 模板管道之中。

为了能够包含一个模板，并且对模板的输出进行操作，helm 特别提供了 `include` 方法：

```
{{ include "toYaml" $value | indent 2 }}
```

上例包含了一个名为 `toYaml` 的模板，将 `$value` 传给它，然后将模板输出传递给 `indent` 方法。

因为 YAML 的特点是缩进等级和空白非常重要，这是一个很好的方式，可以包含代码片段，但是在相关上下文中处理缩进。

## 使用 `required` 方法

Go 提供了一种设置模板选项的方式来控制一个 map 索引了一个未出现在 map 中的键的行为。典型的就是设置 `template.Options("missingkey=option")`，这里的 `option` 可以是 `default`，`zero` 或者 `error`。当这个选项设置为 `error` 时将停止执行，并抛出这个错误，这条规则将应用在 map 中所有缺失的键中。某些情况下，开发者想要增强这种行为，只在 values.yaml 文件中选择一些值。

`required` 方法让开发者拥有了这样的能力，声明一些值入口在模板渲染时是必须的。如果这个入口在 values.yaml 中是空的，这个模板将不会渲染，并且返回一个开发者提供的错误消息。

比如：

```
{{ required "A valid foo is required!" .Values.foo }}
```

上面这个例子会在 `.Values.foo` 定义时渲染模板，但是会在 `.Values.foo` 未定义时渲染失败并退出。

## 使用 `tpl` 函数

`tpl` 方法允许开发者在一个模板内部计算字符串作为模板。这在传递一个模板字符串作为值到一个 chart 中或者渲染外部配置文件的时候非常有用。语法：`{{ tpl TEMPLATE_STRING VALUES }}`

例如：

```
# values
template: "{{ .Values.name }}"
name: "Tom"

# template
{{ tpl .Values.template . }}

# output
Tom
```

渲染一个外部的配置文件

```
# external configuration file conf/app.conf
firstName={{ .Values.firstName }}
lastName={{ .Values.lastName }}

# values
firstName: Peter
lastName: Parker

# template
{{ tpl (.Files.Get "conf/app.conf") . }}

# output
firstName=Peter
lastName=Parker
```

## 创建镜像拉取密钥

镜像拉取密钥（image pull secret）本质上是一个镜像仓库（registry），用户名（username），密码（password）的组合。你可能会在部署应用的时候需要它们，但是创建它们需要运行多次 `base64` 方法。我们可以写一个辅助模板来组成 docker 配置文件，用作密钥的负载。这是一个例子：

首先，假设这些凭证在 `values.yaml` 中定义如下：

```
imageCredentials:
  registry: quay.io
  username: someone
  password: sillyness
  email: someone@host.com
```

然后定义我们的辅助模板如下：

```
{{- define "imagePullSecret" }}
{{- with .Values.imageCredentials }}
{{- printf "{\"auths\":{\"%s\":{\"username\":\"%s\",\"password\":\"%s\",\"email\":\"%s\",\"auth\":\"%s\"}}}" .registry .username .password .email (printf "%s:%s" .username .password | b64enc) | b64enc }}
{{- end }}
{{- end }}
```

最后，我们使用辅助模板在更大的模板中创建密钥清单

```
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: {{ template "imagePullSecret" . }}
```

## 自动滚动部署

通常情况下，configmap 或者 secret 会被作为配置文件注入到容器中，或者有其他的依赖变化时需要滚动 pod。根据应用的不同，随后的 `helm upgrade` 升级，这些更新可能会导致应用的重启，但是如果部署规格本身没有改变应用程序保持老的配置运行，将导致部署的不一致。

`sha256sum` 方法可以用来保证部署的注解部分在文件变化后是最新的：

```yaml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
[...]
```

注意：如果你添加这个到 chart 库中，你将无法通过 `$.Template.BasePath` 访问你的文件。相反你可以通过 `{{ include ("mylibchart.configmap") . | sha256sum }}` 引用你的定义。

如果你总是希望滚动更新，你可以使用类似的上面步骤的注解，替换成随机的字符串，所以它总是会变化，并且导致滚动更新：

```yaml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        rollme: {{ randAlphaNum 5 | quote }}
[...]
```

## 链接

- Chart Development Tips and TricksL: <https://helm.sh/docs/howto/charts_tips_and_tricks/>
