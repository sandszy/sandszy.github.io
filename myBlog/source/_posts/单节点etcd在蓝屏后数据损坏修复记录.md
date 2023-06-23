---
title: 单节点etcd在蓝屏后数据损坏修复记录
date: 2023-01-03 22:48:30
tags: kubernetes
---

# 报错信息

```log
# docker logs 10022a842715 查看etcd容器启动日志
[WARNING] Deprecated '--logger=capnslog' flag is set; use '--logger=zap' flag instead
2023-01-02 13:36:53.186216 I | etcdmain: etcd Version: 3.4.13
2023-01-02 13:36:53.186253 I | etcdmain: Git SHA: ae9734ed2
2023-01-02 13:36:53.186256 I | etcdmain: Go Version: go1.12.17
2023-01-02 13:36:53.186260 I | etcdmain: Go OS/Arch: linux/amd64
2023-01-02 13:36:53.186264 I | etcdmain: setting maximum number of CPUs to 4, total number of available CPUs is 4
2023-01-02 13:36:53.186325 N | etcdmain: the server is already initialized as member before, starting as etcd member...
[WARNING] Deprecated '--logger=capnslog' flag is set; use '--logger=zap' flag instead
2023-01-02 13:36:53.186360 I | embed: peerTLS: cert = /etc/kubernetes/pki/etcd/peer.crt, key = /etc/kubernetes/pki/etcd/peer.key, trusted-ca = /etc/kubernetes/pki/etcd/ca.crt, client-cert-auth = true, crl-file = 
2023-01-02 13:36:53.186725 I | embed: name = localhost.localdomain
2023-01-02 13:36:53.186742 I | embed: data dir = /var/lib/etcd
2023-01-02 13:36:53.186744 I | embed: member dir = /var/lib/etcd/member
2023-01-02 13:36:53.186746 I | embed: heartbeat = 100ms
2023-01-02 13:36:53.186748 I | embed: election = 1000ms
2023-01-02 13:36:53.186749 I | embed: snapshot count = 10000
2023-01-02 13:36:53.186752 I | embed: advertise client URLs = https://192.168.221.128:2379
2023-01-02 13:36:53.186754 I | embed: initial advertise peer URLs = https://192.168.221.128:2380
2023-01-02 13:36:53.186757 I | embed: initial cluster = 
2023-01-02 13:36:53.658826 I | etcdserver: recovered store from snapshot at index 3620377
2023-01-02 13:36:53.681707 C | etcdserver: recovering backend from snapshot error: failed to find database snapshot file (snap: snapshot file doesn't exist)
panic: recovering backend from snapshot error: failed to find database snapshot file (snap: snapshot file doesn't exist)
	panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x20 pc=0xc0587e]

goroutine 1 [running]:
go.etcd.io/etcd/etcdserver.NewServer.func1(0xc0002d4e30, 0xc0002d2d80)
	/tmp/etcd-release-3.4.13/etcd/release/etcd/etcdserver/server.go:334 +0x3e
panic(0xee6840, 0xc00012c290)
	/usr/local/go/src/runtime/panic.go:522 +0x1b5
github.com/coreos/pkg/capnslog.(*PackageLogger).Panicf(0xc000123b80, 0x10c13b2, 0x2a, 0xc0002d2e50, 0x1, 0x1)
	/home/ANT.AMAZON.COM/leegyuho/go/pkg/mod/github.com/coreos/pkg@v0.0.0-20160727233714-3ac0863d7acf/capnslog/pkg_logger.go:75 +0x135
go.etcd.io/etcd/etcdserver.NewServer(0x7fff4837ce2f, 0x15, 0x0, 0x0, 0x0, 0x0, 0xc000242c00, 0x1, 0x1, 0xc000242d80, ...)
	/tmp/etcd-release-3.4.13/etcd/release/etcd/etcdserver/server.go:464 +0x433c
go.etcd.io/etcd/embed.StartEtcd(0xc000290000, 0xc0002a2000, 0x0, 0x0)
	/tmp/etcd-release-3.4.13/etcd/release/etcd/embed/etcd.go:214 +0x988
go.etcd.io/etcd/etcdmain.startEtcd(0xc000290000, 0x10963d6, 0x6, 0x1, 0xc0001e9140)
	/tmp/etcd-release-3.4.13/etcd/release/etcd/etcdmain/etcd.go:302 +0x40
go.etcd.io/etcd/etcdmain.startEtcdOrProxyV2()
	/tmp/etcd-release-3.4.13/etcd/release/etcd/etcdmain/etcd.go:144 +0x2ef9
go.etcd.io/etcd/etcdmain.Main()
	/tmp/etcd-release-3.4.13/etcd/release/etcd/etcdmain/main.go:46 +0x38
main.main()
	/tmp/etcd-release-3.4.13/etcd/release/etcd/main.go:28 +0x20
```

# 参考文档

etcd社区有相同问题issue：https://github.com/etcd-io/etcd/issues/11949。

# 解决过程

