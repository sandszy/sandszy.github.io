---
title: hadoop环境搭建和验证记录
date: 2022-01-26 14:47:54
tags:
---

# hadoop环境搭建和验证记录
## 作业1:本地部署 Hadoop 环境
### step1:准备基础运行环境
操作系统：windows 11
虚拟化软件：vmware workstation 16
虚拟机1操作系统：opensuse leap 15.3
虚拟机2操作系统：opensuse leap 15.3

### step2:在虚拟机1和2上部署docker容器运行环境

```shell
#采用suse的包管理工具zypper命令部署
zypper in docker
```

### step3:在虚拟机1和2上部署kubernetes容器运行环境

使用kubeadm参考如下文档部署k8s环境，k8s环境配置较为复杂，使用kubeadm工具可以大幅度简化，具体步骤非重点不赘述。

使用k8s的原因为可以利用k8s生态已经封装好的hadoop部署API，一键式搭建hadoop环境。

https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

k8s版本为1.21.2

```shell
localhost:~ # kubectl version
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.2", GitCommit:"092fbfbf53427de67cac1e9fa54aaa09a28371d7", GitTreeState:"clean", BuildDate:"2021-06-16T12:59:11Z", GoVersion:"go1.16.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.2", GitCommit:"092fbfbf53427de67cac1e9fa54aaa09a28371d7", GitTreeState:"clean", BuildDate:"2021-06-16T12:53:14Z", GoVersion:"go1.16.5", Compiler:"gc", Platform:"linux/amd64"}
```

### step4:采用k8s operator部署hadoop环境

采用项目：https://github.com/alicek106/k8s-hadoop-operator

```shell
#部署步骤：
git clone https://github.com/alicek106/k8s-hadoop-operator.git
cd k8s-hadoop-operator/temporary-gopath/src/hadoop-operator/
kubectl apply -f deploy/
kubectl apply -f deploy/crds/alicek106_v1alpha1_hadoopservice_cr.yaml
```
### step5:验证hadoop环境

- 登录master节点验证

```shell
#验证步骤
localhost:~ # kubectl get pods
NAME                                                              READY   STATUS    RESTARTS   AGE
example-hadoopservice-hadoop-master-0                             1/1     Running   0          17h
example-hadoopservice-hadoop-slave-0                              1/1     Running   0          17h
example-hadoopservice-hadoop-slave-1                              1/1     Running   0          17h
example-hadoopservice-hadoop-slave-2                              1/1     Running   0          17h
hadoop-operator-7467d5d8f4-vhxhm                                  1/1     Running   0          17h
localhost:~ # kubectl get hds
NAME                    AGE
example-hadoopservice   17h
#获取登录密码
localhost:~ # kubectl logs hadoop-operator-7467d5d8f4-vhxhm|grep pass
{"level":"info","ts":1646403081.5452356,"logger":"controller_hadoopservice","msg":"Generated password is MFIN"}
#获取登录地址
localhost:~ # kubectl get svc|grep hadoop
example-hadoopservice-hadoop-master-svc                ClusterIP      None            <none>        <none>                                        17h
example-hadoopservice-hadoop-master-svc-external       NodePort       10.110.162.31   <none>        22:31242/TCP,8088:31715/TCP,50070:32525/TCP   17h
example-hadoopservice-hadoop-slave-svc                 ClusterIP      None            <none>        22/TCP                                        17h
hadoop-operator                                        ClusterIP      10.111.104.64   <none>        8383/TCP                                      17h
#登录hadoop master节点：ssh://root:***@192.168.221.128:31242
root@example-hadoopservice-hadoop-master-0:~# hdfs dfsadmin -report
Configured Capacity: 64411926528 (59.99 GB)
Present Capacity: 41890959360 (39.01 GB)
DFS Remaining: 41890947072 (39.01 GB)
DFS Used: 12288 (12 KB)
DFS Used%: 0.00%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0

-------------------------------------------------
Live datanodes (3):

Name: 10.244.1.167:50010 (example-hadoopservice-hadoop-slave-1.example-hadoopservice-hadoop-slave-svc.default.svc.cluster.local)
Hostname: example-hadoopservice-hadoop-slave-1.example-hadoopservice-hadoop-slave-svc.default.svc.cluster.local
Decommission Status : Normal
Configured Capacity: 21470642176 (20.00 GB)
DFS Used: 4096 (4 KB)
Non DFS Used: 7506989056 (6.99 GB)
DFS Remaining: 13963649024 (13.00 GB)
DFS Used%: 0.00%
DFS Remaining%: 65.04%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Mar 05 07:36:18 UTC 2022


Name: 10.244.1.165:50010 (example-hadoopservice-hadoop-slave-2.example-hadoopservice-hadoop-slave-svc.default.svc.cluster.local)
Hostname: example-hadoopservice-hadoop-slave-2.example-hadoopservice-hadoop-slave-svc.default.svc.cluster.local
Decommission Status : Normal
Configured Capacity: 21470642176 (20.00 GB)
DFS Used: 4096 (4 KB)
Non DFS Used: 7506989056 (6.99 GB)
DFS Remaining: 13963649024 (13.00 GB)
DFS Used%: 0.00%
DFS Remaining%: 65.04%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Mar 05 07:36:18 UTC 2022


Name: 10.244.1.166:50010 (example-hadoopservice-hadoop-slave-0.example-hadoopservice-hadoop-slave-svc.default.svc.cluster.local)
Hostname: example-hadoopservice-hadoop-slave-0.example-hadoopservice-hadoop-slave-svc.default.svc.cluster.local
Decommission Status : Normal
Configured Capacity: 21470642176 (20.00 GB)
DFS Used: 4096 (4 KB)
Non DFS Used: 7506989056 (6.99 GB)
DFS Remaining: 13963649024 (13.00 GB)
DFS Used%: 0.00%
DFS Remaining%: 65.04%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Mar 05 07:36:18 UTC 2022
```
- 打开web页面查看总览信息

