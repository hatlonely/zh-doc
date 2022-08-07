# kubernetes 分发指南

helm 可以在任何[标准的 kubernetes 版本](https://github.com/cncf/k8s-conformance)上运行（无论是否经过[认证](https://www.cncf.io/certification/software-conformance/)）。

这个文档捕获了关于在特定的 kubernetes 环境下使用 helm 的信息。如果需要，请提供更多关于发行版本（按字母顺序排序）的详细信息。

## AKS

helm 可以在 [Azure kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/kubernetes-helm) 下工作。

## DC/OS

helm 在 Mesospheres DC/OS 1.11 kubernetes 平台上已经经过测试，并可以正常工作，并且不需要额外的配置。

## EKS

helm 可以在 Amazon Elastic Kubernetes Service (Amazon EKS) 中工作：[在 Amazon EKS 中使用 helm](https://docs.aws.amazon.com/eks/latest/userguide/helm.html)

## GKE

Google 的 GKE 托管的 kubernetes 平台是已知可以使用 helm 的，并且不需要额外的配置

## Hyperkube

HyperKube 通过 `scripts/local-cluster.sh` 已知是可以工作的。对于纯 Hyperkube 你可能需要做一些手动的配置。

## IKS

helm 可以在 [IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers?topic=containers-helmhttps://cloud.ibm.com/docs/containers?topic=containers-helm) 中工作。

## KIND（kubernetes in docker）

helm 在 [KIND](https://github.com/kubernetes-sigs/kind) 中通过了常规测试。

## KubeOne

helm 在通过 KubeOne 创建的集群中可以正常工作，并且没有警告。

## Kubermatic

helm 在通过 Kubermatic 创建的用户的集群中可以正常工作，并且没有警告。因为种子集群可以通过不同的方式创建，helm 的支持取决于他们的配置。

## MincroK8s

helm 可以在 microk8s 使用命令 `microk8s.enable helm3` 中启用。

## Minikube

helm 在 minikube 中可以正常工作，并且测试通过。它不需要额外的配置。

## Openshift

helm 可以直接在 OpenShift Online, OpenShift Dedicated, OpenShift Container Platform (版本 >= 3.6) 或者 OpenShift Origin (版本 >= 3.6) 上工作。了解更多，阅读这个[博客](https://blog.openshift.com/getting-started-helm-openshift/)

## Platform9

helm 预装在 [Platform9 Managed Kubernetes](https://platform9.com/managed-kubernetes/?utm_source=helm_distro_notes) 中。Platform9 提供了通过 App Catalog UI 访问所有官方 helm chart 的客户端。其他的 repository 可以手动添加。更多细节参考这个 [Platform9 App Catalog 文章](https://platform9.com/support/deploying-kubernetes-apps-platform9-managed-kubernetes/?utm_source=helm_distro_notes)。

## 装有 kubeadm 的 Ubuntu

众所周知，使用 `kuebadm` 引导的 kubernetes 可以在下面的 linux 发行版本中工作：

- Ubuntu 16.04
- Fedora release 25

一些 helm 版本（v2.0.0-beta2）需要你 `export KUBECONFIG=/etc/kubernetes/admin.conf` 或者创建一个 `~/.kube/config`。

## 链接

- Kubernetes Distribution Guide: <https://helm.sh/docs/topics/kubernetes_distros/>
