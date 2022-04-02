---
title: Network related commands
date: 2021-09-19 22:29:48
categories:
- network
tags:
- command
---
## IP地址和路由配置
```shell
#!/bin/bash
ip link set eth0 up
ip addr add 192.168.221.10/24 dev eth0
ip route add default via 192.168.221.2 dev eth0 onlink
```

## 科学上网配置
```shell
#!/bin/bash
export http_proxy=http://192.168.159.1:7890
export https_proxy=http://192.168.159.1:7890
export no_proxy=localhost,127.0.0.1
```

## opensuse leap 15.3重启网络链接

```shell
#查看配置
sudo nmcli connection show eth0
#重启生效
sudo nmcli connection down eth0
sudo nmcli connection up eth0
```