![image-20220305153935431](C:\Users\User2\AppData\Roaming\Typora\typora-user-images\image-20220305153935431.png)



## 作业2:使用 Hadoop Shell 以及 Python / Java 对 HDFS 进行文件的上传和下载

### step1:复制测试文件Alice.txt到master节点

```
D:\notebook\浙江大学培训\第九周2.16\大数据作业>scp Alice.txt root@192.168.221.128:/root/homework
The authenticity of host '192.168.221.128 (192.168.221.128)' can't be established.
ECDSA key fingerprint is SHA256:209VnIXh0c4lChAozSLuSDX3tAZfTkeuoKT3GsgG/NM.
Are you sure you want to continue connecting (yes/no/[fingerprint])?

D:\notebook\浙江大学培训\第九周2.16\大数据作业>scp Alice.txt root@192.168.221.128:/root/homework

D:\notebook\浙江大学培训\第九周2.16\大数据作业>scp -oPort=31242 Alice.txt root@192.168.221.128:/root/homework
The authenticity of host '[192.168.221.128]:31242 ([192.168.221.128]:31242)' can't be established.
ECDSA key fingerprint is SHA256:IQib1/sqe4U1SmQ3YoHSjtSeKEnJuI+4g1xzLaNlmLI.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
Please type 'yes', 'no' or the fingerprint:
Warning: Permanently added '[192.168.221.128]:31242' (ECDSA) to the list of known hosts.
root@192.168.221.128's password:
Alice.txt                                                                             100%  141KB   9.9MB/s   00:00
```

### step2:shell在hdfs上创建一个新的文件夹并上传Alice.txt

```shell
root@example-hadoopservice-hadoop-master-0:~/homework# hadoop fs -mkdir /input
root@example-hadoopservice-hadoop-master-0:~/homework# hadoop fs -put /root/homework/Alice.txt /input/
root@example-hadoopservice-hadoop-master-0:~/homework# 
```

### step3:shell验证上传成功

- 命令行查看

```shell
root@example-hadoopservice-hadoop-master-0:~/homework# hadoop fs -ls -R /
drwxr-xr-x   - root supergroup          0 2022-03-05 08:01 /input
-rw-r--r--   2 root supergroup     144737 2022-03-05 08:01 /input/Alice.txt
```

