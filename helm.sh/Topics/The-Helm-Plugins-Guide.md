# helm 插件指南

helm 插件是一个你可以通过 helm 客户端访问的工具，但是不是内置 helm 代码库的一部分。

已有的插件可以在[相关](https://helm.sh/docs/community/related/#helm-plugins)章节中找到，或者在 [Github](https://github.com/search?q=topic%3Ahelm-plugin&type=Repositories) 上搜索。

这个指南介绍了如何使用和创建插件。

## 概览

helm 插件是和 helm 无缝集成的附加组件。他们提供了一种方式来拓展 helm 的核心功能集合，但是不需要每一个新功能都使用 go 来编写以及添加到核心工具之中。

helm 插件有如下特征：

- 他们可以从 helm 安装中添加或者移除，而不影响核心的 helm 工具
- 他们可以使用任何编程语言编写
- 他们和 helm 继承，并且出现在 `helm help` 或者其他地方

helm 插件坐落在 `$HELM_PLUGINS`。你可以使用 `helm env` 命令找到当前的值，包括你没有设置环境时的默认值。

helm 插件模型部分基于 git 插件模型。因此，你有时候可能会听到 helm 被称为 procelain 层，而插件是 plumbing。这是一种简写方式，建议 helm 提供用户体验以及顶层处理逻辑，而插件处理执行所需动作的”细节工作“。

## 安装插件

插件使用 `helm plugin install <path|url>` 命令安装。你可以传递插件的本地路径或者远端版本管理系统仓库的 url 链接。`helm plugin install` 命令克隆或者复制这个插件从给定的路径/url到 `$HELM_PLUGINS`。

```
$ helm plugin install https://github.com/adamreese/helm-env
```

如果你有一个 tar 插件包，只需要解压插件到 `$HELM_PLUGINS` 目录即可。你也可以直接通过 url 使用 `helm plugin install https://domain/path/to/plugin.tar.gz` 安装 tarball 插件

## 构建插件

很多方面，插件和 chart 类似。每一个插件都有一个顶层目录，然后有 `plugin.yaml` 文件。

```
$HELM_PLUGINS/
  |- keybase/
      |
      |- plugin.yaml
      |- keybase.sh
```

在上面的例子中，`keybase` 插件被包含在名为 `keybase` 的目录中。它有两个文件：`plugin.yaml`（必填）和一个可执行脚本 `keybase.sh`（可选）。

核心的插件是一个简单的名为 `plugin.yaml` 的 yaml 文件。这是用来添加 keybase 操作支持的插件的 yaml：

```yaml
name: "last"
version: "0.1.0"
usage: "get the last release name"
description: "get the last release name"
ignoreFlags: false
command: "$HELM_BIN --host $TILLER_HOST list --short --max 1 --date -r"
platformCommand:
  - os: linux
    arch: i386
    command: "$HELM_BIN list --short --max 1 --date -r"
  - os: linux
    arch: amd64
    command: "$HELM_BIN list --short --max 1 --date -r"
  - os: windows
    arch: amd64
    command: "$HELM_BIN list --short --max 1 --date -r"
```

`name` 是插件的名称。当 helm 执行这个插件的时候，会使用这个名字（比如 `helm NAME` 将会调用这个插件）。

`name` 应该和目录名称一致。在我们上面的例子中，这意味着名为 `keybase` 的插件应该包含在名为 `keybase` 的目录中。

`name` 的限制：

- `name` 不能和 helm 已经存在的顶层命令重复
- `name` 必须限制字符 `a-z`，`A-Z`，`0-9`，`_` 和 `-`

`version` 是插件的 SemVer2 版本。`usage` 和 `description` 都是用来生成命令的帮助信息的。

`ingoreFlags` 开关告诉 helm 不要传递参数给插件。所以如果插件被 `helm myplugin --foo` 调用，并且 `ignoreFlags: true`，`--foo` 会被默默丢弃。

最后，也是最重要的，`platformCommand` 或者 `command` 是这个插件在调用时会执行的命令。`platformCommand` 章节定义了命令在具体操作系统下的变体。下面的规则会应用于决定使用哪条命令：

- 如果存在 `platformCommand`，他会首先搜索。
- 如果 `os` 和 `arch` 同时匹配了当前的平台，搜索会停止，命令会被使用。
- 如果 os 匹配但是没有 `arch` 匹配，命令会被使用。
- 如果 `platformCommand` 匹配没有找到，默认 `command` 会被使用。
- 如果 `platformCommand` 没有匹配，并且不存在 `command`，helm 会错误退出。

环境变量会在插件执行之前插入。上面的模式表明，更好地方式是表明插件程序所在位置。

这里有一些使用插件的策略：

- 如果插件包含可执行程序，可执行程序针对 `platformCommand` 或者 `command`，应该被打包在目录中。
- `platformCommand` 或者 `command` 行中任何环境变量展开会先于执行。`$HELM_PLUGIN_DIR` 会执行插件目录。
- 命令本身不会在 shell 中执行，所以你不能一行一个 shell 脚本
- helm 注入了大量的配置到环境变量中。从环境变量中查看哪些信息是有用的。
- helm 不假设插件的语言。你可以编写任何你喜欢的。
- 命令应该实现特定的帮助文件，使用 `-h` 和 `--help`。helm 会使用 `usage` 和 `description` 给 `helm help` 以及 `helm help myplugin`，但是不会处理 `helm myplugin --help`

## 下载插件

默认情况下，helm 可以使用 HTTP/S 拉取 chart。在 helm 2.4.0 之后，插件可以有特别的能力下载任意源的 chart。

插件会在顶层的 `plugin.yaml` 文件中声明这个特别的能力。

```yaml
downloaders:
- command: "bin/mydownloader"
  protocols:
  - "myprotocol"
  - "myprotocols"
```

如果这样的插件安装了，helm 可以通过调用 `command` 使用指定的协议。添加 repository 和普通的类似 `helm repo add favorite myprotocol://example.com/`。特殊 repo 的规则和普通的也一样：helm 必须能够下载 index.yaml 文件来发现和缓存可用 chart 的列表。

定义的命令会用如下模式调用：`command certFile keyFile caFile full-URL`。ssl 证书来自仓库的定义，保存在 `$HELM_REPOSITORY_CONFIG`（比如：`$HELM_CONFIG_HOME/repositories.yaml`）中。下载插件应该将原始内容输出到标准输出，并在标准错误中上报错误。

下载命令也支持子命令参数，允许你在 `plugin.yaml` 中指定，比如 `bin/mydownloader subcommand -d`。如果你想要使用在主插件命令和下载命令中使用同一个可执行程序，这会很有用，但是每个有不同的子命令。

## 环境变量

当 helm 执行插件的时候，它会传递外面的环境变量到插件中，并且会注入额外的环境变量。

变量比如 `KUBECONFIG` 是插件的配置，如果他们被设置在外层的环境中。

下面这些变量是保证会设置的：

- `HELM_PLUGINS`：插件目录的路径
- `HELM_PLUGIN_NAME`：插件名，被 helm 调用。所以 `helm myplug` 会有一个短的名字 `myplug`
- `HELM_PLUGIN_DIR`：包含插件的目录
- `HELM_BIN`：helm 命令的路径（当成用户执行）
- `HELM_DEBUG`：表明是否调试参数被设置
- `HELM_REGISTRY_CONFIG`：registry 配置（如果使用了）的地址。注意 helm 的 registry 用法是一个试验性的功能
- `HELM_REPOSITORY_CACHE`：repository 缓存文件的路径
- `HELM_REPOSITORY_CONFIG`：repository 配置文件的路径
- `HELM_NAMESPACE`：给 helm 命令的命名空间（通常使用 `-n` 参数）
- `HELM_KUBECONTEXT`：给 helm 命令的 kubernetes 配置内容的名字

额外的，如果 kubernetes 配置文件被显式指定，它也会被设置在 `KUBECONFIG` 变量

## 参数解析的注意事项

在执行插件的时候，helm 会解析我们使用的全局参数。下面这些参数不会传到插件中。

- `--debug`：如果指定了这个，`$HELM_DEBUG` 会设置成 1
- `--registry-config`：转换成 `$HELM_REGISTRY_CONFIG`
- `--repository-cache`：转换成 `$HELM_REPOSITORY_CACHE`
- `--repository-config`：转换成 `$HELM_REPOSITORY_CONFIG`
- `--namespace` 和 `-n`：转换成 `$HELM_NAMESPACE`
- `--kubeconfig`：转换成 `$KUBECONFIG`

插件应该在 `-h` 和 `--help` 的时候展示帮助文本并退出。其他情况下，插件可以恰当地使用参数。

## 提供 shell 自动补全

在 helm 3.2 之后，插件可选地提供了 shell 自动补全支持作为 helm 自动补全系统的一部分。

### 静态自动补全

如果一个插件提供了它自己的参数或者子命令，它可以通过在插件根目录放置 `completion.yaml` 文件来通知 helm。`completion.yaml` 文件的格式如下：

```
name: <pluginName>
flags:
- <flag 1>
- <flag 2>
validArgs:
- <arg value 1>
- <arg value 2>
commands:
  name: <commandName>
  flags:
  - <flag 1>
  - <flag 2>
  validArgs:
  - <arg value 1>
  - <arg value 2>
  commands:
     <and so on, recursively>
```

注意：

1. 所有部分都是可选的，但是应该适时提供
2. 参数不应该包含 `-` 或者 `--` 前缀
3. 短参数和长参数都应该指定。短参数不需要和它相应的长参数关联，但是两种参数都需要列出来
4. 参数不需要任何顺序，但是需要在文件子命令的层次结构中列出正确的位置
5. helm 存在的全局参数已经被 helm 的自动补全机制处理了，因此参数不需要指定如下参数 `--debug`，`--namespace` 或者 `-n`，`--kube-context` 以及 `--kubeconfig` 或者其他任何全局参数
6. `validArgs` 列表提供了子命令接下来的第一个参数可能的补全的静态列表。不太可能总是提供这样一个列表（查阅下面的[动态补全](./The-Helm-Plugins-Guide.md#动态补全)部分），这种情况下 `validArgs` 部分可以省略

`completion.yaml` 文件整个都是可选的。如果没有提供，helm 不会为这个插件提供自动补全（除非这个插件支持[动态补全](./The-Helm-Plugins-Guide.md#动态补全)）。同样，添加一个 `completion.yaml` 文件是向后兼容的，并且不会影响插件在使用老版本的 helm 时候的行为。

作为一个例子，对于 `fullstatus plugin` 没有子命令，但是接收和 `helm status` 同样参数的命令，`completion.yaml` 文件是这样：

```yaml
name: fullstatus
flags:
- o
- output
- revision
```

一个更加复杂的例子 `2to3 plugin`，有一个 `completion.yaml` 文件：

```yaml
name: 2to3
commands:
- name: cleanup
  flags:
  - config-cleanup
  - dry-run
  - l
  - label
  - release-cleanup
  - s
  - release-storage
  - tiller-cleanup
  - t
  - tiller-ns
  - tiller-out-cluster
- name: convert
  flags:
  - delete-v2-releases
  - dry-run
  - l
  - label
  - s
  - release-storage
  - release-versions-max
  - t
  - tiller-ns
  - tiller-out-cluster
- name: move
  commands:
  - name: config
    flags:
    - dry-run
```

## 动态补全

同样从 helm 3.2 开始，插件可以提供他们自己的动态补全。动态补全是无法提前定义的参数或者选项值的补全。比如，补全目前集群中可用的 helm release 的名字。

对于支持动态补全的插件，它必须在根目录提供名为 `plugin.complete` 的可执行文件。当 helm 补全脚本需要为插件动态补全的时候，它会调用 `plugin.complete` 文件，传递需要补全的命令行。`plugin.complete` 可执行程序会需要有逻辑去决定那个补全更合适并且输出他们到标准输出来被 helm 补全脚本消费。

`plugin.complete` 文件是可选的。如果没有提供，helm 不会为插件提供动态补全。同样，添加 `plugin.complete` 文件是向后兼容的，并且不影响使用老版本 helm 时插件的行为。

`plugin.complete` 脚本的输出需要是以换行分割的列表，比如：

```
rel1
rel2
rel3
```

当 `plugin.complete` 被调用了，插件的环境被设置成和插件住程序调用时一样。因此，`$HELM_NAMESPACE`，`$HELM_KUBECONTEXT` 变量，以及所有插件变量都已经设置好了，它们相应的全局参数被移除了。

`plugin.complete` 文件可以是任何可执行文件格式，可以说是 shell 脚本，或者是 go 程序，或者是任何其他类型的 helm 可以执行的程序。`plugin.complete` 文件必须有用户可执行权限。`plugin.complete` 文件必须以成功码（0）退出。

在很多场景中，动态补全需要从 kubernetes 集群中获取信息。比如，`helm fullstatus` 插件需要 release 名字作为输入。在 `fullstatus` 插件中，对于它的 `plugin.complete` 脚本提供了当前 release 名字的补全，他可以运行 `helm list -q` 然后输出结果。

如果有需要使用同一个可执行程序来执行插件和补全，`plugin.complete` 脚本可以做成使用一些特殊的参数或者选项来调用主插件可执行程序，它会知道运行补全。在我们的例子中，`plugin.complete` 可能是实现是这样：

```shell
#!/usr/bin/env sh

# "$@" is the entire command-line that requires completion.
# It is important to double-quote the "$@" variable to preserve a possibly empty last parameter.
$HELM_PLUGIN_DIR/status.sh --complete "$@"
```

`fullstatus` 插件的真正脚本（`status.sh`）必须查找 `--complete` 选项，如果找到，打印合适的补全。

### 提示和技巧

1. shell 会自动过滤没有匹配用户输入的补全。插件因此可以返回所有相关的补全，而不需要移除没有匹配用户的那些。比如如果命令是 `helm fullstatus ngin<TAB>`，`plugin.complete` 脚本可以打印所有 release 名字（在 `default` 命名空间），不仅仅以 `ngin` 开头的，shell 只会保留以 `ngin` 开头的。
2. 为了简化动态补全支持，特别是如果你负载的插件，你可以让 `plugin.complete` 脚本调用你的主插件以及请求补全选择。查看上面例子中[动态补全]的(./The-Helm-Plugins-Guide.md#动态补全)部分。
3. 要调试动态补全以及 `plugin.complete` 文件，你可以运行如下命令来看补全结果：
    - `helm __complete <pluginName> <arguments to complete>` 比如：
    - `helm __complete fullstatus --output js<ENTER>`
    - `helm __complete fullstatus -o json ""<ENTER>`

## 链接

- The Helm Plugins Guide：<https://helm.sh/docs/topics/plugins/>
