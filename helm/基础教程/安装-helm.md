# 安装 Helm

本指南介绍如何安装 helm 客户端。helm 可以从源码或者构建好的二进制版本中安装。

## 从 Helm 项目中

Helm 项目提供了两种方式来获取和安装 Helm。他们是官方提供的获取 Helm 版本的方法。除此之外，Helm 社区提供了通过不同包管理工具安装 Helm 的方法。通过这些方法的安装可以在官方方法下找到。

### 从二进制版本

每一个 Helm 版本为各个操作系统提供了二进制版本。这些二进制版本可以手动下载和安装。

1. 下载你所需的版本
2. 解压（`tar -zxvf helm-v3.0.0-linux-amd64.tar.gz`）
3. 在解压目录中找到 `helm` 二进制，然后移动到所需的目录（`mv linux-amd64/helm /usr/local/bin/helm`）

现在，你应该可以运行客户端并[添加仓库](快速开始.md#初始化一个 Helm Chart 仓库): `helm help`

注意: 只有在 CircleCi 构建和发布期间，才会对 Linux AMD64 执行 Helm 的自动化测试。其他操作系统的测试由要求相关的操作系统的社区负责。

## 从脚本

Helm 现在有一个安装脚本可以自动获取最新的 Helm 版本并且在本地安装

您可以获取这个脚本，并在本地执行。它有非常好的文档，因此您可以在运行它之前通过阅读它了解它到底做了什么。

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

是的，如果你想更方便一点，可以 `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`

## 通过包管理工具

Helm 社区提供了通过操作系统包管理工具安装 Helm 的能力。这些方式不受 Helm 项目支持，并且也不被视为受信任的第三方。

### 通过 Homebrew（macOS）

Helm 社区成员为 Homebrew 贡献了一个 Helm 的构建。这个构建通常是最新的

```
brew install helm
```

（注意：在另一个项目里面还有一个 emacs-helm 版本）

### 通过 Chocolatey（Windows）

Helm 社区成员为 Chocolatey 贡献了一个 Helm 包。这个包通常是最新的。

```
choco install kubernetes-helm
```

### 通过 Scoop（Windows）

Helm 社区成员为 Scoop 贡献了一个 Helm 包。这个包通常是最新的。

```
scoop install helm
```

### 通过 Apt（Debian/Ubuntu）

Helm 社区成员为 Apt 贡献了一个 Helm 包。这个包通常是最新的。

```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### 通过 Snap

Snapcrafters 社区维护了 Snap 版本的 Helm 包

```
sudo snap install helm --classic
```

### 通过 pkg（FreeBSD）

FreeBSD 社区成员为 FreeBSD Ports 集合贡献了一个 Helm 包。这个包通常是最新的

```
pkg install helm
```

### 开发版本

除了正式版本，你也可以下载和安装开发版本的 Helm 快照。

### 金丝雀版本

金丝雀版本是从最新的 main 分支构建出来的 Helm 软件版本。这些不是官方发布的，可能会不太稳定。然而，它们提供了新功能的测试机会。

金丝雀版本的 Helm 二进制保存在 [get.helm.sh](https://get.helm.sh/)。这里有一些通用构建的链接

- [Linux AMD64](https://get.helm.sh/helm-canary-linux-amd64.tar.gz)
- [macOS AMD64](https://get.helm.sh/helm-canary-darwin-amd64.tar.gz)
- [Experimental Windows AMD64](https://get.helm.sh/helm-canary-windows-amd64.zip)

### 通过源码（Linux，macOS）

从源码构建 Helm 需要做更多的工作，但是如果你想测试最新的 Helm 版本，这是最好的方法。

你必须有一个可以工作的 Go 环境

```
$ git clone https://github.com/helm/helm.git
$ cd helm
$ make
```

如果有必要的话，它会拉取依赖并且缓存它们，然后校验配置。然后它会编译 `helm` 并把它放到 `bin/helm`

## 结论

大多数情况下，安装和获取一个构建好的 helm 二进制一样简单。这个文档涵盖了那些想要用 Helm 做更复杂事情的人的其他案例

一旦你成功地安装了 Helm 客户端，你可以开始使用 Helm 来管理 charts 以及添加仓库

## 链接

- Installing Helm: <https://helm.sh/docs/intro/install/>