- 通过web前台查看http://192.168.221.128:32525/explorer.html#/input

![image-20220305160418272](C:\Users\User2\AppData\Roaming\Typora\typora-user-images\image-20220305160418272.png)

### step4:shell下载文件

```shell
root@example-hadoopservice-hadoop-master-0:~/homework# hadoop fs -get /input/Alice.txt /root/homework/Alice.txt-download
```

### step5:shell下载文件验证

```shell
root@example-hadoopservice-hadoop-master-0:~/homework# ls
Alice.txt  Alice.txt-download
```

### step6:python上传环境准备

- 创建python虚拟环境

```shell
D:\notebook\浙江大学培训\第九周2.16\大数据作业>virtualenv hadoop
created virtual environment CPython3.9.10.final.0-64 in 981ms
  creator CPython3Windows(dest=D:\notebook\浙江大学培训\第九周2.16\大数据作业\hadoop, clear=False, no_vcs_ignore=False, global=False)
  seeder FromAppData(download=False, pip=bundle, setuptools=bundle, wheel=bundle, via=copy, app_data_dir=C:\Users\User2\AppData\Local\pypa\virtualenv)
    added seed packages: pip==21.3.1, setuptools==60.2.0, wheel==0.37.1
  activators BashActivator,BatchActivator,FishActivator,NushellActivator,PowerShellActivator,PythonActivator
```

- 激活虚拟环境并安装hdfs类库

```
D:\notebook\浙江大学培训\第九周2.16\大数据作业>hadoop\Scripts\activate
(hadoop) D:\notebook\浙江大学培训\第九周2.16\大数据作业>pip install hdfs
WARNING: Retrying (Retry(total=4, connect=None, read=None, redirect=None, status=None)) after connection broken by 'SSLError(SSLEOFError(8, 'EOF occurred in violation of protocol (_ssl.c:1129)'))': /simple/hdfs/
WARNING: Retrying (Retry(total=3, connect=None, read=None, redirect=None, status=None)) after connection broken by 'SSLError(SSLEOFError(8, 'EOF occurred in violation of protocol (_ssl.c:1129)'))': /simple/hdfs/
Collecting hdfs
  Downloading hdfs-2.6.0-py3-none-any.whl (33 kB)
Collecting six>=1.9.0
  Using cached six-1.16.0-py2.py3-none-any.whl (11 kB)
Collecting requests>=2.7.0
  Using cached requests-2.27.1-py2.py3-none-any.whl (63 kB)
Collecting docopt
  Downloading docopt-0.6.2.tar.gz (25 kB)
  Preparing metadata (setup.py) ... done
Collecting charset-normalizer~=2.0.0
  Downloading charset_normalizer-2.0.12-py3-none-any.whl (39 kB)
Collecting certifi>=2017.4.17
  Using cached certifi-2021.10.8-py2.py3-none-any.whl (149 kB)
Collecting urllib3<1.27,>=1.21.1
  Using cached urllib3-1.26.8-py2.py3-none-any.whl (138 kB)
Collecting idna<4,>=2.5
  Using cached idna-3.3-py3-none-any.whl (61 kB)
Building wheels for collected packages: docopt
  Building wheel for docopt (setup.py) ... done
  Created wheel for docopt: filename=docopt-0.6.2-py2.py3-none-any.whl size=13723 sha256=ec26198f487d8a0a3484f2100737696de4b06aebb9fa207491676ec68b6cfe9b
  Stored in directory: c:\users\user2\appdata\local\pip\cache\wheels\70\4a\46\1309fc853b8d395e60bafaf1b6df7845bdd82c95fd59dd8d2b
Successfully built docopt
Installing collected packages: urllib3, idna, charset-normalizer, certifi, six, requests, docopt, hdfs
Successfully installed certifi-2021.10.8 charset-normalizer-2.0.12 docopt-0.6.2 hdfs-2.6.0 idna-3.3 requests-2.27.1 six-1.16.0 urllib3-1.26.8
WARNING: You are using pip version 21.3.1; however, version 22.0.3 is available.
You should consider upgrading via the 'D:\notebook\浙江大学培训\第九周2.16\大数据作业\hadoop\Scripts\python.exe -m pip install --upgrade pip' command.

(hadoop) D:\notebook\浙江大学培训\第九周2.16\大数据作业>
```

