# helm 来源和完整性

helm 拥有来源工具，帮助 chart 用户校验包的完整性和出处。使用企业标准的工具，基于 PKI，GnuPG，以及流行的包管理器，helm 可以生成和校验签名文件。

## 概览

完整性是通过比较 chart 和出处记录来简历的。出处记录保存在出处文件中，和 chart 包保存在一起。比如，如果一个 chart 的名字是 `myapp-1.2.3.tgz`，它的出处文件会是 `myapp.1.2.3.tgz.prov`。

出处文件在打包的时候生成（`helm package --sign ...`），可以被多个命令校验，特别是 `helm install --verify`。

## 工作流

这个章节描述了一个潜在的有效使用出处文件的工作流。

先决条件：

- 有效的二进制（非 ASCII 码）格式的 PGP 秘钥对
- `helm` 命令行工具
- `GnuPG` 命令行工具（可选）
- `Keybase` 命令行工具（可选）

注意：如果你的 PGP 私钥有密码。在所有支持 --sign 选项的命令中，你会被提示要输入密码。

创建一个和之前一样的新的 chart：

```
$ helm create mychart
Creating mychart
```

一旦做好了打包的准备，在 `helm package` 选项中增加 `--sign` 选项。同时，指定已知的签名秘钥和私钥环。

```
$ helm package --sign --key 'John Smith' --keyring path/to/keyring.secret mychart
```

注意：`--key` 参数的值，必须是相应秘钥 `uid`（`gpg --list-keys` 的输出） 的字串，比如 email 的名字。指纹不能使用。

小贴士：对于 `GnuPG` 用户，你的秘钥环在 `~/.gnupg/secring.gpg`。你可以使用 `gpg --list-secret-keys` 来列出你拥有的秘钥。

警告：GnuPG v2 使用了新的格式 `kbx` 来保存你的你秘钥环，默认在 `~/.gnupg/pubring.kbx`。使用如下的命令来转换你的秘钥环到老的 gpg 格式：

```
$ gpg --export >~/.gnupg/pubring.gpg
$ gpg --export-secret-keys >~/.gnupg/secring.gpg
```

这个时候，你应该能看到 `mychart.0.1.0.tgz` 和 `mychart-0.1.0.tgz.prov`。两个文件都会最终上传到所需的 chart repository 中。

你可以使用 `helm verify` 校验 chart：

```
$ helm verify mychart-0.1.0.tgz
```

一个失败的校验看起来像这样：

```
$ helm verify topchart-0.1.0.tgz
Error: sha256 sum does not match for topchart-0.1.0.tgz: "sha256:1939fbf7c1023d2f6b865d137bbb600e0c42061c3235528b1e8c82f4450c12a7" != "sha256:5a391a90de56778dd3274e47d789a2c84e0e106e1a37ef8cfa51fd60ac9e623a"
```

在安装的时候使用 `--verify` 参数来校验。

```
$ helm install --generate-name --verify mychart-0.1.0.tgz
```

如果签名的 chart 关联的包含公共秘钥的秘钥环不在默认的地址，你可能需要使用 `--keyring PATH` 指定秘钥环，和在 `helm package` 例子中一样。

如果校验失败，安装会在渲染之前中止。

### 使用 kybase.io 证书

