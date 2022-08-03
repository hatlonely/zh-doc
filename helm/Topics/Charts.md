# Chart

helm 用来打包的格式叫做 chart。一个 chart 是描述一组相关的 kubernetes 资源的文件集合。一个单独的 chart 可以用来部署一些简单的东西，比如 memcached 的 pod，或者一些复杂的东西，比如一个完整的 web app 技术栈，包括 http 服务，数据库，缓存等等。

chart 作为特定目录树中的文件被创建。他们可以被打包成一个用于部署的带版本的存档。

如果你想要下载并且查看已发布的 chart 中的文件，不需要安装它，只需要 `helm pull chartrepo/chartname` 即可。

本文档说明了 chart 格式，并且提供了基本的使用 helm 构建 chart 的基本指导。

## chart 文件结构

chart 在目录里以文件的集合的方式组织。目录名字和 chart 的名字相同（没有版本信息）。因此，一个用来描述 wordpress 的 chart 会被保存在一个名为 `wordpress/` 的目录中。

在这个目录里面，helm 期望的结构如下：

```
wordpress/
  Chart.yaml          # 包含 chart 信息的 Yaml 文件
  LICENSE             # 可选：包含 chart 授权信息的文本文件
  README.md           # 可选：可读的自描述文件
  values.yaml         # chart 的默认配置
  values.schema.json  # 可选：json 格式的 values.yaml 文件
  charts/             # 包含其他依赖 chart 目录
  crds/               # 用户自定义资源
  templates/          # 模板目录，结合 values 可以生成有效的 kubernetes 清单文件
  templates/NOTES.txt # 可选：包含简要使用说明的文本文件
```

helm 保留了使用 `charts`，`crds`，和 `templates` 目录，以及上面列举的文件名。其他文件可以保持他们原始的样子。

## Chart.yaml 文件

Chart.yaml 对于 chart 来说是必要的文件。它包含了如下的字段：

```yaml
apiVersion: chart 的 api 版本 (必填)
name: chart 的名字 (必填)
version: chart 的版本 (必填)
kubeVersion: 兼容的 kubenetes 版本 (可选)
description: 一句话的项目描述 (可选)
type: chart 类型 (可选)
keywords:
  - 项目的关键词列表 (可选)
home: 项目主页的 URL (可选)
sources:
  - 项目源代码的 URL 列表 (可选)
dependencies: # chart 的依赖列表 (可选)
  - name: 依赖 chart 名字 (nginx)
    version: 依赖的 chart 版本 ("1.2.3")
    repository: (可选) 依赖 repository URL ("https://example.com/charts") 或者别名 ("@repo-name")
    condition: (可选) 解析为布尔值的 yaml 路径, 用来开启或者禁用 chart (e.g. subchart1.enabled )
    tags: # (可选)
      - tags 可以用来给 chart 分组，这样可以同时开启或者禁用一组 chart
    import-values: # (可选)
      - ImportValues 保留了源键到导入父键的映射。每个条目可以是一个字符串，或者是一堆子/父列表项。
    alias: (可选) 使用这个 chart 的别名. 在你需要多次添加相同 chart 的时候很有用
maintainers: # (可选)
  - name: 维护者名字 (必填)
    email: 维护者邮箱 (可选)
    url: 维护者 URL (可选)
icon: 用来作为图标的 SVG 或者 PNG URL (可选)
appVersion: app 版本 (可选). 不需要 SemVer. 建议使用引号
deprecated: chart 是否已经废弃 (可选，布尔值)
annotations:
  example: 按名字输出的批注列表 (可选)
```

从 v3.3.2 开始，不允许再有额外的字段。推荐的方式是添加自定义的元数据信息在 `annotations` 中。

### version

