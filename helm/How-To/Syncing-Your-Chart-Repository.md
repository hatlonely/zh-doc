# 同步 chart 到 repository

注意：本示例专门针对谷歌云存储桶（GCS）作为 chart repository。

## 先决条件

- 安装 gsutil 工具。我们强依赖 gsutil 的 rsync 功能
- 保证你有权限访问 helm 二进制
- 可选：我们建议你开启了 GCS 的 [object versioning](https://cloud.google.com/storage/docs/gsutil/addlhelp/ObjectVersioningandConcurrencyControl#top_of_page) 功能，以防你不小心删除了什么东西。

## 创建一个本地 chart repository 目录

像我们之前在 [chart repostory 指南](https://helm.sh/docs/topics/chart_repository/) 做得一样，创建一个本地目录，然后将打包好的 chart 放到这个目录

举个例子：

```
$ mkdir fantastic-charts
$ mv alpine-0.1.0.tgz fantastic-charts/
```

## 更新 index.yaml

将目录路径和远端 repository 地址给 `helm repo index` 命令，可以更新 index.yaml，就像这样：

```
$ helm repo index fantastic-charts/ --url https://fantastic-charts.storage.googleapis.com
```

这会更新 index.yaml 文件，并将它放在 `fantastic-charts/` 目录。

## 同步本地和远端的 chart repository

运行 `scripts/sync-repo.sh` 将目录中的内容上传到 GCS 桶，需要传递本地目录名字和 GCS 桶名字作为参数。

举个例子：

```
$ pwd
/Users/me/code/go/src/helm.sh/helm
$ scripts/sync-repo.sh fantastic-charts/ fantastic-charts
Getting ready to sync your local directory (fantastic-charts/) to a remote repository at gs://fantastic-charts
Verifying Prerequisites....
Thumbs up! Looks like you have gsutil. Let's continue.
Building synchronization state...
Starting synchronization
Would copy file://fantastic-charts/alpine-0.1.0.tgz to gs://fantastic-charts/alpine-0.1.0.tgz
Would copy file://fantastic-charts/index.yaml to gs://fantastic-charts/index.yaml
Are you sure you would like to continue with these changes?? [y/N]} y
Building synchronization state...
Starting synchronization
Copying file://fantastic-charts/alpine-0.1.0.tgz [Content-Type=application/x-tar]...
Uploading   gs://fantastic-charts/alpine-0.1.0.tgz:              740 B/740 B
Copying file://fantastic-charts/index.yaml [Content-Type=application/octet-stream]...
Uploading   gs://fantastic-charts/index.yaml:                    347 B/347 B
Congratulations your remote chart repository now matches the contents of fantastic-charts/
```

## 更新你的 chart repository

你要保留 chart repository 的本地副本，或者使用 `gsutil rsync` 来拷贝你的远端的 chart repostory 到本地目录。

举个例子：

```
$ gsutil rsync -d -n gs://bucket-name local-dir/    # the -n flag does a dry run
Building synchronization state...
Starting synchronization
Would copy gs://bucket-name/alpine-0.1.0.tgz to file://local-dir/alpine-0.1.0.tgz
Would copy gs://bucket-name/index.yaml to file://local-dir/index.yaml

$ gsutil rsync -d gs://bucket-name local-dir/       # performs the copy actions
Building synchronization state...
Starting synchronization
Copying gs://bucket-name/alpine-0.1.0.tgz...
Downloading file://local-dir/alpine-0.1.0.tgz:                        740 B/740 B
Copying gs://bucket-name/index.yaml...
Downloading file://local-dir/index.yaml:
```

有用的链接：

- [gsutil rsync](https://cloud.google.com/storage/docs/gsutil/commands/rsync#description) 文档
- [chart repository 指南](https://helm.sh/docs/topics/chart_repository/)
- GCS 文档 [object versioning and concurrency control](https://cloud.google.com/storage/docs/gsutil/addlhelp/ObjectVersioningandConcurrencyControl#overview)

## 链接

- Syncing Your Chart Repository: <https://helm.sh/docs/howto/chart_repository_sync_example/>
