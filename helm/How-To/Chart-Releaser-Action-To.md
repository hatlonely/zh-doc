# Chart Releaser Action 自动化发布 github pages chart

本指南描述了如何使用 [Chart Releaser Action](https://github.com/marketplace/actions/helm-chart-releaser) 来通过 github pages 自动化发布 chart。chart 发布动作是一个 github action 工作流，用来将 github 项目转换成自托管的 helm chart 仓库，使用了 [helm/chart-releaser](https://github.com/helm/chart-releaser) 客户端工具。

## repository 变化

在你的组织下面创建一个 git 仓库。你可以命名为 helm-charts，尽管其他名字也是可以的。所有 chart 的源码可以放到 main 分支上。chart 应该放到顶层目录树下的 `/charts` 目录

另一个叫做 `gh-pages` 的分支用来发布 chart。这个分支的变化将由这里描述的 chart 发布操作自动创建。然而，你可以创建 `gh-branch` 分支并添加 `README.md` 文件，这些将被浏览这个页面的用户看到。

你可以在 `README.md` 中给 chart 的安装添加一些指令，就像下面这样（替换 `<alias>`，`<orgname>` 以及 `<chart-name>`）：

```
## Usage

[Helm](https://helm.sh) must be installed to use the charts.  Please refer to
Helm's [documentation](https://helm.sh/docs) to get started.

Once Helm has been set up correctly, add the repo as follows:

  helm repo add <alias> https://<orgname>.github.io/helm-charts

If you had already added this repo earlier, run `helm repo update` to retrieve
the latest versions of the packages.  You can then run `helm search repo
<alias>` to see the charts.

To install the <chart-name> chart:

    helm install my-<chart-name> <alias>/<chart-name>

To uninstall the chart:

    helm delete my-<chart-name>
```

这个 chart 会被发布到一个网站，url 就像下面这样：

```
https://<orgname>.github.io/helm-charts
```

## Github Actions 工作流

在 main 分支创建一个 Github Actions 工作流文件 `.github/workflows/release.yml`

```yaml
name: Release Charts

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"

      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.1.0
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
```

上面这个配置使用 [@helm/chart-releaser-action](https://github.com/helm/chart-releaser-action) 来讲你的 github 项目转换成自托管的 helm chart 仓库。在每次 main 分支的推送中，他做了如下事情，检查 project 中的每一个 chart，并且一旦有一个新的 chart 版本，创建一个响应的 github 发布版本，添加 helm chart 组件到这个版本中，并用这个版本的元数据创建或者更新 index.yaml 文件，然后托管到 github pages 上。

上面例子中 chart 发布动作版本号是 v1.1.0。你可以将它改成[最新的版本](https://github.com/helm/chart-releaser-action/releases)。

注意：Chart Releaser Action 几乎总是同时和 [Helm Testing Action](https://github.com/marketplace/actions/helm-chart-testing) 以及 [Kind Action](https://github.com/marketplace/actions/kind-cluster) 使用

## 链接

- Chart Releaser Action to Automate GitHub Page Charts: <https://helm.sh/docs/howto/chart_releaser_action/>