在issue的最后由**[MrMYHuang](https://github.com/MrMYHuang)**提出了相关workaround如下：

```
sudo rm /var/lib/etcd/member/wal/*.wal
```

or

```
sudo rm /var/lib/etcd/member/snap/*.snap
```

经过实测，需要两者同时删除后重启etcd启动成功

```shell
rm /var/lib/etcd/member/wal/*.wal /var/lib/etcd/member/snap/*.snap
```

etdd启动成功日志如下：

```log
localhost:/var/lib/etcd/member/snap # docker logs 7d38d854b7ef
[WARNING] Deprecated '--logger=capnslog' flag is set; use '--logger=zap' flag instead
2023-01-03 14:41:56.781722 I | etcdmain: etcd Version: 3.4.13
2023-01-03 14:41:56.781776 I | etcdmain: Git SHA: ae9734ed2
2023-01-03 14:41:56.781778 I | etcdmain: Go Version: go1.12.17
2023-01-03 14:41:56.781781 I | etcdmain: Go OS/Arch: linux/amd64
2023-01-03 14:41:56.781783 I | etcdmain: setting maximum number of CPUs to 4, total number of available CPUs is 4
2023-01-03 14:41:56.781853 N | etcdmain: the server is already initialized as member before, starting as etcd member...
[WARNING] Deprecated '--logger=capnslog' flag is set; use '--logger=zap' flag instead
2023-01-03 14:41:56.781875 I | embed: peerTLS: cert = /etc/kubernetes/pki/etcd/peer.crt, key = /etc/kubernetes/pki/etcd/peer.key, trusted-ca = /etc/kubernetes/pki/etcd/ca.crt, client-cert-auth = true, crl-file = 
2023-01-03 14:41:56.782345 I | embed: name = localhost.localdomain
2023-01-03 14:41:56.782350 I | embed: data dir = /var/lib/etcd
2023-01-03 14:41:56.782353 I | embed: member dir = /var/lib/etcd/member
2023-01-03 14:41:56.782354 I | embed: heartbeat = 100ms
2023-01-03 14:41:56.782356 I | embed: election = 1000ms
2023-01-03 14:41:56.782357 I | embed: snapshot count = 10000
2023-01-03 14:41:56.782364 I | embed: advertise client URLs = https://192.168.221.128:2379
2023-01-03 14:41:56.782366 I | embed: initial advertise peer URLs = https://192.168.221.128:2380
2023-01-03 14:41:56.782370 I | embed: initial cluster = 
2023-01-03 14:41:56.783998 I | etcdserver: restarting member 2906be03e76930a4 in cluster 106db65009dfc481 at commit index 4
raft2023/01/03 14:41:56 INFO: 2906be03e76930a4 switched to configuration voters=()
raft2023/01/03 14:41:56 INFO: 2906be03e76930a4 became follower at term 2
raft2023/01/03 14:41:56 INFO: newRaft 2906be03e76930a4 [peers: [], term: 2, commit: 4, applied: 0, lastindex: 4, lastterm: 2]
2023-01-03 14:41:56.785492 W | auth: simple token is not cryptographically signed
2023-01-03 14:41:56.788104 I | etcdserver: starting server... [version: 3.4.13, cluster version: to_be_decided]
2023-01-03 14:41:56.790247 I | embed: ClientTLS: cert = /etc/kubernetes/pki/etcd/server.crt, key = /etc/kubernetes/pki/etcd/server.key, trusted-ca = /etc/kubernetes/pki/etcd/ca.crt, client-cert-auth = true, crl-file = 
2023-01-03 14:41:56.790407 I | embed: listening for metrics on http://127.0.0.1:2381
2023-01-03 14:41:56.790437 I | embed: listening for peers on 192.168.221.128:2380
raft2023/01/03 14:41:56 INFO: 2906be03e76930a4 switched to configuration voters=(2956259129391919268)
2023-01-03 14:41:56.790634 I | etcdserver/membership: added member 2906be03e76930a4 [https://192.168.221.128:2380] to cluster 106db65009dfc481
2023-01-03 14:41:56.790718 N | etcdserver/membership: set the initial cluster version to 3.4
2023-01-03 14:41:56.790748 I | etcdserver/api: enabled capabilities for version 3.4
raft2023/01/03 14:41:58 INFO: 2906be03e76930a4 is starting a new election at term 2
raft2023/01/03 14:41:58 INFO: 2906be03e76930a4 became candidate at term 3
raft2023/01/03 14:41:58 INFO: 2906be03e76930a4 received MsgVoteResp from 2906be03e76930a4 at term 3
raft2023/01/03 14:41:58 INFO: 2906be03e76930a4 became leader at term 3
raft2023/01/03 14:41:58 INFO: raft.node: 2906be03e76930a4 elected leader 2906be03e76930a4 at term 3
2023-01-03 14:41:58.286120 I | etcdserver: published {Name:localhost.localdomain ClientURLs:[https://192.168.221.128:2379]} to cluster 106db65009dfc481
2023-01-03 14:41:58.286163 I | embed: ready to serve client requests
2023-01-03 14:41:58.286221 I | embed: ready to serve client requests
2023-01-03 14:41:58.287902 I | embed: serving client requests on 127.0.0.1:2379
2023-01-03 14:41:58.288457 I | embed: serving client requests on 192.168.221.128:2379
2023-01-03 14:42:07.501668 I | etcdserver/api/etcdhttp: /health OK (status code 200)
```

