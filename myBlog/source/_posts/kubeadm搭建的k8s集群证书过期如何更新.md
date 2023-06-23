---
title: kubeadm搭建的k8s集群证书过期如何更新
date: 2023-06-23 23:25:09
tags: kubernetes
---

- 参考文档：https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/

# 检查证书是否过期

```shell
kubeadm certs check-expiration
#运行后发现默认情况下除了CA证书是8年过期，其他包括etcd在内的所有证书均是1年过期，为已过期状态
```

# 手动更新证书

```shell
kubeadm certs renew
#执行后需要重启apiserver etcd scheduler和controller-manager，但是使用如下参考文档中的做法未生效
#要重启静态 Pod 你可以临时将清单文件从 /etc/kubernetes/manifests/ 移除并等待 20 秒 （参考 KubeletConfiguration 结构 中的fileCheckFrequency 值）。 
#所以通过docker restart命令重启如上4个容器
docker restart <容器id>
#如上命令执行完成后集群恢复正常
```



# [重要]2023年6月23日更新

由于kubelet的证书更新不在其中：

> **Note:** `kubelet.conf` is not included in the list above because kubeadm configures kubelet for [automatic certificate renewal](https://kubernetes.io/docs/tasks/tls/certificate-rotation/) with rotatable certificates under `/var/lib/kubelet/pki`. To repair an expired kubelet client certificate see [Kubelet client certificate rotation fails](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/#kubelet-client-cert).

在kubelet证书过期后，管控面的kubelet均无法启动，导致无法更新证书。解决方法如下：

首先要手段启动管控面的etcd、apiserver、scheduler、controller-manager：

```shell
docker start <k8s network container id> <etcd container id>
docker start <k8s network container id> <apiserver container id>
docker start <k8s network container id> <scheduler container id>
docker start <k8s network container id> <controller-manager container id>
```

然后参考https://v1-23.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/#kubelet-client-cert对kubelet进行修复：

## Kubelet client certificate rotation fails

By default, kubeadm configures a kubelet with automatic rotation of client certificates by using the `/var/lib/kubelet/pki/kubelet-client-current.pem` symlink specified in `/etc/kubernetes/kubelet.conf`. If this rotation process fails you might see errors such as `x509: certificate has expired or is not yet valid` in kube-apiserver logs. To fix the issue you must follow these steps:

1. Backup and delete `/etc/kubernetes/kubelet.conf` and `/var/lib/kubelet/pki/kubelet-client*` from the failed node.

2. From a working control plane node in the cluster that has `/etc/kubernetes/pki/ca.key` execute `kubeadm kubeconfig user --org system:nodes --client-name system:node:$NODE > kubelet.conf`. `$NODE` must be set to the name of the existing failed node in the cluster. Modify the resulted `kubelet.conf` manually to adjust the cluster name and server endpoint, or pass `kubeconfig user --config` (it accepts `InitConfiguration`). If your cluster does not have the `ca.key` you must sign the embedded certificates in the `kubelet.conf` externally.

3. Copy this resulted `kubelet.conf` to `/etc/kubernetes/kubelet.conf` on the failed node.

4. Restart the kubelet (`systemctl restart kubelet`) on the failed node and wait for `/var/lib/kubelet/pki/kubelet-client-current.pem` to be recreated.

5. Manually edit the `kubelet.conf` to point to the rotated kubelet client certificates, by replacing `client-certificate-data` and `client-key-data` with:

   ```yaml
   client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
   client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
   ```

6. Restart the kubelet.

7. Make sure the node becomes `Ready`.



