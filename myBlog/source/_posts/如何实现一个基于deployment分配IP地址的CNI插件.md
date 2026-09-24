---
title: 如何实现一个基于deployment分配IP地址的CNI插件
date: 2025-07-22 11:12:05
tags: kubernetes
---

# 如何用kind搭建一个自带calico CNI的kubernetes集群

主要参考其官网：https://docs.tigera.io/calico/latest/getting-started/kubernetes/kind

- 注意事项1：如果遇到node节点中的containerd无法拉去镜像，则需要对容器内的containerd配置代理。方法如下：

1. **登录工作节点**

   ```
   docker exec -it kind-20250714-worker bash
   docker exec -it kind-20250714-control-plane bash
   ```

2. **创建 systemd 代理配置文件**

   ```
   mkdir -p /etc/systemd/system/containerd.service.d
   cat > /etc/systemd/system/containerd.service.d/http-proxy.conf <<EOF
   [Service]
   Environment="HTTP_PROXY=http://172.30.16.1:7890/"
   Environment="HTTPS_PROXY=http://172.30.16.1:7890/"
   Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,.svc,.cluster.local,.example.com"
   EOF
   ```

3. **重启 containerd 并应用配置**

   ```
   systemctl daemon-reload
   sleep 1
   systemctl restart containerd
   sleep 1
   systemctl status containerd
   ```

> [!NOTE]
>
> 记得把代理开到全局模式。

# 安装calico配置工具calicoctl

主要参考文档：https://archive-os-3-25.netlify.app/calico/3.25/operations/calicoctl/install

选择Install calicoctl as a binary on a single host.

该工具可以查看calico各种配置是否符合spiderpool的要求。

- 注意事项1：选择安装和集群版本匹配的版本

  ```
  export http_proxy=http://172.30.16.1:7890
  export https_proxy=http://172.30.16.1:7890
  export no_proxy=localhost,127.0.0.1
  cd /usr/local/bin/
  curl -L https://github.com/projectcalico/calico/releases/download/v3.30.2/calicoctl-linux-amd64 -o calicoctl
  chmod +x calicoctl	
  ```

- 注意事项2：要选择进入node节点容器安装calicoctl，才能查看calicoctl node status。查看配置则要在能力联通apiserver的宿主机上。

# 如何配置calico BGP+spider pool

主要参考文档：https://github.com/spidernet-io/spiderpool/blob/main/docs/usage/install/underlay/get-started-calico-zh_CN.md

- 注意事项1：配置FRR中注意要替换IP

```
root@router:~# vtysh
router# config
router(config)# router bgp 23000 
router(config)# bgp router-id 172.18.0.1
router(config)# neighbor 172.18.0.2 remote-as 64512 
router(config)# neighbor 172.18.0.3 remote-as 64512
router(config)# neighbor 172.18.0.4 remote-as 64512
router(config)# no bgp ebgp-requires-policy 
```

