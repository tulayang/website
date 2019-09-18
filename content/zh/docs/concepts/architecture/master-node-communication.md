---
approvers:
- dchen1107
- roberthbailey
- liggitt

title: Master-Node 通信
---

{{< toc >}}


## 概览


本文对 master（准确说是 apiserver）和 Kubernetes cluster 之间的通信路径进行分类。目的是允许用户自定义安装，对网络配置进行加固，使集群安全运行在不可信的网络上（或者运行在一云服务商完全公开的 IP 上）。


## Cluster 到 Master


从 cluster 到 master 的所有通信路径，其终点是 apiserver（根据设计，其它 master 组件不提供远程服务）。对于一个经典部署，apiserver 配置为监听安全 HTTPS 端口（443），并且启用一种或多种客户端[身份认证](/docs/admin/authentication/)策略。（为了安全）应该启用一种或多种[权限控制](/docs/admin/authentication/)，特别是允许 [匿名请求（anonymous requests）](/docs/admin/authentication/#anonymous-requests) 或 [服务账户通行证（service account tokens）](/docs/admin/authentication/#service-account-tokens) 的时候。


应该为 nodes 提供公开的根证书（public root certificate），这样，它们就能使用有效的客户端证件（credentials）安全地连接 apiserver。例如：对于一个默认配置的 GCE deployment，其提供给 kubelet 的客户端证件（credentials），是一个客户端证书（certificate）。请查看 [kubelet TLS bootstrapping](/docs/admin/kubelet-tls-bootstrapping/) 获取如何自动提供 kubelet 客户端证书。


Pods 可以借助一个 service account 以安全地连接 apiserver，这种情况下，当一个 pod 被实例化时，Kubernetes 自动将公开的根证书（the public root certificate）和一个有效的不记名通行证（a valid bearer token）注入到 pod。`kubernetes` service （所有 namespaces 中）配置一个虚拟 IP 地址，用于转发（通过 kube-proxy）请求到 apiserver 的 HTTPS 端。


Master 组件通过非安全（没有加密或认证）端口和集群的 apiserver 通信。这个端口通常只在 master 节点的 localhost 接口暴露，这样，所有在相同机器上运行的 master 组件就能和集群的 apiserver 通信。一段时间以后，master 组件将变为使用带身份认证和权限验证的安全端口（查看[#13598](https://github.com/kubernetes/kubernetes/issues/13598)）。


结果是，cluster（nodes 和运行在其上的 pods）到 master 的默认连接过程是安全的，能够安全运行在不可信或公开的网络中。


## Master 到 Cluster


从 master（apiserver）到 cluster 有两种主要的通信路径。第一种是从 apiserver 到 kubelet 进程，kubelet 进程运行在 cluster 所在的 node 上。第二种是从 apiserver 通过它的代理功能到任何 node、pod 或者 service。


### apiserver 到 kubelet


从 apiserver 连接 kubelet 用于：

- 获取 pods 日志
- 链接（通过 kubectl）运行中的 pods
- 提供 kubelet 的端口转发功能


这些连接的终点是 kubelet 的 HTTPS 端。默认，apiserver 不验证 kubelet 的服务证书，这会导致连接遭到中间人攻击，因而在不可信或公开的网络上是不安全的。  


为了验证连接，请使用 `--kubelet-certificate-authority` 标志给 apiserver 提供一个根证书包（a root certificate bundle），以验证 kubelet 的服务证书。


如果上面无法做到，又必须运行在不可信或公开的网络上，请在 apiserver 和 kubelet 之间使用 [SSH 隧道](/docs/concepts/architecture/master-node-communication/#ssh-tunnels)。


最后，应该启用[Kubelet 身份认证和/或权限认证](/docs/admin/kubelet-authentication-authorization/)以提供 kubelet API 安全。


### apiserver 到 nodes, pods, and services


从 apiserver 到 node、pod 或者 service 的连接默认是 HTTP 连接，因此既没有认证，也没有加密。为 node、pod 或 service 的 API URL 提供 `https:` 前缀，能够使其运行在安全的 HTTPS 连接。但是，在这种情况下，即不验证 HTTPS 各端提供的证书，也不（自动）提供客户端证件。这样，连接虽然是加密的，但是无法保证连接的各端是可信的。当前，在不可信或公开的网络上运行这些连接**仍然是不安全的**。


### SSH 隧道


[Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/docs/) 支持 SSH 隧道保护 Master -> Cluster 通信路径。在这种配置下，apiserver 与 cluster 的各个 node 建立一个 SSH 隧道（连接到 ssh server -- 监听端口 22 ），并通过隧道传输所有到 kubelet、node、pod 或者 service 的流量。隧道保证流量不暴露在 nodes 运行的网络之外。

现在，SSH 隧道是弃用的。你不应该使用它们，当然我们也无法阻止你非要这么做，假如你知道这么做的后果。我们正在开发一个替代品，以替代这种通信信道。
