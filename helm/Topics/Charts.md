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

注意 `appVersion` 字段和 `version` 字段没有关系。它是一种指定应用版本的方式。举例来说，`drupal` chart 有一个 app 版本 `appVersion: "8.2.1"`，表明 chart 中（默认情况下）的 drupal 的版本是 8.2.1。这个字段是一个描述性的信息，不影响 chart 版本的计算。

### dependencies

### type