- 注意事项2：If the default BGP configuration resource does not exist, you need to create it first. See [BGP configuration](https://docs.tigera.io/calico/latest/reference/resources/bgpconfig) for more information.
- 注意事项3：Disabling the node-to-node mesh will break pod networking until/unless you configure replacement BGP peerings using BGPPeer resources. You may configure the BGPPeer resources before disabling the node-to-node mesh to avoid pod networking breakage.
- 注意事项4：大坑！参考官方文档创建默认的BGP config，其中的listenport要从178改成默认的179。serviceClusterIPs要和apiserver中配置的一致。安装步骤中有一个关闭full mesh的步骤。可以通过修改该配置文件重新apply的方式实现。

```
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: false
  # nodeMeshMaxRestartTime: 120s
  asNumber: 64512
  serviceClusterIPs:
    - cidr: 10.96.0.0/16
  serviceExternalIPs:
    - cidr: 104.244.42.129/32
    - cidr: 172.217.3.0/24
  listenPort: 179
  bindMode: NodeIP
  communities:
    - name: bgp-large-community
      value: 64512:300:100
  prefixAdvertisements:
    - cidr: 172.218.4.0/26
      communities:
        - bgp-large-community
        - 64512:120
```

然后执行命令

```
calicoctl apply -f BGPConfiguration-default.yaml
```

- 注意事项5：如果要在容器内创建网络连通性测试工具nc，方法如下

```
apt update
apt install -y netcat-openbsd
```

使用命令

```
root@kind-20250701-worker2:/usr/local/bin# nc -zv 172.18.0.4 179
Connection to 172.18.0.4 179 port [tcp/bgp] succeeded!
root@kind-20250701-worker2:/usr/local/bin# nc -zv 172.18.0.3 179
Connection to 172.18.0.3 179 port [tcp/bgp] succeeded!
```

- 注意事项6：创建 BGPPeer时，注意修改peerIP为前面frr配置中的router-id，asNumber 为 BGP Router 的 AS 号

```
[root@master1 ~]# cat << EOF | calicoctl apply -f -
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: my-global-peer
spec:
  peerIP: 172.18.0.1
  asNumber: 23000
EOF
```

- 注意事项7：大坑！遇到如下报错日志。

```
2025-07-13 15:39:59.803 [DEBUG][2052254] cni-plugin/utils.go 138: Calling IPAM plugin spiderpool ContainerID="827dfa95bb777502b42bdd770dc9952e3ed04b1ab4df40069d7e03ce8d229219" Namespace="default" Pod="nginx-58f595c86-c5gd6" WorkloadEndpoint="kind--20250701--worker2-k8s-nginx--58f595c86--c5gd6-eth0"
2025-07-13 15:39:59.819 [ERROR][2052254] cni-plugin/plugin.go 162: Final result of CNI ADD was an error. error=failed to set link up: failed to get link: Link not found
```

修改spiderpool/cmd/spiderpool/cmd/command_add.go，把涉及容器内eth0操作的两个步骤注释，只保留ip分配的部分。重新编译后的二进制文件spiderpool更新到宿主机/opt/cni/bin目录下，无需编译重启，立即生效。

```go
// Copyright 2022 Authors of spidernet-io
// SPDX-License-Identifier: Apache-2.0

package cmd

import (
	"context"
	"errors"
	"fmt"
	"net"
	"runtime/debug"
	"time"

	"github.com/containernetworking/cni/pkg/skel"
	"github.com/containernetworking/cni/pkg/types"
	current "github.com/containernetworking/cni/pkg/types/100"
	"github.com/containernetworking/plugins/pkg/ns"
	"github.com/go-openapi/strfmt"
	"go.uber.org/multierr"
	"go.uber.org/zap"

	"github.com/spidernet-io/spiderpool/api/v1/agent/client/connectivity"
	"github.com/spidernet-io/spiderpool/api/v1/agent/client/daemonset"
	"github.com/spidernet-io/spiderpool/api/v1/agent/models"
	"github.com/spidernet-io/spiderpool/pkg/constant"
	spiderpoolip "github.com/spidernet-io/spiderpool/pkg/ip"
	"github.com/spidernet-io/spiderpool/pkg/openapi"
)

// CmdAdd follows CNI SPEC cmdAdd.
func CmdAdd(args *skel.CmdArgs) (err error) {
	var logger *zap.Logger
	// Defer a panic recover, so that in case we panic we can still return
	// a proper error to the runtime.
	defer func() {
		if e := recover(); e != nil {
			msg := fmt.Sprintf("Spiderpool IPAM CNI panicked during ADD: %v", e)

			if err != nil {
				// If it is recovering and an error occurs, then we need to
				// present both.
				msg = fmt.Sprintf("%s: error=%v", msg, err.Error())
			}

			if logger != nil {
				logger.Sugar().Errorf("%s\n\n%s", msg, debug.Stack())
			}
		}
	}()

	conf, err := LoadNetConf(args.StdinData)
	if nil != err {
		return fmt.Errorf("failed to load CNI network configuration: %v", err)
	}

	netns, err := ns.GetNS(args.Netns)
	if err != nil {
		return fmt.Errorf("failed to GetNS %q for pod: %v", args.Netns, err)
	}
	defer netns.Close()

	logger, err = SetupFileLogging(conf)
	if err != nil {
		return fmt.Errorf("failed to setup file logging: %v", err)
	}

	// When IPAM is invoked, the NIC is down and must be set it up in order to detect IP conflicts and
	// gateway reachability.
	// err = netns.Do(func(netNS ns.NetNS) error {
	// 	l, err := netlink.LinkByName(args.IfName)
	// 	if err != nil {
	// 		return fmt.Errorf("failed to get link: %w", err)
	// 	}

	// 	if err = netlink.LinkSetUp(l); err != nil {
	// 		return fmt.Errorf("failed to set link up: %w", err)
	// 	}

	// 	logger.Sugar().Debugf("Set link %s to up for IP conflict and gateway detection", args.IfName)
	// 	return nil
	// })

	// if err != nil {
	// 	return fmt.Errorf("failed to set link up: %w", args.IfName, args.ContainerID, args.Netns, err)
	// }

	hostNs, err := ns.GetCurrentNS()
	if err != nil {
		return fmt.Errorf("failed to get current netns: %v", err)
	}
	defer hostNs.Close()

	logger = logger.Named(BinNamePlugin).With(
		zap.String("Action", "ADD"),
		zap.String("ContainerID", args.ContainerID),
		zap.String("Netns", args.Netns),
		zap.String("IfName", args.IfName),
	)
	logger.Debug("Processing CNI ADD request")
	logger.Sugar().Debugf("CNI network configuration: %+v", *conf)

	k8sArgs := K8sArgs{}
	if err = types.LoadArgs(args.Args, &k8sArgs); nil != err {
		err := fmt.Errorf("failed to load CNI ENV args: %w", err)
		logger.Error(err.Error())
		return err
	}

	logger = logger.With(
		zap.String("PodName", string(k8sArgs.K8S_POD_NAME)),
		zap.String("PodNamespace", string(k8sArgs.K8S_POD_NAMESPACE)),
		zap.String("PodUID", string(k8sArgs.K8S_POD_UID)),
	)
	logger.Sugar().Debugf("CNI ENV args: %+v", k8sArgs)

	spiderpoolAgentAPI, err := openapi.NewAgentOpenAPIUnixClient(conf.IPAM.IPAMUnixSocketPath)
	if nil != err {
		err := fmt.Errorf("failed to create spiderpool-agent client: %w", err)
		logger.Error(err.Error())
		return err
	}

	logger.Debug("Send health check request to spiderpool-agent backend")
	_, err = spiderpoolAgentAPI.Connectivity.GetIpamHealthy(connectivity.NewGetIpamHealthyParams())
	if nil != err {
		err := fmt.Errorf("%w, failed to check: %v", ErrAgentHealthCheck, err)
		logger.Error(err.Error())
		return err
	}

	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Second)
	defer cancel()

	params := daemonset.NewPostIpamIPParams().
		WithContext(ctx).
		WithIpamAddArgs(&models.IpamAddArgs{
			ContainerID:       &args.ContainerID,
			NetNamespace:      &args.Netns,
			IfName:            &args.IfName,
			PodName:           (*string)(&k8sArgs.K8S_POD_NAME),
			PodNamespace:      (*string)(&k8sArgs.K8S_POD_NAMESPACE),
			PodUID:            (*string)(&k8sArgs.K8S_POD_UID),
			DefaultIPV4IPPool: conf.IPAM.DefaultIPv4IPPool,
			DefaultIPV6IPPool: conf.IPAM.DefaultIPv6IPPool,
			CleanGateway:      conf.IPAM.CleanGateway,
		})

	logger.Debug("Send IPAM request")
	ipamResponse, err := spiderpoolAgentAPI.Daemonset.PostIpamIP(params)
	if err != nil {
		err := fmt.Errorf("%w: %v", ErrPostIPAM, err)
		logger.Error(err.Error())
		return err
	}

	// Validate IPAM request response.
	if err = ipamResponse.Payload.Validate(strfmt.Default); nil != err {
		err := fmt.Errorf("%w: %v", ErrPostIPAM, err)
		logger.Error(err.Error())
		return err
	}

	// do ip conflict and gateway detection
	logger.Sugar().Info("postIpam response",
		zap.Any("DNS", ipamResponse.Payload.DNS),
		zap.Any("Routes", ipamResponse.Payload.Routes))

	if err = DetectIPConflictAndGatewayReachable(logger, args.IfName, hostNs, netns, ipamResponse.Payload.Ips); err != nil {
		if errors.Is(err, constant.ErrIPConflict) || errors.Is(err, constant.ErrGatewayUnreachable) {
			logger.Info("failed to detect IP conflict or gateway unreachable, clean up IPs")
			if e := deleteIpamIps(spiderpoolAgentAPI, args, k8sArgs); e != nil {
				logger.Sugar().Errorf("failed to clean up conflict IPs, error: %v", e)
				return multierr.Append(err, e)
			}
			logger.Info("Successfully cleaned up IPs")
		}
		return err
	}

	// CNI will set the interface to up, and the kernel only sends GARPs/Unsolicited NA when the interface
	// goes from down to up or when the link-layer address changes on the interfaces. in order to the
	// kernel send GARPs/Unsolicited NA when the interface goes from down to up.
	// see https://github.com/spidernet-io/spiderpool/issues/4650
	var ipRes []net.IP
	for _, i := range ipamResponse.Payload.Ips {
		if i.Address != nil && *i.Address != "" {
			ipa, _, err := net.ParseCIDR(*i.Address)
			if err != nil {
				logger.Error(err.Error())
				continue
			}
			ipRes = append(ipRes, ipa)
		}
	}

	// err = netns.Do(func(netNS ns.NetNS) error {
	// 	return networking.AnnounceIPs(logger, args.IfName, ipRes)
	// })

	// if err != nil {
	// 	logger.Error(err.Error())
	// }

	// Assemble the result of IPAM request response.
	result, err := assembleResult(conf.CNIVersion, args.IfName, ipamResponse)
	if err != nil {
		err := fmt.Errorf("%w: %v", ErrPostIPAM, err)
		logger.Error(err.Error())
		return err
	}

	logger.Sugar().Infof("IPAM allocation result: %+v", *result)
	return types.PrintResult(result, conf.CNIVersion)
}

// assembleResult groups the IP allocation resutls of IPAM request response
// based on NIC and combines them into CNI results.
func assembleResult(cniVersion, IfName string, ipamResponse *daemonset.PostIpamIPOK) (*current.Result, error) {
	result := &current.Result{
		CNIVersion: cniVersion,
	}

	// Mock DNS.
	if nil != ipamResponse.Payload.DNS {
		result.DNS = types.DNS{
			Nameservers: ipamResponse.Payload.DNS.Nameservers,
			Domain:      ipamResponse.Payload.DNS.Domain,
			Search:      ipamResponse.Payload.DNS.Search,
			Options:     ipamResponse.Payload.DNS.Options,
		}
	}

	var routes []*types.Route
	for _, route := range ipamResponse.Payload.Routes {
		if *route.IfName == IfName {
			_, dst, err := net.ParseCIDR(*route.Dst)
			if err != nil {
				return nil, err
			}
			routes = append(routes, &types.Route{
				Dst: *dst,
				GW:  net.ParseIP(*route.Gw),
			})
		}
	}
	result.Routes = routes

	for _, ip := range ipamResponse.Payload.Ips {
		if *ip.Nic == IfName {
			address, err := spiderpoolip.ParseIP(*ip.Version, *ip.Address, true)
			if err != nil {
				return nil, err
			}
			result.IPs = append(result.IPs, &current.IPConfig{
				Address: *address,
				Gateway: net.ParseIP(ip.Gateway),
			})
		}
	}

	if len(result.IPs) == 0 {
		return nil, fmt.Errorf("no Interface %s IP allocation found", IfName)
	}

	return result, nil
}
```

# calico的基础知识

主要参考文档：https://docs.tigera.io/calico/latest/networking/determine-best-networking

# calico的运维操作

- 滚动重启

```
kubectl rollout restart daemonset calico-node -n kube-system
```

中途遇到一次如下报错，执行以上重启命令解决：

```
Warning  FailedCreatePodSandBox  5s    kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "679d95d248614136614423cbca20239100b685d43fe7e30d2e5f98d096ca7e57": plugin type="calico" failed (add): error getting ClusterInformation: connection is unauthorized: Unauthorized
```

- 配置文件

/etc/cni/net.d/10-calico.conflist

存在于每台宿主机上。

- 日志

/var/log/calico/cni/cni.log

- cni插件的调试方式（没成功过，因为容器会不断被清理）

```
KUBERNETES_MASTER=https://172.30.22.119:6443 \
CNI_COMMAND=ADD \
CNI_CONTAINERID=8ffcfdd04ca77 \
CNI_NETNS=/proc/$PID/ns/net \
CNI_IFNAME=eth0 \
CNI_PATH=/opt/cni/bin \
/opt/cni/bin/calico < /etc/cni/net.d/10-calico.conflist
```
