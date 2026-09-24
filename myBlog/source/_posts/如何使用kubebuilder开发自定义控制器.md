---
title: 如何使用kubebuilder开发自定义控制器
date: 2025-12-17 15:17:38
tags: kubernetes
---

# 安装

参考官方文档：https://book.kubebuilder.io/quick-start.html

# 注意要点

1. 创建项目

```shell
mkdir guestbook
cd guestbook
# --domain用于拼接API的group
# --repo相当于go mod中的模块名称
kubebuilder init --domain szy.domain --repo szy.domain/guestbook
```

2. 创建API

```shell
# --controller 生成controller实例代码
# --resource 生成自定义对象的结构体代码
# --group 的值会和第一步中domain的值拼接形成最终的group
kubebuilder create api --group webapp --version v1 --kind Guestbook --controller --resource
```

> [!NOTE]
>
> 结构体的代码生成在api/vi目录下，对应的控制器代码生成在internal/controller目录下。
>
> 如果修改了xxx_types.go的结构体代码需要，只需make generate命令重新生成深拷贝等相关代码文件，具体其执行逻辑可以看Makefile下的".PHONY: generate"部分。

3. 创建crd定义，rbac权限等。If you are editing the API definitions, generate the manifests such as Custom Resources (CRs) or Custom Resource Definitions (CRDs) using

```shell
# 该命令会根据internal/controller/guestbook_controller.go文件中的+kubebuilder相关的脚手架注释生成yaml配置文件到config目录下。
make manifests
```

4. 主要控制器逻辑的开发在internal/controller/guestbook_controller.go文件的Reconcile方法中。
5. 核心注册控制器的方法如下：

```go
// 该方法的核心目的是：将 Guestbook 控制器与 Manager 关联，定义控制器要监听的资源类型，并完成控制器的初始化注册，使控制器能够加入 Kubernetes 的调和（Reconcile）循环。
func (r *GuestbookReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).  // 1. 基于 Manager 创建控制器构建器
		For(&webappv1.Guestbook{}).          // 2. 指定控制器的核心监听资源
		Named("guestbook").                  // 3. 为控制器命名
		Complete(r)                          // 4. 完成控制器初始化并关联 Reconciler
}
```

# 例子

参考官方文档：https://book.kubebuilder.io/getting-started.html

```shell
mkdir memcached-operator
cd memcached-operator
kubebuilder init --domain szy.domain --repo szy.domain/memcached
kubebuilder create api --group cache --version v1alpha1 --kind Memcached --controller --resource
```