每一个 chart 都必须有版本号。一个版本号必须遵循 [SemVer2](https://semver.org/spec/v2.0.0.html) 规范。和老版本的 helm 不一样，helm v2 之后使用版本号作为发布标记。repository 里面的包通过名字和版本来标识。

举个例子，nginx chart 的版本字段是 `version: 1.2.3`，那就会被命名为 `nginx-1.2.3.tgz`

更多复杂的 SemVer2 名字也是支持的，比如 `version: 1.2.3-alpha.1+ef365`。但是不符合 SemVer 规范的名字是被系统显示拒绝的。

注意：说到 chart，虽然老版本的 helm 还是部署管理器都是面向 Github 的，但是 helm v2 及其后续版本就不再依赖 Github 甚至 Git 了。所以，也不再使用 Git SHAs 作版本控制。

`Chart.yaml` 中的 `version` 字段在很多 helm 工具中都有使用，包括 helm 客户端。打包的时候，`helm package` 命令会使用从 `Chart.yaml` 中找到的版本号作为 token 放到包名中。系统假设这个 chart 包的版本号和 `Chart.yaml` 中的版本号是匹配的。不满足这个假设会导致错误。

### apiVersion

对于至少需要 helm 3 的 chart，`apiVersion` 字段应该是 v2。chart 支持老版本的 helm 的 apiVersion 是 v1，并且仍然可以使用 helm 3 来安装。

从 v1 改成 v2：

- [`dependencies`](#dependencies) 字段定义了 chart 的依赖，和 v1 chart 中的 `requirements.yaml` 文件分开在不同的地方。
- [`type`](#type) 字段，用于区别应用类型和库类型的 chart。

### appVersion

注意 `appVersion` 字段和 `version` 字段没有关系。它是一种指定应用版本的方式。举例来说，`drupal` chart 有一个 app 版本 `appVersion: "8.2.1"`，表明 chart 中（默认情况下）的 drupal 的版本是 8.2.1。这个字段是一个描述性的信息，不影响 chart 版本的计算。强烈建议用引号将版本引起来。它能强制 Yaml 解析器将版本号当成一个字符串。不加引号在一些场景下可能会导致一些问题。比如，Yaml 会将 `1.0` 当成浮点数来处理，一个 git commit SHA 比如 `1234e10` 会被当成科学计数法。

helm v3.5.0 以后，`helm create` 默认会将 `appVersion` 字段用引号引起来。

### kubeVersion

可选字段 `kubeVersion` 可以定义支持的 kubernetes 版本约束。helm 会在安装 chart 的时候检查版本约束，并且在不支持的 kubernetes 版本中运行会失败。

版本约束包含空白分隔和比较运算符，比如

```
>= 1.13.0 < 1.15.0
```

他们自身可以通过或运算符（`||`）组合起来，就像下面这样

```
>= 1.13.0 < 1.14.0 || >= 1.14.1 < 1.15.0
```

在这个例子中，版本号 `1.14.0` 是不包含的，如果已知某些版本有 bug 导致 chart 不能正常运行，这是很有意义的。

除了版本约束外，使用运算符（`=`，`!=`，`>`，`<`，`>=`，`<=`）支持以下速记符号。

- 连字符（`-`）连接闭合区间吗，`1.1-2.3.4` 相当于 `>= 1.1 <= 2.3.4`
- 通配符 `x`，`X` 和 `*`，`1.2.x` 相当于 `>= 1.2.0 < 1.3.0`
- 波浪号（`~`）（允许补丁版本），`~1.2.3` 相当于 `>= 1.2.3 < 1.3.0`
- 脱字符（`^`）（允许小版本），`^1.2.3` 相当于 `>= 1.2.3 < 2.0.0`

更多关于支持版本约束的细节解释参考 [Masterminds/semverMasterminds/semver](https://github.com/Masterminds/semver)

### deprecated

在 chart repository 里面管理 chart 的时候，有时候需要废弃一个 chart。`Chart.yaml` 里面的可选字段 `deprecated` 可以用来标记 chart 为废弃状态。如果 repository 中最新版本的 chart 被标记为废弃，那整个 chart 都会被认为已废弃。chart 名字可以重新发布一个新的版本并且不标记废弃。废弃 chart 的流程是这样：

1. 更新 `Chart.yaml` 来标记 chart 为废弃，并更改版本
2. 发布一个新的 chart 版本到 chart repository
3. 从源代码仓库汇总删除这个 chart（比如 git）

### type

type 字段定义了 chart 的类型。有两种类型 `application` 和 `library`。`application` 是默认的类型，它是标准的 chart，可以进行所有的操作。chart 库（`library` chart） 给 chart 构建者提供了程序和功能。chart 库和应用 chart 是不同的，因为 chart 库是不能安装的，所以通常不包含任何资源对象。

注意：一个应用 chart 也可以被当成 chart 库。只需要设置 type 字段为 library 即可。这个 chart 就会被渲染成一个库 chart，所有的程序和功能都可以被使用。所有的资源对象都将不会被渲染。

## LICENSE，README，NOTES

chart 也可以包含安装，配置，使用和授权许可文件。

`LICENSE` 是一个包含 chart 授权的纯文本文件。chart 里面包含一个许可，因为里面可能包含程序逻辑，而不仅仅只有配置。如果有必要的话，也可能会有单独的应用安装授权许可。

`README` 是一个 markdown 格式的文件，通常会包含：

- chart 提供的应用程序或者服务的描述
- chart 运行所需要的先决条件和要求
- `values.yaml` 中选项的描述和默认值
- 任何其他可能和 chart 安装和配置相关的信息

在 hub 或者其他用户界面显示的关于 chart 的详细信息都是来自于 `README.md` 文件的内容

chart 也会包含一个短的文本文件 `templates/NOTES.txt`，这个文件会在安装之后或者查看 release 状态的时候打印。这个文件会被当成一个模板，可以用来展示使用说用，下一步，或者任何其他和 release 相关的信息。比如，连接数据库的指令，或者访问 web ui。因为这个文件在运行 `helm install` 或者 `helm status` 的时候输出到标准输出，建议保持这个文件的简短，并指向 README 文件获取更多细节。

## chart 依赖

在 helm 中，一个 chart 可以依赖任何数量的其他 chart。这些依赖可以通过 `Chart.yaml` 中的 `dependencies` 字段动态链接，或者放到 `charts` 目录中手动管理。

### 使用 dependencies 字段管理依赖

在 `dependencies` 字段里，当前 chart 依赖的其他 chart 会被定义成一个列表。

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: https://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: https://another.example.com/charts
```

- `name` 字段是你需要的 chart 的名字
- `version` 字段是你需要的 chart 版本
- `repository` 字段是 chart repository 的 url 全路径。注意你也需要使用 `helm repo add` 添加这个 repo 到本地
- 你可以使用 repo 的名字来替代 url

```
$ helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com
```

```yaml
dependencies:
  - name: awesomeness
    version: 1.0.0
    repository: "@fantastic-charts"
```

一旦你定义了依赖，你可以运行 `helm dependency update`，它会使用你的依赖文件来下载所有你指定的 chart 到 `charts/` 目录。

```
$ helm dep up foochart
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "example" chart repository
...Successfully got an update from the "another" chart repository
Update Complete. Happy Helming!
Saving 2 charts
Downloading apache from repo https://example.com/charts
Downloading mysql from repo https://another.example.com/charts
```

当 `helm dependency update` 拉取 chart 时，会在 `charts/` 目录保存他们的 chart 存档。所以上面的例子中，将在 `charts` 目录中看到如下文件：

```
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```

### dependencies 中的 alias 字段

除了上面的字段之外，每一个依赖也会包含一个可选的字段 `alias`。

增加一个依赖 chart 的别名，会使用别名作为新依赖的名称。

在 chart 重名的情况下可以使用别名。

```yaml
# parentchart/Chart.yaml

dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-1
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    alias: new-subchart-2
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
```

在上面的例子中，我们总共会得到 parentchart 的 3 个依赖项

```
subchart
new-subchart-1
new-subchart-2
```

手动实现这个的方式是多次使用不同的名字复制粘贴相同的 chart 到 `charts/` 目录。

### dependencies 中的 tags 和 condition 字段

除了上面的字段之外，每一个依赖也会包含可选字段 `tags` 和 `condition`。

所有的 chart 默认都会加载。如果设置了 `tags` 和 `condition` 字段，它们将被计算并且用来控制它们应用的 chart 的加载。

`condition` 字段包含一个或者多个 yaml 路径（使用逗号隔开）。如果在最上层的存在这个路径，并且被解析成布尔值，这个 chart 就会被启用或者禁用，取决于布尔值。只有找到列表中第一个有效的路径会被评估，并且如果所有的路径都不存在，这个条件也将无效。

`tags` 字段是一个用来关联 chart 的 yaml 列表。最上层的值中，所有带 tag 的 chart 可以通过指定 tag 和布尔值来启用或者禁用。

```yaml
dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart1.enabled, global.subchart1.enabled
    tags:
      - front-end
      - subchart1
  - name: subchart2
    repository: http://localhost:10191
    version: 0.1.0
    condition: subchart2.enabled,global.subchart2.enabled
    tags:
      - back-end
      - subchart2
```

```yaml
# parentchart/values.yaml

subchart1:
  enabled: true
tags:
  front-end: false
  back-end: true
```

在上面的例子中，所有带有 `front-end` tag 的 chart 都会被禁用，但是因为 `subchart1.enabled` 这个路径计算结果为 `true`，这个条件会覆盖 `front-end` tag，`subchart1` 会被启用。

由于 `subchart2` 带有 `back-end` 的 tag，并且这个 tag 计算结果为 `true`，`subchart2` 会被启用。同时注意尽管 `subchart2` 也指定了条件，但是上层的值中没有相应的路径和值，所以条件没有生效。

#### 在客户端中使用 tags 和 condition

`--set` 参数通常可以用来修改 tag 和 condition 的值

```
$ helm install --set tags.front-end=true --set subchart2.enabled=false
```

#### tags 和 condition 的解析

- condition（在 values 中设置时）总是会覆盖 tag。第一个存在的 condition 路径赢了，其他的会被忽略。
- tags 在计算的时候会被当做“任何一个 chart 的 tag 为 true 时，启用 chart”
- tags 和 condition 的值必须在最上层设置
- tags 键必须在 values 的最顶层。全局和内嵌的 tags 目前还不支持

### 通过依赖项导入子值

在一些场景中，需要允许子 chart 的值冒泡到父 chart 中，并且共享同样的默认值。使用 `export` 格式的另一个好处是，它能使用未来的工具检查用户设置的值。

包含被到处值的 key 可以在父 chart 的 dependencies 的 import-values 中使用 yaml 列表指定。列表中的每条都是一个从子 chart 的 `exports` 字段导入的键

为了导入没有在 `exports` 中的键，使用 `child-parent` 格式。两中格式的例子如下。

#### 使用 exports 格式

如果一个字 chart 的 `values.yaml` 文件在根目录下包含了 `exports` 字段，它的内容就能直接导入到父 chart 的 values 中，通过指定 `import-values` 即可，如下面的例子：

```yaml
# parent's Chart.yaml file

dependencies:
  - name: subchart
    repository: http://localhost:10191
    version: 0.1.0
    import-values:
      - data
```

```yaml
# child's values.yaml file

exports:
  data:
    myint: 99
```

因为我们指定了 `data` 这个键在我们的导入列表中，helm 在子 chart 的 `exports` 字段中查找 `data` 键，并导入它的内容。

请注意父键 `data` 并没有在父 chart 的最终 values 中包含。如果你需要指定父 chart 的键，使用 `child-parent` 格式。

#### 使用 child-parent 格式

为了访问没有包含在 `exports` 中的子 chart values 中的键，你需要指定需要被导入的源键和在父 chart values 中的目标路径。

`import-values` 在下面这个例子中，指示 helm 使用任何在子 chart values 找到的 `child` 路径，拷贝他们到父 chart values `parent` 路径。

```
# parent's Chart.yaml file

dependencies:
  - name: subchart1
    repository: http://localhost:10191
    version: 0.1.0
    ...
    import-values:
      - child: default.data
        parent: myimports
```

在上面的例子中，subchart1 的 values 中的 `default.data` 会被导入到父 chart values 的 myimports 键，细节如下：

```
# parent's values.yaml file

myimports:
  myint: 0
  mybool: false
  mystring: "helm rocks!"
```

```
# subchart1's values.yaml file

default:
  data:
    myint: 999
    mybool: true
```

父 chart 的结果会是：

```
# parent's final values

myimports:
  myint: 999
  mybool: true
  mystring: "helm rocks!"
```

父 chart 最终的 values 现在包含了从 subchart1 导入的 `myint` 和 `mybool` 字段。

### 通过 charts/ 目录手动管理依赖

如果需要更多对依赖的控制，可以通过拷贝这些依赖的 chart 到 `charts/` 目录来显示的表示这些依赖。

依赖应该是未压缩的 chart 目录，但是他的名字不能以 `_` 或者 `.` 开头，这些文件会被 chart 加载器忽略。

举个例子，如果 wordpress chart 依赖了 apache chart，那 apache chart（版本正确）会在 wordpress chart 的 `charts` 目录被提供。

```
wordpress:
  Chart.yaml
  # ...
  charts/
    apache/
      Chart.yaml
      # ...
    mysql/
      Chart.yaml
      # ...
```

上面这个例子展示了 wordpress 如何表示它对 apache 和 mysql 的依赖，通过在 `charts/` 目录中包含这些 chart。

提示：要讲依赖放入 `chart/` 目录，使用 `helm pull` 命令

### 使用依赖的操作部分

上面的章节解释了如何指定一个 chart 的依赖，但是这个如何影响 chart 的安装，在使用 `helm install` 和 `helm upgrade` 的时候。

假设一个名为 A 的 chart 创建了如下的 kubernetes 对象

- namespace "A-Namespace"
- statefulset "A-StatefulSet"
- service "A-Service"

此外，A 依赖于 chart B，B 创建了对象

- namespace "B-Namespace"
- replicaset "B-ReplicaSet"
- service "B-Service"

在 chart A 安装/升级后，一个 helm release 就被创建/修改了。这个 release 会创建/更新所有上面的 kuberenetes 对象，按照如下的顺讯：

- A-Namespace
- B-Namespace
- A-Service
- B-Service
- B-ReplicaSet
- A-StatefulSet

这是因为当 helm 安装/升级 chart 的时候，来自 chart 和它的依赖的 chart 的 kubernetes 对象是

- 聚合到一个集合；然后
- 按类型和名称排序；然后
- 按顺序创建/跟新。

因此一个 release 被创建了，包括 chart 以及它的依赖的 chart 的所有的对象。

kubernetes 类型的安装顺序是由 InstallOrder 的枚举来给出，包含在 [kind_sorter.go](https://github.com/helm/helm/blob/484d43913f97292648c867b56768775a55e4bba6/pkg/releaseutil/kind_sorter.go) 中。

## 模板和值

helm chart 模板使用 go 模板语法编写，新增了 50 个左右的附加功能，包括[Sprig 库](https://github.com/Masterminds/sprig)，和其他一些[特殊的功能](https://helm.sh/docs/howto/charts_tips_and_tricks/)

所有的模板文件保存在 chart 的 `templates/` 目录。当 helm 渲染 chart 时，它将目录中的每一个文件传递给模板引擎。

模板的值由下面两种方式提供：

- chart 开发者会在 chart 里面提供一个叫做 `values.yaml` 的文件。这个文件包含了默认的值。
- chart 用户可以自己提供一个包含值的 yaml 文件。这个文件可以在命令行中通过 `helm install` 提供

当一个用户提供了自定义的值，这些值会覆盖 chart 中 `values.yaml` 文件中的值

### 模板文件

模板文件遵循 go 模板的标准约定（查看[text/template 文档](https://golang.org/pkg/text/template/)了解更多）。一个模板的文件的例子看起来可能是这样：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{ .Values.imageRegistry }}/postgres:{{ .Values.dockerTag }}
          imagePullPolicy: {{ .Values.pullPolicy }}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{ default "minio" .Values.storage }}
```

上面这个例子，大致基于 <https://github.com/deis/charts>，是一个 kubernetes 副本控制器模板。它可以使用下面四个模板值（通常在 `values.yaml` 文件中定义）：

- `imageRegistry`：docker 镜像仓库
- `dockerTag`：docker 镜像 tag
- `pullPolicy`：kubernetes 的镜像拉取策略
- `storage`：存储后端，默认是 "minio"

所有这些值都有模板作者定义。helm 不需要也不规定这些参数。

要了解更多工作的 chart，请查阅 CNCF [Artifact Hub](https://artifacthub.io/packages/search?kind=0)。

### 预定义值

通过 values.yaml 文件提供（或者通过 --set 参数）的 values ，在模板中通过 .Values 对象来访问。但是也有其他一些预定义你可以在模板中访问的数据。

这些值是预定义的，在每个模板中都可用，并且不能被覆盖。和所有值一样，名字是大小写铭感的。

- Release.Name：release 的名字（不是 chart）
- Release.Namespace：和 chart 相关的命名空间。
- Release.Service：进行发布的服务。
- Release.IsUpgrade：如果当前操作是更新或者回滚操作，这个值会被设置成 true
- Release.IsInstall：如果当前操作是安装操作，这个值会被设置成 true
- Chart：`Chart.yaml` 的内容。因此 chart 版本可以通过 `Chart.Version` 来获取，维护者在 `Chart.Maintainers` 中。
- Files: 类似 map 的对象，包含了 chart 中所有非特殊的文件。你不能访问模板，但是可以访问其他文件（除非它们是被 .helmignore 排除的）。文件可以使用 `{{ index .Files "file.name" }}` 或者 `{{ .Files.Get name }}` 功能来访问。你也可以使用 `{{ .Files.GetBytes }}` 以 `[]byte` 的方式来访问文件的内容。
- Capabilities：类似 map 的对象，包含了 kubernetes 的版本信息（`{{ .Capabilities.KubeVersion }}`）以及支持的 kubernetes api 版本（`{{ .Capabilities.APIVersions.Has "batch/v1" }}`）。

注意：`Chart.yaml` 中任何位置的字段都会被丢弃。它们不能在 `Chart` 对象中访问。因此，`Chart.yaml` 不能用来传递任意结构化的数据到模板中。不过，values 文件可以用来做这个事情。

### values 文件

回顾一下前面章节关于模板的介绍，`values.yaml` 文件提供了必要的值，就像这样：

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

values 文件的格式是 yaml。chart 会包含一个默认的 values.yaml 文件。helm 安装命令允许用户提供一个额外的 yaml 值文件来覆盖原来的值。

```
$ helm install --generate-name --values=myvals.yaml wordpress
```

values 通过这种方式传递的时候，它们将会和默认值合并。举个例子，`myvals.yaml` 文件内容如下：

```yaml
storage: "gcs"
```

当它在 chart 中和 `values.yaml` 合并的时候，生成的结果内容会是：

```yaml
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "gcs"
```

注意只有最后一个字段被覆盖了。

注意：chart 中的默认值文件必须命名为 `values.yaml`，但是在命令行中指定的文件可以是任意名字

注意：如果在 `helm install` 或者 `helm upgrade` 中使用了 `--set` 参数，这些值会在客户端侧简单地转换成 yaml。

注意：如果 values 文件中存在任意必要的条目，他们可以在 chart 模板中使用 `required` 方法来生命成必要的。

这里所有值都可以通过 `.Values` 对象在模板中被访问到。

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{ .Values.imageRegistry }}/postgres:{{ .Values.dockerTag }}
          imagePullPolicy: {{ .Values.pullPolicy }}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{ default "minio" .Values.storage }}
```

### 作用域，依赖，和值

values 文件可以为顶层 chart 申明值，同样也可以为任何包含在 chart 的 `charts/` 目录中子 chart 申明值。或者，换个说法，一个 values 文件可以同时给 chart 和任何它的依赖提供值。举个例子，上面示范的 wordpress chart 有 `mysql` 和 `apache` 两个依赖。values 文件可以给所有的组件提供值：

```
title: "My WordPress Site" # Sent to the WordPress template

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

更高的层级的 chart，可以访问所有下面变量。所有 wordpress chart 可以通过 `.Values.mysql.password` 访问 MySQL 的密码。但是低层级的 chart 无法访问父 chart 的任何东西，所以 mysql 无法访问 `title` 属性。同样的问题，也不能访问 `apache.port`。

values 是就是命名空间，但是命名空间是可以精简的。所以对于 wordpress chart，他可以通过 `.Values.mysql.password` 访问 mysql 的密码字段。但是对于 mysql chart，values 的作用域已经被缩小了，并且命名空间的前缀也被移除了，所以它看到的密码字段仅仅是 `.Values.password`。

### 全局值

在 2.0.0-Alpha.2 之后，helm 支持特殊的 `global` 值。考虑前面例子的修改版本：

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  port: 8080 # Passed to Apache
```

上面添加了一个 `global` 的章节，附带值 `app:MyWordPress`。这个值在所有的 chart 中都可以通过 `.Values.global.app` 来引用。

举个例子，mysql 模板可以通过 `{{ .Values.global.app }}` 来访问 `app`，`apache` chart 也一样。事实上，上面的 values 文件会被重新生成成这样：

```yaml
title: "My WordPress Site" # Sent to the WordPress template

global:
  app: MyWordPress

mysql:
  global:
    app: MyWordPress
  max_connections: 100 # Sent to MySQL
  password: "secret"

apache:
  global:
    app: MyWordPress
  port: 8080 # Passed to Apache
```

这提供了一种共享顶层变量到所有子 chart 的方式，这在一些事情比如设置类似标签这样的 `metadata` 属性的时候很有用。

如果子 chart 声明了一个全局变量，这个全局变量会向下传递（到子 chart 的子 chart），但是不会向上传递到父 chart。子 chart 是没有办法影响父 chart 的值的。

同样，父 chart 的全局变量由于子 chart 的全局变量。

### 模式文件

有时候，chart 的维护者可以想要定义一个他们值的结构。这个可以通过在 `values.scheme.json` 中定义一个模式来实现。一个模式使用 [json schema](https://json-schema.org/) 来表示。它看起来像这样：

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "properties": {
    "image": {
      "description": "Container Image",
      "properties": {
        "repo": {
          "type": "string"
        },
        "tag": {
          "type": "string"
        }
      },
      "type": "object"
    },
    "name": {
      "description": "Service name",
      "type": "string"
    },
    "port": {
      "description": "Port",
      "minimum": 0,
      "type": "integer"
    },
    "protocol": {
      "type": "string"
    }
  },
  "required": [
    "protocol",
    "port"
  ],
  "title": "Values",
  "type": "object"
}
```

这个模式可以被应用到值中去校验。当任何下面的命令被调用时，校验会发生：

- `helm install`
- `helm upgrade`
- `helm lint`
- `helm template`

一个满足这个模式要求的 `values.yaml` 文件的例子看起来像这样：

```yaml
name: frontend
protocol: https
port: 443
```

注意这个模式被应用到最终的 `.Values` 对象中，而不仅仅是 `values.yaml` 文件。这就意味着下面的 yaml 文件，在 chart 被安装时附带合适的 `--set` 选项时，也是有效的。

```yaml
name: frontend
protocol: https
```

```
helm install --set port=443
```

此外，最终的 `.Values` 对象会被所有的子 chart 模式检查。这意味子 chart 中的限制是无法在父 chart 中绕过的。反过来，如果子 chart 的 `values.yaml` 不满足要求，父 chart 必须满足这些限制才有效。

### 参考

在编写模板，值，以及模式文件的时候，这里有一些标准的参考可以帮助你。

- [Go 模板](https://godoc.org/text/template)
- [额外的模板方法](https://godoc.org/github.com/Masterminds/sprig)
- [yaml 格式](https://yaml.org/spec/)
- [JSON Schema](https://json-schema.org/)

## 自定义资源定义（CRDs）

kubernetes 提供了一种声明新对象的机制。使用 `CustomResouceDefinition` （CRDs），kubenetes 开发者可以自定义资源类型。

在 helm 3 中，CRDs 被当成一种特殊的对象。他们最先被安装，并且收到一些限制。

CRD yaml 文件应该放到 chart 的 crds 目录下面。多个 CRD（通过 yaml 的开始和结束标记分开） 可以放到同一个文件中。helm 会尝试加载 crd 目录下的所有文件到 kubernetes 中。

crd 文件不能是模板。他们必须是纯 yaml 文档。

当 helm 安装一个新的 chart，它会上传 CRDs，暂停直到 CRDs 可以被 API 服务使用，然后开启模板引擎，渲染剩下的 chart，并且上传到 kuberntes 中。因为这个顺序，CRD 的信息在 `.Capabilities` 对象在模板中是可用的，并且 helm 模板会创建新的在 CRDs 中申明的对象实例。

举个例子，如果你 chart 在 `crds` 目录中有一个 `CronTab` 的 CRD，你可以在 `templates/` 目录中创建一个 `CronTab` 的实例：

```yaml
crontabs/
  Chart.yaml
  crds/
    crontab.yaml
  templates/
    mycrontab.yaml
```

`crontab.yaml` 文件必须包含没有模板指令的 CRD：

```yaml
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
```

然后模板文件 `mycrontab.yaml` 会创建创建一个新的 `CronTab`（通常使用模板）：

```yaml
apiVersion: stable.example.com
kind: CronTab
metadata:
  name: {{ .Values.name }}
spec:
   # ...
```

helm 会保证 `CronTab` 类型已经被安装，并且在安装 `templates/` 里面的东西之前，可以通过 kubernetes API 服务使用。

### CRDs 的限制

和大多数 kuberntes 里面的对象不一样，CRDs 的安装是全局的。因此，helm 用了一个非常谨慎的方法来管理 CRD。CRD 有如下限制：

- CRD 从不重新安装。如果 helm 确定 `crds` 目录中的 CRD 已经存在（不管版本），helm 都将不会尝试安装或者更新。
- CRD 在更新和回滚的时候从不安装。helm 只会在安装操作的时候创建 CRD。
- CRD 从不删除。删除 CRD 的时候会自动删除集群中所有命名空间下的 CRD 内容。所以 helm 不会删除 CRD。

操作人员如果想要更新或者删除 CRD，建议手动操作，并且要非常小心。

## 使用 helm 来管理 chart

helm 工具有几个命令来处理 chart。

它可以为你创建一个新的 chart：

```
$ helm create mychart
Created mychart/
```

一旦你已经编辑了一个 chart，`helm` 可以为你打包成一个 chart 归档：

```
$ helm package mychart
Archived mychart-0.1.-.tgz
```

你也可以用 helm 来帮助你找到 chart 中的格式或者消息的问题

```
$ helm lint mychart
No issues found
```

## chart repository

chart repository 是一个 http 服务，存储一个或多个打包的 chart。helm 可以管理本地的 chart，而如果要共享 chart，更好地机制是 chart repository。

任何可以 http 服务器，只要可以服务 yaml 和 tar 文件，并且回复 GET 请求，都可以作为一个 chart 仓库服务器。helm 团队已经测试过了一些服务，包括开启网站模式的 Google Cloud Storage，以及开启网站模式的 S3 服务。

一个 respository 的主要特征是存在特殊的叫做 `index.yaml` 文件，它列出了所有 repository 提供的包，以及允许检索和校验这些包的元数据。

在客户端侧，respository 由 `helm repo` 命令来管理。然而，helm 没有提供工具来上传 chart 到远端的 repository 服务器。这是因为这样做会增加大量需求来实现一个服务器，从而增加建立一个 repository 的阻碍。

## chart 初始包

`helm create` 命令提供了一个可选 `--starter` 选项来让你指定一个初始包（`starter chart`）。

初始包只是一个普通的 chart，只是位于 `$XDG_DATA_HOME/helm/starters`。作为一个 chart 开发者，你可以编写专门设计用于初始化的 chart。这种 chart 的设计应该考虑如下因素：

- `Chart.yaml` 可以被生成器覆盖。
- 用户期望修改这个 chart 的内容，所以文档需要说明用户如何修改。
- 任何 `<CHARTNAME>` 出现的地方都会被替换成特定的 chart 名字，以便初始化包可以用于模板

目前唯一添加一个初始化 chart 的方式是，手动拷贝它到 `$XDG_DATA_HOME/helm/starters` 目录。在你的 chart 文档中，你可能想要需要这个过程。

## 链接

- Charts: <https://helm.sh/docs/topics/charts/>
