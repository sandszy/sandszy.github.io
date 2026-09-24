---
title: 如何使用kind创建一个kubernetes开发环境
date: 2025-05-29 11:15:07
tags: kubernetes
---

# 如何用kind创建一个k8s集群

参考其官网：https://kind.sigs.k8s.io/docs/user/quick-start/

选择[Installing From Release Binaries](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries)

配置后连接集群的kubeconfig在：root用户的 ~/.kube/config中



其中kind的配置文件如下：

```shell
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  # 将地址改为宿主机IP或0.0.0.0
  # 注意：使用0.0.0.0会监听所有接口，可能存在安全风险
  apiServerAddress: "0.0.0.0" # 或者换成你的宿主机IP，或者0.0.0.0
  # 如果需要指定端口，可以设置：
  # apiServerPort: 6443

nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      certSANs:
      - "localhost"
      - "127.0.0.1"
      - "0.0.0.0"
      - "192.168.2.3"        # 你的主机IP，当集群部署在虚拟机网络环境内时，必须配置，不然kubectl无法通过证书连接集群
      - "primary"       # 其他要访问的主机IP
      - "kubernetes"            # 服务名
      - "kubernetes.default"
      - "kubernetes.default.svc"
      - "kubernetes.default.svc.cluster.local"
- role: worker
- role: worker
- role: worker
```

