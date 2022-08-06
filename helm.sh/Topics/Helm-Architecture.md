# Helm 架构

这个文档从较高层次介绍 helm 的架构

##  helm 的目标

helm 是一个管理名为 chart 的 kubenetes 包的工具。helm 可以做如下事情：

- 从头开始创建新的 chart
- 打包 chart 成归档文件（tgz）
- 和存储 chart 的 chart repository 交互
- 从已经存在的 kubernetes 集群中安装和卸载 chart
- 管理已经通过 helm 安装的 release 的生命周期

对于 helm，有三个重要的概念：

1. chart 是一组用于创建 kubernetes 应用实例的必要信息。
2. config 包含了配置信息，用于合并进入打包好的 chart中，以创建发布对象。
3. release 是一个运行的 chart 实例，与特定的配置相结合。

## 组件

helm 是一个可执行文件，实现了两个不同的部分：

**helm 客户端**是一个终端用户命令行客户端。这个客户端负责下面这些：

- 本地 chart 开发
- 管理 repository
- 管理 release
- 和 helm 库交互
    - 发送 chart 来安装
    - 升级或者卸载已经存在的 release

**helm 库**提供了执行所有 helm 操作的逻辑。它和 kubernetes api 服务器交互，并提供如下能力：

- 结合了 chart 和配置来构建了一个 release
- 安装 chart 到 kubernetes，并提供了后续的 release 对象
- 和 kubernetes 交互来升级和卸载 chart

独立的 helm 库封装了 helm 的逻辑，所以它可以被其他的客户端使用。

## 实现

helm 客户端和库使用 go 语言编写。

库使用 kubernetes 客户端库来和 kubernetes 交互。目前，库使用 `REST+JSON`。它在 kubernetes 中保存了秘钥。它不需要自己的数据库。

配置文件尽可能使用 yaml 编写。

## 链接

- Helm Architecture: <https://helm.sh/docs/topics/architecture/>
