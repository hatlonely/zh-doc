# chart repository 指南

这个章节介绍如何创建和使用 helm chart repository。在高级别，chart repository 是一个存放打包和分享 chart 包的地方。

分布式社区 helm chart repository 在 [Artifact Hub](https://helm.sh/docs/topics/chart_repository/) ，欢迎大家参与。但是 helm 也支持创建和运行你自己的 chart repository。这个指南会介绍要如何做到。

## 先决条件

- 浏览[快速开始](../Introduction/Quickstart-Guide.md)指南
- 阅读[chart](Charts.md)文档

## 创建一个 cahrt repository

chart repository 是一个存储了 index.yaml 文件和可选的一些 chart 包的 http 服务。当你准备好了共享你的 chart，更好地方式是上传他们到一个 chart repository。

从 helm 2.2.0 之后，支持了客户端的 ssl 授权给 repository。其他的授权协议也可通过插件的方式使用。

因为 chart repository 可以是任何提供 yaml 和 tar 文件并响应 http 请求的 http 服务，如果你想要托管自己的 chart repository 的时候有很多选择。比如你可以使用谷歌存储服务 GCS，亚马逊 S3 服务，Github Pages，甚至创建自己的 web 服务。

### chart repository 结构

chart repository 由 chart 包和名为 `index.yaml` 的特殊文件组成，`index.yaml` 包含了所有 repository 中 chart 的索引。通常，`index.yaml` 描述的 chart 也托管在同样的服务商，[出处文件](Helm-Provenance-and-Integrity.md)也是。

举个例子，`https://example.com/charts` repository 的布局看起来像这样：

```
charts/
  |
  |- index.yaml
  |
  |- alpine-0.1.2.tgz
  |
  |- alpine-0.1.2.tgz.prov
```

在这个例子中，索引文件会包含一个 chart 的信息，Alpine chart，以及提供这个 chart 的下载链接 `https://example.com/charts/alpine-0.1.2.tgz`。

chart 包不需要和和 index.yaml 在同一个服务器。然而，这样做通常是最简单的。

### 索引文件

索引文件是一个 yaml 格式文件叫做 `index.yaml`。它包含了包的元数据，包括 chart 的 `Chart.yaml` 文件的内容。一个有效的 chart repository 必须包含一个索引文件。这个索引文件包含每个 chart repository 中 chart 的信息。`helm repo index` 命令会基于给定本地目录和包含的 chart 包生成一个索引文件。

这是一个索引文件的例子：

```yaml
apiVersion: v1
entries:
  alpine:
    - created: 2016-10-06T16:23:20.499814565-06:00
      description: Deploy a basic Alpine Linux pod
      digest: 99c76e403d752c84ead610644d4b1c2f2b453a74b921f422b9dcb8a7c8b559cd
      home: https://helm.sh/helm
      name: alpine
      sources:
      - https://github.com/helm/helm
      urls:
      - https://technosophos.github.io/tscharts/alpine-0.2.0.tgz
      version: 0.2.0
    - created: 2016-10-06T16:23:20.499543808-06:00
      description: Deploy a basic Alpine Linux pod
      digest: 515c58e5f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cd78727
      home: https://helm.sh/helm
      name: alpine
      sources:
      - https://github.com/helm/helm
      urls:
      - https://technosophos.github.io/tscharts/alpine-0.1.0.tgz
      version: 0.1.0
  nginx:
    - created: 2016-10-06T16:23:20.499543808-06:00
      description: Create a basic nginx HTTP server
      digest: aaff4545f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cdffffff
      home: https://helm.sh/helm
      name: nginx
      sources:
      - https://github.com/helm/charts
      urls:
      - https://technosophos.github.io/tscharts/nginx-1.1.0.tgz
      version: 1.1.0
generated: 2016-10-06T16:23:20.499029981-06:00
```

## 托管 chart repository

这个部分介绍几种提供 chart repository 服务的方法。

### Google Cloud Storage

第一步是创建你的 GCS 桶，我们叫做 `fantastic-charts`。

![](https://helm.sh/img/create-a-bucket.png)

然后，修改桶的权限为公开。

![](https://helm.sh/img/edit-permissions.png)

插入这一行来公开你的桶

![](https://helm.sh/img/make-bucket-public.png)

恭喜，现在你有一个空的 GCS 桶，可以提供 chart repository 服务了！

你可以使用 GCS 命令行工具来上传你的 chart repository，或者使用 GCS 的 web 界面。一个公开的 GCS 桶可以通过简单的 https 请求地址 `https://bucket-name.storage.googleapis.com/` 访问到。

### Cloudsmith

你也可以使用 Cloudsmith 来创建你的 chart repository。阅读更多关于使用 Cloudsmith 搭建 chart repository 的信息，[这里](https://help.cloudsmith.io/docs/helm-chart-repository)

### JFrog Artifactory

同样的，你也可以使用 JFrog Artifactory 来创建你的 chart repository。于都更多关于使用 JFrog Artifactory 搭建 chart repository 的信息，[这里](https://www.jfrog.com/confluence/display/RTF/Helm+Chart+Repositories)

### Github Pages 例子

同样的方法，使用 GitHub Pages 也可以创建 chart repository。

Github 允许你通过两种不同的方式来提供静态网页服务：

- 通过配置一个项目来服务它的 `docs/` 目录下的内容
- 通过配置一个项目来服务特定的分支

我们使用第二种方式，尽管第一种更加简单。

第一步创建你的 `gh-pages` 分支，你可以在本地执行：

```
$ git checkout -b gh-pages
```

或者通过 web 浏览器，在你的 GitHub 仓库中点击 branch 按钮：

![](https://helm.sh/img/create-a-gh-page-button.png)

接下来，你会想要确保你的 `gh-pages` 分支被设置成了 GitHub Pages，点击你的仓库设置，滚动到 GitHub Pages 章节，并按如下设置：

![](https://helm.sh/img/set-a-gh-page.png)

默认抢矿下，源通常设置成 gh-pages 分支。如果这不是默认设置，选择它。

如果你想要的话，这里可以使用自定义的域名。

检查 `Enforce HTTPS` 被勾选了，这样的话 HTTPS 会在 chart 服务的时候使用。

这样的配置下，你可以使用默认的分支来存储你的 chart 代码，gh-pages 分支作为 chart repository，比如：`https://USERNAME.github.io/REPONAME`。TS Charts repository 样例可以通过 `https://technosophos.github.io/tscharts/.` 访问。

如果你已经决定使用 GitHub Pages 来托管你的 chart repository，请查看[chart 发布操作](../How-To/Chart-Releaser-Action-To-Automate-Github-Page-Charts.md)。

### 普通 Web 服务器

要配置一个普通的 web 服务器提供 helm chart 服务，你只需要做下面的事情：

- 把索引和 chart 放到服务器可以服务的目录
- 确保 `index.yaml` 文件可以被无鉴权访问
- 保证 `yaml` 文件会在服务的时候被设置成正确的内容类型（`text/yaml` 或者 `text/x-yaml`）

举个例子，如果你想要在 `$WEBROOT/charts` 提供 chart 服务，确保这里你的服务根目录有一个 `charts/` 目录，并把索引和 chart 放到这个目录里面。

### ChartMuseum 仓库服务

ChartMuseum 是一个开源的 helm chart repository 服务，使用 golang 编写，支持云服务后端，包括 [Google Cloud Storage](https://cloud.google.com/storage/)，[Amazon S3](https://aws.amazon.com/s3/)，[Microsoft Azure Blob Storage](https://azure.microsoft.com/en-us/services/storage/blobs/)，[Alibaba Cloud OSS Storage](https://www.alibabacloud.com/product/oss)，[Openstack Object Storage](https://developer.openstack.org/api-ref/object-store/)，[Oracle Cloud Infrastructure ObjectStorage](https://cloud.oracle.com/storage)，[Baidu Cloud BOS Storage](https://cloud.baidu.com/product/bos.html)，[Tencent Cloud Obejct Storage](https://intl.cloud.tencent.com/product/cos)，[DigitalOcean Spaces](https://www.digitalocean.com/products/spaces/)，[Minio](https://min.io/)，以及 [etcd](https://etcd.io/)。

你也可以使用 [ChartMuseum] 服务器来从本地文件系统托管 chart repository。

### GitLab Package Registry

在 GitLab 中你可以在项目的 Package Registry 中发布 helm chart，阅读更多关于通过 GitLab Package Registry 创建 helm chart repository 的信息，[这里](https://docs.gitlab.com/ee/user/packages/helm_repository/)。

## 管理 chart repository

现在你有一个 chart repository，最后这部分介绍在 repository 中如何维护 chart。

### 在 chart repository 中存储 chart

现在你有一个 chart repository，让我们上传一个索引文件到仓库。chart repository 中的 cahrt 必须是打包好的（`helm pacakge chart-name/`）并且版本正确（符合 [SemVer2](https://semver.org/) 规范）。

接下来的步骤构成了一个工作流的例子，但是欢迎您使用任何你喜欢的工作流来保存和更新 chart repository 中的 chart。

一旦已经有一个打包好的 chart，创建一个新的目录，然后移动这个打包好的 chart 到那个目录。

```
$ helm package docs/examples/alpine/
$ mkdir fantastic-charts
$ mv alpine-0.1.0.tgz fantastic-charts/
$ helm repo index fantastic-charts --url https://fantastic-charts.storage.googleapis.com
```

最后一条命令会用你刚刚创建爱你的本地目录和你远端的 chart repository 链接，构建一个 `index.yaml` 文件放到指定的目录路径。

现在你可以使用同步工具或者手动上传这个 chart 和索引文件到你的 chart repository。如果你使用 GCS，使用 gsutil 客户端检查这个工作流例子。对于 GitHub，你仅仅需要把 chart 放到合适的目标分支。

### 添加一个新的 chart 到一个存在的 repository

每次你想要添加一个新的 chart 到你的 repository，你必须要重新生成索引。`helm repo index` 命令会从头开始完全重建 `index.yaml` 文件，只包括本地能找到的 chart。

然后你可以使用 `--merge` 参数来增量增加新的 chart 到一个已经存在的 `index.yaml` 文件中（在使用类似 GCS 的远端 repository 时非常有用）。运行 `helm repo index --help` 来了解更多。

去报你同时上传了修正过的 `index.yaml` 文件和 chart。如果你生成了一个出处文件，也上传它。

### 和其他人分享你的 chart

当你准备好分享你的 chart，只需要让别人知道你的 repository 链接地址即可。

他们会通过 `helm repo add [NAME] [URL]` 命令来添加 repository 到他们的 helm 客户端，使用任何他们喜欢的名字来引用这个 repository。

```
$ helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com
$ helm repo list
fantastic-charts    https://fantastic-charts.storage.googleapis.com
```

如果 chart 后端使用了 http 基础授权，你也可以提供用户名和密码：

```
$ helm repo add fantastic-charts https://fantastic-charts.storage.googleapis.com --username my-username --password my-password
$ helm repo list
fantastic-charts    https://fantastic-charts.storage.googleapis.com
```

注意：如果没有包含一个有效的 index.yaml 文件，repository 不会被添加。

注意：如果你的 helm chart repository 使用自己签名的证书，你可以使用 `helm repo add --insecure-skip-tls-verify ...` 来跳过 CA 认证。

在这之后，你的用户可以搜索你的 chart。在你更新 repository 之后，他们可以使用 `helm repo update` 命令来获取最新的 chart 信息。

在内部，`helm repo add` 和 `helm repo update` 命令会获取 `index.yaml` 文件，并将他们保存在 `$XDG_CACHE_HOME/helm/repository/cache/` 目录中。这是 `helm search` 功能查找 chart 信息的地方。

## 链接

- The Chart Repository Guide: <https://helm.sh/docs/topics/chart_repository/>
