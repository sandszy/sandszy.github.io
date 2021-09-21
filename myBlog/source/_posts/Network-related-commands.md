---
title: Network related commands
date: 2021-09-19 22:29:48
tags:
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