### step7:python上传

- 使用vscode打开python虚拟环境所在目录，创建hadoop.py文件，内容如下

```python
from hdfs.client import Client
client = Client("http://192.168.221.128:32525")
client.makedirs("/input_python")
client.upload("/input_python", "/root/homework/Alice.txt")
```
#### 注意事项1
> 此处创建目录或其他write类操作时遇到如下报错：
>
> **HdfsError**: Permission denied: user=dr.who, access=WRITE, inode="/input":root:supergroup:drwxr-xr-x
>
> 原因是hdfs-default.xml中dfs.permissions.enabled为true所致，此处通过 hadoop fs -chmod -R 777 / 规避解决
>
> <property>
>   <name>dfs.permissions.enabled</name>
>   <value>true</value>
>   <description>
>     If "true", enable permission checking in HDFS.
>     If "false", permission checking is turned off,
>     but all other behavior is unchanged.
>     Switching from one parameter value to the other does not change the mode,
>     owner or group of files or directories.
>   </description>
> </property>

#### 注意事项2

> 实际测试中发现，python上传文件时，除了通过node port映射出来的web端口通信外，还会与具体的slave通信，而slave则需要通过k8s集群的cluster ip与之通信。
>
> 而本地windows环境与k8s集群的cluster ip无法联通，故最后通过k8s的master或者node节点部署python运行环境解决

