---
title: 如何做mac下创建一个轻量级的虚拟机
date: 2025-12-16 14:37:17
tags: MAC skills
---

# 需求

我的需求是在mac上安装类似一个windows wsl的虚拟机，随时可以中mac的命令行中切换。无需像vmware等虚拟机，需要配置复杂的网络，并且用ssh连接。

在做kubernetes等集群开发时，只需要用kind模拟多个node节点即可。

# 方案

可以选择Multipass这个ubuntu官方提供的虚拟化，虽然只可以安装ubuntu系统，但是着足够了。

# 操作方式

根据官方文档安装即可：

https://documentation.ubuntu.com/multipass/latest/how-to-guides/install-multipass/

安装后创建虚拟机实例：

https://documentation.ubuntu.com/multipass/latest/how-to-guides/manage-instances/create-an-instance/#how-to-guides-manage-instances-create-an-instance

记得把CPU调整成8，内存调整成12G，磁盘调整成40G，才够用。（我的电脑是M4 16G 256G的Macbook AIr）

multipass的基本操作

```shell
#列出虚拟机
multipass list
#启动虚拟机，其中primary是虚拟机的名字
multipass start primary
```

