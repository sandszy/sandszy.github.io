---
title: 如何在kubeadm搭建的k8s集群中新加入一个node
date: 2022-03-28 22:57:27
tags: kubernetes
---

# vmware虚拟机的配置
## 创建虚拟机
以opensuse_leap_15.3_node0为模板复制一个新的node节点虚拟机

## 为虚拟机配置静态IP地址

在配置静态IP后，在/etc/hosts中配置node名称和IP地址的映射

```
192.168.221.130 k8s-node2
```

# 配置node节点的初始环境

> 参考：https://v1-21.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

## Installing runtime

可以采用suse的源安装docker运行时

```shell
#安装
zypper in docker
#启动
systemctl enable docker
systemctl start docker

localhost:~ # docker version
Client:
 Version:           20.10.12-ce
 API version:       1.41
 Go version:        go1.16.13
 Git commit:        459d0dfbbb51
 Built:             Mon Jan 17 12:00:00 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server:
 Engine:
  Version:          20.10.12-ce
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.16.13
  Git commit:       459d0dfbbb51
  Built:            Mon Jan 17 12:00:00 2022
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v1.4.12
  GitCommit:        7b11cfaabd73bb80907dd23182b9347b4245eb5d
 runc:
  Version:          1.0.3
  GitCommit:        
 docker-init:
  Version:          0.1.5_catatonit
  GitCommit:        
localhost:~ # 

```

配置dockerd代理，便于顺利拉取镜像

```shell
mkdir -p /etc/systemd/system/docker.service.d
vi http-proxy.conf
```

内容如下：

```
[Service]
Environment="HTTP_PROXY=http://192.168.159.1:7890/"
Environment="HTTPS_PROXY=http://192.168.159.1:7890/"
Environment="NO_PROXY=localhost,127.0.0.1,.example.com"
```

配置dockerd的storage driver为overlay2fs，配置dockerd的cgroupfs为systemd，不然添加node节点时，k8s校验不通过

修改配置文件/etc/docker/daemon.json

```json
{
  "log-level": "warn",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  },
  "exec-opts": ["native.cgroupdriver=systemd"],
  "storage-driver": "overlay2"
}
```

重启dockerd

```shell
systemctl restart docker
```

## Installing kubeadm, kubelet and kubectl

选择[Without a package manager](https://v1-21.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#k8s-install-2)的模式安装

### Install CNI plugins

(required for most pod network)

```shell
CNI_VERSION="v0.8.2"
ARCH="amd64"
sudo mkdir -p /opt/cni/bin
curl -L "https://github.com/containernetworking/plugins/releases/download/${CNI_VERSION}/cni-plugins-linux-${ARCH}-${CNI_VERSION}.tgz" | sudo tar -C /opt/cni/bin -xz
```

### Install crictl
(required for kubeadm / Kubelet Container Runtime Interface (CRI))

```shell
DOWNLOAD_DIR=/usr/local/bin
sudo mkdir -p $DOWNLOAD_DIR

CRICTL_VERSION="v1.17.0"
ARCH="amd64"
curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-${ARCH}.tar.gz" | sudo tar -C $DOWNLOAD_DIR -xz
```

### Install `kubeadm`, `kubelet`, `kubectl` and add a `kubelet` systemd service:

```shell
#RELEASE="$(curl -sSL https://dl.k8s.io/release/stable.txt)"
#不能采用自动获取的稳定版本，自动获取的为最新的稳定版，需要人工指定版本
RELEASE="v1.21.2"
ARCH="amd64"
cd $DOWNLOAD_DIR
sudo curl -L --remote-name-all https://storage.googleapis.com/kubernetes-release/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet,kubectl}
sudo chmod +x {kubeadm,kubelet,kubectl}

RELEASE_VERSION="v0.4.0"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service
sudo mkdir -p /etc/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

Enable and start `kubelet`:

```bash
systemctl enable --now kubelet
```

# 添加NODE节点

> 参考：https://v1-21.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes

## [master节点]创建token

```bash
kubeadm token create
#查看
kubeadm token list
```

## [master节点]获取token-ca-cert-hash

```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

## [node节点]执行添加命令

```bash
kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
```

其中<control-plane-host>:<control-plane-port>为

```bash
kubeadm join --token 7smcf2.esphmekugtna7ua4 192.168.221.128:6443 --discovery-token-ca-cert-hash sha256:cb0be251cd8ec1498e1e30a0249e5cc7741945ea5caa1a2e52f0bd45fda03a44
```

安装过程中出现如下报错，需要安装conntrack-tools socat

```bash
k8s-node2:~ # kubeadm join --token 7smcf2.esphmekugtna7ua4 192.168.221.128:6443 --discovery-token-ca-cert-hash sha256:cb0be251cd8ec1498e1e30a0249e5cc7741945ea5caa1a2e52f0bd45fda03a44
[preflight] Running pre-flight checks
[WARNING FileExisting-socat]: socat not found in system path
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR FileExisting-conntrack]: conntrack not found in system path
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

```bash
zypper in conntrack-tools socat
```

成功加入node的回显

```shell
k8s-node2:/var/lib/docker # kubeadm join --token 7smcf2.esphmekugtna7ua4 192.168.221.128:6443 --discovery-token-ca-cert-hash sha256:cb0be251cd8ec1498e1e30a0249e5cc7741945ea5caa1a2e52f0bd45fda03a44
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

## [master节点]查看node节点信息

```shell
localhost:~ # kubectl get nodes -o wide
NAME                    STATUS   ROLES                  AGE     VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION         CONTAINER-RUNTIME
k8s-node1               Ready    <none>                 273d    v1.21.2   192.168.221.129   <none>        openSUSE Leap 15.3   5.3.18-59.10-default   docker://20.10.6-ce
k8s-node2               Ready    <none>                 2m31s   v1.21.2   192.168.221.130   <none>        openSUSE Leap 15.3   5.3.18-59.10-default   docker://20.10.12-ce
localhost.localdomain   Ready    control-plane,master   274d    v1.21.2   192.168.221.128   <none>        openSUSE Leap 15.3   5.3.18-59.10-default   docker://20.10.6-ce
```