```shell
root@example-hadoopservice-hadoop-master-0:~/homework# wget https://bootstrap.pypa.io/pip/3.4/get-pip.py
--2022-03-05 14:02:46--  https://bootstrap.pypa.io/pip/3.4/get-pip.py
Resolving bootstrap.pypa.io (bootstrap.pypa.io)... 151.101.108.175, 2a04:4e42:11::175
Connecting to bootstrap.pypa.io (bootstrap.pypa.io)|151.101.108.175|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1708647 (1.6M) [text/x-python]
Saving to: 'get-pip.py'

100%[========================================================================================================================================================================>] 1,708,647   3.79MB/s   in 0.4s   

2022-03-05 14:02:48 (3.79 MB/s) - 'get-pip.py' saved [1708647/1708647]

root@example-hadoopservice-hadoop-master-0:~/homework# ls
Alice.txt  Alice.txt-download  get-pip.py  hadoop  hadoop.tar  hadoop.zip
root@example-hadoopservice-hadoop-master-0:~/homework# python3 get-pip.py 
DEPRECATION: Python 3.4 support has been deprecated. pip 19.1 will be the last one supporting it. Please upgrade your Python as Python 3.4 won't be maintained after March 2019 (cf PEP 429).
Collecting pip<19.2
  Downloading https://files.pythonhosted.org/packages/5c/e0/be401c003291b56efc55aeba6a80ab790d3d4cece2778288d65323009420/pip-19.1.1-py2.py3-none-any.whl (1.4MB)
     |################################| 1.4MB 847kB/s 
Collecting setuptools
  Downloading https://files.pythonhosted.org/packages/91/af/18d58ed8a8e7e6b91d71b0367034faf8ea41e1004018811388ed07a7f2d6/setuptools-43.0.0-py2.py3-none-any.whl (583kB)
     |################################| 583kB 4.7MB/s 
Collecting wheel
  Downloading https://files.pythonhosted.org/packages/00/83/b4a77d044e78ad1a45610eb88f745be2fd2c6d658f9798a15e384b7d57c9/wheel-0.33.6-py2.py3-none-any.whl
Installing collected packages: pip, setuptools, wheel
Successfully installed pip-19.1.1 setuptools-43.0.0 wheel-0.33.6
root@example-hadoopservice-hadoop-master-0:~/homework# pip3 install hdfs
DEPRECATION: Python 3.4 support has been deprecated. pip 19.1 will be the last one supporting it. Please upgrade your Python as Python 3.4 won't be maintained after March 2019 (cf PEP 429).
Collecting hdfs
  Downloading https://files.pythonhosted.org/packages/08/f7/4c3fad73123a24d7394b6f40d1ec9c1cbf2e921cfea1797216ffd0a51fb1/hdfs-2.6.0-py3-none-any.whl
Collecting docopt (from hdfs)
  Downloading https://files.pythonhosted.org/packages/a2/55/8f8cab2afd404cf578136ef2cc5dfb50baa1761b68c9da1fb1e4eed343c9/docopt-0.6.2.tar.gz
Collecting requests>=2.7.0 (from hdfs)
  Downloading https://files.pythonhosted.org/packages/7d/e3/20f3d364d6c8e5d2353c72a67778eb189176f08e873c9900e10c0287b84b/requests-2.21.0-py2.py3-none-any.whl (57kB)
     |################################| 61kB 657kB/s 
Collecting six>=1.9.0 (from hdfs)
  Downloading https://files.pythonhosted.org/packages/d9/5a/e7c31adbe875f2abbb91bd84cf2dc52d792b5a01506781dbcf25c91daf11/six-1.16.0-py2.py3-none-any.whl
Collecting chardet<3.1.0,>=3.0.2 (from requests>=2.7.0->hdfs)
  Downloading https://files.pythonhosted.org/packages/bc/a9/01ffebfb562e4274b6487b4bb1ddec7ca55ec7510b22e4c51f14098443b8/chardet-3.0.4-py2.py3-none-any.whl (133kB)
     |################################| 143kB 1.8MB/s 
Collecting idna<2.9,>=2.5 (from requests>=2.7.0->hdfs)
  Downloading https://files.pythonhosted.org/packages/14/2c/cd551d81dbe15200be1cf41cd03869a46fe7226e7450af7a6545bfc474c9/idna-2.8-py2.py3-none-any.whl (58kB)
     |################################| 61kB 1.2MB/s 
Collecting certifi>=2017.4.17 (from requests>=2.7.0->hdfs)
  Downloading https://files.pythonhosted.org/packages/37/45/946c02767aabb873146011e665728b680884cd8fe70dde973c640e45b775/certifi-2021.10.8-py2.py3-none-any.whl (149kB)
     |################################| 153kB 5.8MB/s 
Collecting urllib3<1.25,>=1.21.1 (from requests>=2.7.0->hdfs)
  Downloading https://files.pythonhosted.org/packages/01/11/525b02e4acc0c747de8b6ccdab376331597c569c42ea66ab0a1dbd36eca2/urllib3-1.24.3-py2.py3-none-any.whl (118kB)
     |################################| 122kB 7.6MB/s 
Building wheels for collected packages: docopt
  Building wheel for docopt (setup.py) ... done
  Stored in directory: /root/.cache/pip/wheels/9b/04/dd/7daf4150b6d9b12949298737de9431a324d4b797ffd63f526e
Successfully built docopt
Installing collected packages: docopt, chardet, idna, certifi, urllib3, requests, six, hdfs
Successfully installed certifi-2021.10.8 chardet-3.0.4 docopt-0.6.2 hdfs-2.6.0 idna-2.8 requests-2.21.0 six-1.16.0 urllib3-1.24.3
root@example-hadoopservice-hadoop-master-0:~/homework# python3
Python 3.4.3 (default, Oct 14 2015, 20:28:29) 
[GCC 4.8.4] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from hdfs.client import Client
>>> client = Client("http://192.168.221.128:32525")
>>> client.upload("/input_python", "Alice.txt")
'/input_python/Alice.txt'
>>> client.download("/input_python/Alice.txt","/root/homework/Alice.txt-download-by-python")
'/root/homework/Alice.txt-download-by-python'
```



### step8:python上传验证

通过前台查看http://192.168.221.128:32525/explorer.html#/input_python

![image-20220305221554363](C:\Users\User2\AppData\Roaming\Typora\typora-user-images\image-20220305221554363.png)

