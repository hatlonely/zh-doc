# Registries

Helm 3 支持 OCI 用于包分发。chart 包可以通过基于 OCI 的 registry 存储和共享。

## 启用 OCI 支持

helm v3.8.0 之前，OCI 支持被认为是试验性的，需要开启才能使用。v3.8.0 之后，默认开启了。

为了在 v3.8.0 之前开启 OCI 试验性的支持，请设置 `HELM_EXPERIMENTAL_OCI` 环境变量：

```
export HELM_EXPERIMENTAL_OCI=1
```

## 运行一个 registry

开启一个 registry 来测试是很容易的。只要你安装了 docker，运行下面这个命令即可：

```
docker run -dp 5000:5000 --restart=always --name registry registry
```

这个会在 `localhost:5000` 启动一个 registry 服务。

使用 `docker logs -f registry` 来查看日志，以及 `docker rm -r registry` 来停止。

如果你希望持久化保存，你可以添加 `-v $(pwd)/registry:/var/lib/registry` 到上面的命令行中。

更多的配置选项，请查阅[docker 文档](https://docs.docker.com/registry/deploying/)

注意：在 macOS 中，端口 5000 已经被”AirPlay Receiver“占用了。你可以选择另一个本地端口（比如：`-p 5001:5000`），或者在系统配置->共享中禁用它。

## 认证

如果你想要开启 registry 的认证，按如下步骤：

首先创建包含用户名和密码组合的文件 `auth.htpasswd`：

```
htpasswd -cB -b auth.htpasswd myuser mypass
```

然后，开启这个服务，挂载文件并设置 `REGISTRY_AUTH` 环境变量：

```
docker run -dp 5000:5000 --restart=always --name registry \
  -v $(pwd)/auth.htpasswd:/etc/docker/registry/auth.htpasswd \
  -e REGISTRY_AUTH="{htpasswd: {realm: localhost, path: /etc/docker/registry/auth.htpasswd}}" \
  registry
```

## 用于 registry 的命令

### registry 命令

`login`

登录一个 registry（手动输入密码）

```
$ helm registry login -u myuser localhost:5000
Password:
Login succeeded
```

`logout`

从 registry 中登出

```
$ helm registry logout localhost:5000
Logout succeeded
```

### push 子命令

上传 chart 到 registry

```
$ helm push mychart-0.1.0.tgz oci://localhost:5000/helm-charts
Pushed: localhost:5000/helm-charts/mychart:0.1.0
Digest: sha256:ec5f08ee7be8b557cd1fc5ae1a0ac985e8538da7c93f51a51eff4b277509a723
```

#### push 子命令其他注意事项

`push` 子命令只能用于 `helm package` 命令提前创建的 `.tgz` 文件。

当使用 `helm push` 来上传 chart 到 OCI registry 时，引用必须以 `oci://` 开头，并且必须包含基础名字或者 tag。

这个 registry 引用的基础名字从 chart 名字推断而来，tag 是从 chart 的语义化版本推断而来。目前这是一个严格的要求（[更多信息](https://helm.sh/docs/topics/registries/#deprecated-features-and-strict-naming-policies)）

特定的 registry 需要提前创建 repository 和命名空间（如果有指定）。否则，在 `helm push` 操作的时候会产生错误。

如果你创建了一个[出处文件](./Helm-Provenance-and-Integrity.md)（.prov），并且和 `.tgz` 文件在一起，它会自动在 `push` 操作中自动上传到 registry。这个结果在[helm 清单文件](./Registries.md#helm-清单文件)中是一个额外的层级。

[helm push 插件](https://github.com/chartmuseum/helm-push)（上传 chart 到 ChartMuseum）的用户可能碰到过这个问题，插件和新的内置的 `push` 存在冲突。在 v0.10.0 版本之后，插件已经重命名为 `cm-push`。

### 其他子命令

`oci://` 协议也支持很多其他的子命令，这些完整的列表：

- helm pull
- helm show
- helm template
- helm install
- helm upgrade

chart 在 registry 中的基础名字引用包含在涉及下载的任何动作之中（与省略的 `helm push` 相比）。

这里有一些使用子命令列出基于 OCI 的 chart 的例子：

```
$ helm pull oci://localhost:5000/helm-charts/mychart --version 0.1.0
Pulled: localhost:5000/helm-charts/mychart:0.1.0
Digest: sha256:0be7ec9fb7b962b46d81e4bb74fdcdb7089d965d3baca9f85d64948b05b402ff

$ helm show all oci://localhost:5000/helm-charts/mychart --version 0.1.0
apiVersion: v2
appVersion: 1.16.0
description: A Helm chart for Kubernetes
name: mychart
...

$ helm template myrelease oci://localhost:5000/helm-charts/mychart --version 0.1.0
---
# Source: mychart/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
...

$ helm install myrelease oci://localhost:5000/helm-charts/mychart --version 0.1.0
NAME: myrelease
LAST DEPLOYED: Wed Oct 27 15:11:40 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
...

$ helm upgrade myrelease oci://localhost:5000/helm-charts/mychart --version 0.2.0
Release "myrelease" has been upgraded. Happy Helming!
NAME: myrelease
LAST DEPLOYED: Wed Oct 27 15:12:05 2021
NAMESPACE: default
STATUS: deployed
REVISION: 2
NOTES:
```

## 指定依赖

使用 `dependency update` 子命令，chart 的依赖会从 registry 拉取。

`Chart.yaml` 中指定的 repository 入口，作为 registry 引用没有基础名字

```yaml
dependencies:
  - name: mychart
    version: "2.7.0"
    repository: "oci://localhost:5000/myrepo"
```

这会在执行 `dependency update` 的时候，获取 `oci://localhost:5000/myrepo/mychart:2.7.0`。

## helm 清单文件

registry 中的 helm chart 清单文件代表例子（注意 `mediaType` 字段）：

```json
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.cncf.helm.config.v1+json",
    "digest": "sha256:8ec7c0f2f6860037c19b54c3cfbab48d9b4b21b485a93d87b64690fdb68c2111",
    "size": 117
  },
  "layers": [
    {
      "mediaType": "application/vnd.cncf.helm.chart.content.v1.tar+gzip",
      "digest": "sha256:1b251d38cfe948dfc0a5745b7af5ca574ecb61e52aed10b19039db39af6e1617",
      "size": 2487
    }
  ]
}
```

接下来的例子包含了一个[出处文件](./Helm-Provenance-and-Integrity.md)（注意额外的层级）

```json
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.cncf.helm.config.v1+json",
    "digest": "sha256:8ec7c0f2f6860037c19b54c3cfbab48d9b4b21b485a93d87b64690fdb68c2111",
    "size": 117
  },
  "layers": [
    {
      "mediaType": "application/vnd.cncf.helm.chart.content.v1.tar+gzip",
      "digest": "sha256:1b251d38cfe948dfc0a5745b7af5ca574ecb61e52aed10b19039db39af6e1617",
      "size": 2487
    },
    {
      "mediaType": "application/vnd.cncf.helm.chart.provenance.v1.prov",
      "digest": "sha256:3e207b409db364b595ba862cdc12be96dcdad8e36c59a03b7b3b61c946a5741a",
      "size": 643
    }
  ]
}
```

## 从 chart repository 中迁移

从经典的 chart repository（基于 index.yaml 的 repository）和使用 `helm pull` 一样简单，然后使用 `helm push` 来上传 `.tgz` 文件的结果到一个 registry。

## 废弃的特性和严格的命名策略

- 在 helm 3.7.0 之前，helm OCI 支持略有不同。作为 [HIP 6](https://github.com/helm/community/blob/main/hips/hip-0006.md) 的结果，为了简化和稳定这个特性，实现了几个变更：

- `helm chart` 子命令被移除了
- chart 缓存被移除了（没有 `helm chart list` 等）
- OCI registry 引用现在总是以 `oci://` 开头
- registry 引用的基础名字必须和 chart 的名字一致
- registry 引用的 tag 必须和 chart 的语义化版本一致（比如：没有 `latest` tag 了）
- chart 层的 `mediaType` 从 `application/tar+gzip` 变成 `application/vnd.cncf.helm.chart.content.v1.tar+gzip`

感谢您的耐心，helm 团队会继续为 oci registry 提供稳定的本地支持

## 链接

- Registries: <https://helm.sh/docs/topics/registries/>