[keybase.io](https://keybase.io/) 服务让创建一个加密标识的新人链变得非常容易。keybase 证书可以用来签名 chart。

先决条件：

- 配置过的 keybase.io 账号
- 本地安装 GnuPG
- 本地安装 `keybase` 客户端

### 签名包

第一步是导入你的 keybase 秘钥到你本地的 GnuPG 秘钥链中：

```
$ keybase pgp export -s | gpg --import
```

这个会将你的 keybase 秘钥转换成 OpenPGP 格式，然后导入到你本地的 `~/.gnupg/secing.gpg` 文件中。

你可以运行 `gpg --list-secret-keys` 来二次检查。

```
$ gpg --list-secret-keys
/Users/mattbutcher/.gnupg/secring.gpg
-------------------------------------
sec   2048R/1FC18762 2016-07-25
uid                  technosophos (keybase.io/technosophos) <technosophos@keybase.io>
ssb   2048R/D125E546 2016-07-25
```

注意你的秘钥会有一个标识符字符串：

```
technosophos (keybase.io/technosophos) <technosophos@keybase.io>
```

这个是秘钥的全名。

然后你可以用 `helm package` 打包和签名 chart。确保你至少在 `--key` 中使用了那个名字字符串中的一部分。

```
$ helm package --sign --key technosophos --keyring ~/.gnupg/secring.gpg mychart
```

结果是，`package` 这个命令同时产生了 `.tgz` 文件和 `.tgz.prov` 文件。

### 校验包

你也可以用同样的技术来校验一个被某人或者 keybase 签名过得 chart。比如你想要校验一个 `keybase.io/technospohos` 签名过得包。使用 `keybase` 工具来做到这个：

```
$ keybase follow technosophos
$ keybase pgp pull
```

上面第一个命令追踪了用户 technosophos。然后 `keybase pgp pull` 下载了所有你关注账号的 OpenPGP 秘钥，把他们放到你的 GnuPG 秘钥环中（~/.gnupg/pubring.gpg）。

这个时候，你可以用 `helm verify` 或者其他任何有 `--verify` 参数的命令了。

```
$ helm verify somechart-1.2.3.tgz
```

### chart 没有校验通过的原因

这里有一些通常失败的原因。

- `.prov` 文件丢失或者有错误。这表明有些东西没有配置好，或者原始的维护者没有创建出处文件。
- 签名文件的秘钥不在你的秘钥环中。这表明签名 chart 的实体不是你已经标记为信任的人。
- `.prov` 文件校验失败。这表明 chart 或者出处数据有地方出错了。
- 出处文件中的 hash 值，和归档文件的 hash 值不匹配。这表明归档文件已经被篡改过。

如果校验失败，有理由不信任这个包。

## 出处文件

出处文件包含了 chart 的 yaml 文件和一些校验信息。出处文件是自动生成的。

出处文件数据添加了如下内容：

- 包含 chart 文件（`Chart.yaml`），让人和工具都很容看到 chart 的内容。
- 包含 chart 包（`.tgz` 文件）的签名（SHA256，和 Docker 一样）。
- 整个内容使用 OpenPGP 使用的算法来签名（有关加密签名和验证的新兴方法，请参阅[keybase.io](https://keybase.io/)）

这些组合给了用户如下的保证：

- 包本身没有被篡改过（`.tgz` 包的校验和）
- 谁发布的这个包是已知的（通过 GnuPG/PGP 签名）

文件的格式看起来像这样：

```
Hash: SHA512

apiVersion: v2
appVersion: "1.16.0"
description: Sample chart
name: mychart
type: application
version: 0.1.0

...
files:
  mychart-0.1.0.tgz: sha256:d31d2f08b885ec696c37c7f7ef106709aaf5e8575b6d3dc5d52112ed29a9cb92
-----BEGIN PGP SIGNATURE-----

wsBcBAEBCgAQBQJdy0ReCRCEO7+YH8GHYgAAfhUIADx3pHHLLINv0MFkiEYpX/Kd
nvHFBNps7hXqSocsg0a9Fi1LRAc3OpVh3knjPfHNGOy8+xOdhbqpdnB+5ty8YopI
mYMWp6cP/Mwpkt7/gP1ecWFMevicbaFH5AmJCBihBaKJE4R1IX49/wTIaLKiWkv2
cR64bmZruQPSW83UTNULtdD7kuTZXeAdTMjAK0NECsCz9/eK5AFggP4CDf7r2zNi
hZsNrzloIlBZlGGns6mUOTO42J/+JojnOLIhI3Psd0HBD2bTlsm/rSfty4yZUs7D
qtgooNdohoyGSzR5oapd7fEvauRQswJxOA0m0V+u9/eyLR0+JcYB8Udi1prnWf8=
=aHfz
-----END PGP SIGNATURE-----
```

注意这个 yaml 章节包含了两个文档（使用 `...\n` 分隔）。第一个文件的内容是 `Chart.yaml`。第二个是校验和，文件内容在打包时映射文件名的 SHA-256 摘要。

签名块是一个标准的 PGP 签名，由[拒绝篡改](https://www.rossde.com/PGP/pgp_signatures.html)提供。

## chart repository

chart repository 作为一个中心的 helm chart 集合提供服务。

chart repository 必须能够通过特定的 http 请求提供源文件，并且必须使他们和 chart 在同样的 uri 路径上可用。

比如，如果包的基本 url 是 `https://example.com/charts/mychart-1.2.3.tgz`，如果存在出处文件，必须在 `https://example.com/charts/mychart-1.2.3.tgz.prov` 可以访问。

从终端用户的角度来看，`helm install --verify myrep/mychart-1.2.3` 应该同时下载 chart 和出处文件，而不需要额外的用户配置或者操作。

## 建立权威性和真实性

在处理信任链的系统中，建立签名者的权威性很重要。或者，简单地说，上面的系统基于一个事实，你信任那个签名 chart 的人。这反过来意味着，你需要信任签名者的公钥。

helm 的设计决策之一是，helm 项目不会作为必要的一方插入到信任链中。我们不想成为所有 chart 签名者的”权威证书“。相反，我们强烈支持去中心化的模型，也是我们选择 OpenPGP 作为基础技术的部分原因。所以在设计建立权威性的问题上，我们在 helm 2 中或多或少没有去定义这个步骤（接下来 helm 3 中决定）。

然后，我们已经有一些观点和建议给这些对使用来源系统感兴趣的人：

- keybase 平台提供了一个信任信息的中心化的仓库。
    - 你可以使用 keybase 来存储你的秘钥或者获取其他人的公钥
    - keybase 还提供了非常好的文档
    - 尽管我们还没有测试它，keybase 的”安全网站“特性可以用来为 helm chart 服务
    - 基本想法是一个官方的”chart 审核员“用他的私钥签名 chart，并上传来源文件到 chart repository 中
    - 这个想法有一些工作要做，在 repository 的 `index.yaml` 文件中应该包含有效秘钥的列表

## 链接

- Helm Provenance and Integrity: <https://helm.sh/docs/topics/provenance/>