### step9:python下载

```python
client.download("/input_python/Alice.txt" "/root/homework/Alice.txt-download-by-python")
```

### step10:python下载验证

```shell
root@example-hadoopservice-hadoop-master-0:~/homework# ls -al
total 2104
drwxr-xr-x 1 root root     130 Mar  5 14:14 .
drwx------ 1 root root     126 Mar  5 13:49 ..
-rw-r--r-- 1 root root  144737 Mar  5 07:55 Alice.txt
-rw-r--r-- 1 root root  144737 Mar  5 08:08 Alice.txt-download
-rw-r--r-- 1 root root  144737 Mar  5 14:10 Alice.txt-download-by-python
-rw-r--r-- 1 root root 1708647 Feb 22  2021 get-pip.py
root@example-hadoopservice-hadoop-master-0:~/homework# 
```

## 作业3:使用 Hadoop 内置的示例程序完成词频统计实验

### step1:使用内置的程序对alice文件进行词频统计

执行命令hadoop jar /hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.0.jar wordcount /input_python/ /output_python/

```shell
root@example-hadoopservice-hadoop-master-0:~/homework# hadoop jar /hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.0.jar wordcount /input_python/ /output_python/
22/03/05 14:41:33 INFO client.RMProxy: Connecting to ResourceManager at example-hadoopservice-hadoop-master-0.example-hadoopservice-hadoop-master-svc.default.svc.cluster.local/10.244.1.168:8032
22/03/05 14:41:34 INFO input.FileInputFormat: Total input paths to process : 1
22/03/05 14:41:34 INFO mapreduce.JobSubmitter: number of splits:1
22/03/05 14:41:34 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1646490810811_0002
22/03/05 14:41:34 INFO impl.YarnClientImpl: Submitted application application_1646490810811_0002
22/03/05 14:41:34 INFO mapreduce.Job: The url to track the job: http://example-hadoopservice-hadoop-master-0.example-hadoopservice-hadoop-master-svc.default.svc.cluster.local:8088/proxy/application_1646490810811_0002/
22/03/05 14:41:34 INFO mapreduce.Job: Running job: job_1646490810811_0002
22/03/05 14:41:38 INFO mapreduce.Job: Job job_1646490810811_0002 running in uber mode : false
22/03/05 14:41:38 INFO mapreduce.Job:  map 0% reduce 0%
22/03/05 14:41:44 INFO mapreduce.Job:  map 100% reduce 0%
22/03/05 14:41:58 INFO mapreduce.Job:  map 100% reduce 67%
22/03/05 14:41:59 INFO mapreduce.Job:  map 100% reduce 100%
22/03/05 14:42:00 INFO mapreduce.Job: Job job_1646490810811_0002 completed successfully
22/03/05 14:42:01 INFO mapreduce.Job: Counters: 49
File System Counters
FILE: Number of bytes read=71649
FILE: Number of bytes written=355413
FILE: Number of read operations=0
FILE: Number of large read operations=0
FILE: Number of write operations=0
HDFS: Number of bytes read=144941
HDFS: Number of bytes written=50811
HDFS: Number of read operations=6
HDFS: Number of large read operations=0
HDFS: Number of write operations=2
Job Counters 
Launched map tasks=1
Launched reduce tasks=1
Data-local map tasks=1
Total time spent by all maps in occupied slots (ms)=3922
Total time spent by all reduces in occupied slots (ms)=11750
Total time spent by all map tasks (ms)=3922
Total time spent by all reduce tasks (ms)=11750
Total vcore-seconds taken by all map tasks=3922
Total vcore-seconds taken by all reduce tasks=11750
Total megabyte-seconds taken by all map tasks=4016128
Total megabyte-seconds taken by all reduce tasks=12032000
Map-Reduce Framework
Map input records=855
Map output records=26500
Map output bytes=248743
Map output materialized bytes=71649
Input split bytes=204
Combine input records=26500
Combine output records=5301
Reduce input groups=5301
Reduce shuffle bytes=71649
Reduce input records=5301
Reduce output records=5301
Spilled Records=10602
Shuffled Maps =1
Failed Shuffles=0
Merged Map outputs=1
GC time elapsed (ms)=154
CPU time spent (ms)=2350
Physical memory (bytes) snapshot=366796800
Virtual memory (bytes) snapshot=3917611008
Total committed heap usage (bytes)=307757056
Shuffle Errors
BAD_ID=0
CONNECTION=0
IO_ERROR=0
WRONG_LENGTH=0
WRONG_MAP=0
WRONG_REDUCE=0
File Input Format Counters 
Bytes Read=144737
File Output Format Counters 
Bytes Written=50811
root@example-hadoopservice-hadoop-master-0:~/homework#
```

#### 注意事项

> 测试过程中发现hadoop的ResourceManager服务突然异常退出，导致无法正常执行mapreduce，报出无法连ResourceManager的错误，通过分析hadoop master节点的启动脚本/entrypoint.sh发现启动命令为/hadoop/sbin/start-all.sh，故重新执行/hadoop/sbin/start-all.sh后，ResourceManager恢复

```shell
root@example-hadoopservice-hadoop-master-0:/# /hadoop/sbin/start-all.sh
This script is Deprecated. Instead use start-dfs.sh and start-yarn.sh
Starting namenodes on [example-hadoopservice-hadoop-master-0.example-hadoopservice-hadoop-master-svc.default.svc.cluster.local]
example-hadoopservice-hadoop-master-0.example-hadoopservice-hadoop-master-svc.default.svc.cluster.local: namenode running as process 297. Stop it first.
example-hadoopservice-hadoop-slave-2.example-hadoopservice-hadoop-slave-svc.default.svc.cluster.local: datanode running as process 41. Stop it first.
example-hadoopservice-hadoop-slave-0.example-hadoopservice-hadoop-slave-svc.default.svc.cluster.local: datanode running as process 42. Stop it first.
example-hadoopservice-hadoop-slave-1.example-hadoopservice-hadoop-slave-svc.default.svc.cluster.local: datanode running as process 42. Stop it first.
Starting secondary namenodes [0.0.0.0]
0.0.0.0: secondarynamenode running as process 482. Stop it first.
starting yarn daemons
starting resourcemanager, logging to /hadoop/logs/yarn-root-resourcemanager-example-hadoopservice-hadoop-master-0.out
example-hadoopservice-hadoop-slave-2.example-hadoopservice-hadoop-slave-svc.default.svc.cluster.local: nodemanager running as process 141. Stop it first.
example-hadoopservice-hadoop-slave-0.example-hadoopservice-hadoop-slave-svc.default.svc.cluster.local: nodemanager running as process 142. Stop it first.
example-hadoopservice-hadoop-slave-1.example-hadoopservice-hadoop-slave-svc.default.svc.cluster.local: nodemanager running as process 142. Stop it first.
root@example-hadoopservice-hadoop-master-0:/#
```

### step2:查看结果

- 执行命令hadoop fs -cat /output_python/*|more

```shell
root@example-hadoopservice-hadoop-master-0:~/homework# hadoop fs -cat /output_python/*|more
"'TIS 1
"--SAID 1
"Come 1
"Coming 1
"Edwin 1
"French, 1
"HOW 1
"He's 1
"How 1
"I 8
"I'll 2
"Keep 1
"Let 1
"Such 1
"THEY 1
"There 2
"There's 1
"Too 1
"Turtle 1
"Twinkle, 1
"Uglification,"' 1
"Up 1
"What 2
"Who 1
"William 1
"With 1
"YOU 1
"You 2
"come 1
"it" 2
"much 1
"poison" 1
"purpose"?' 1
'em 3
'tis 2
(Alice 4
(And, 1
(As 1
(Before 1
(Dinah 1
(For, 1
(He 1
(IF 1
(In 1
(It 1
(Sounds 1
(The 3
(WITH 1
(We 1
(Which 2
(`I 2
```

- 输出结果文件如下：

![image-20220305225525418](C:\Users\User2\AppData\Roaming\Typora\typora-user-images\image-20220305225525418.png)
