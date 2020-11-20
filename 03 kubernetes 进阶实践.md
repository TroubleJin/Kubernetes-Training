- [1.  Etcd常用操作](#1--Etcd常用操作)
  * [1.1 etcdctl拷贝](#11-etcdctl拷贝)
  * [1.2 查看etcd集群的成员节点](#12-查看etcd集群的成员节点)
  * [1.3 查看etcd集群节点状态](#13-查看etcd集群节点状态)
  * [1.4 查看所有key值](#14-查看所有key值)
  * [1.5 定时备份](#15-定时备份)
- [2. Kubernetes调度](#2-kubernetes调度)
  * [2.1 控制Pod应该如何调度](#21-控制Pod应该如何调度)
  * [2.2 调度过程](#22-调度过程)
  * [2.3 Cordon不可调度](#23-Cordon不可调度)
  * [2.4 NodeSelector节点选择器](#24-NodeSelector节点选择器)
  * [2.5 nodeAffinity节点亲和性](#25-nodeAffinity节点亲和性)
  * [2.6 污点（Taints）与容忍（tolerations）](#26-污点与容忍)
- [3. Kubernetes网络实现](#3-Kubernetes网络实现)
  * [3.1 CNI介绍及集群网络选型](#31-CNI介绍及集群网络选型)
  * [3.2 Flannel网络模型实现剖析](#32-Flannel网络模型实现剖析)
  * [3.3 vxlan介绍及点对点通信的实现](#33-vxlan介绍及点对点通信的实现)
  * [3.4 跨主机容器网络的通信](#34-跨主机容器网络的通信)
  * [3.5 Flannel的vxlan实现](#35-Flannel的vxlan实现)
  * [3.6 跨主机Pod通信的流量详细过程](#36-跨主机Pod通信的流量详细过程)
  * [3.7 利用host-gw模式提升集群网络性能](#37-利用host-gw模式提升集群网络性能)
- [4. Kubernetes认证授权](#4-Kubernetes认证授权)
  * [4.1 APIServer安全控制](#41-APIServer安全控制)
  * [4.2 kubectl的认证授权](#42-kubectl的认证授权)
  * [4.3 RBAC](#43-rbac)
  * [4.4 kubelet的认证授权](#44-kubelet的认证授权)
  * [4.5 ServiceAccount及K8SApi调用](#45-ServiceAccount及K8SApi调用)
  * [4.6 创建用户认证授权的kubeconfig文件](#46-创建用户认证授权的kubeconfig文件)
- [5. 通过HPA实现业务应用的动态扩缩容](#5-通过HPA实现业务应用的动态扩缩容)
  * [5.1 HPA控制器介绍](#51-HPA控制器介绍)
  * [5.2 Metric-Server](#52-Metric-Server)
  * [5.3 安装](#53-安装)
  * [5.4 kubelet的指标采集](#54-kubelet的指标采集)
  * [5.5 kube-aggregator聚合器及Metric-Server的实现](#55-kube-aggregator聚合器及Metric-Server的实现)
  * [5.6 HPA实践](#56-HPA实践)
  * [5.7 基于自定义指标的动态伸缩](#57-基于自定义指标的动态伸缩)
- [6. kubernetes对接分部式存储](#6-kubernetes对接分部式存储)
  * [6.1 emptydir](#61-emptydir)
  * [6.2 hostPath](#62-hostpath)
  * [6.3 PV与PVC管理NFS存储卷实践](#63-PV与PVC管理NFS存储卷实践)
  * [6.4 storageClass实现动态挂载](#64-storageClass实现动态挂载)
- [7.  使用Helm3的部署](#7-使用Helm3的部署)
  * [7.1 认识Helm](#71-认识Helm)
  * [7.2 安装与快速入门实践](#72-安装与快速入门实践)
  * [7.3 Helm基本使用](#73-Helm基本使用)
  * [7.4 Helmrepo更改](#74-Helmrepo更改)
  * [7.5 Helm模板内置函数和Value](#75-Helm模板内置函数和Value)
    + [7.5.1 Helm 创建模板](#751-Helm创建模板)
    + [7.5.2 添加简单的模板](#752-添加简单的模板)
    + [7.5.3 内置对象](#753-内置对象)
    + [7.5.4 Helm 模板之模板函数与管道](#754-Helm模板之模板函数与管道)
    + [7.5.5 Helm模板之控制流程](#755-Helm模板之控制流程)
    + [7.5.6 使用方法](#756-使用方法)
#   1.  Etcd常用操作
##  1.1 etcdctl拷贝
```
[root@k8s-master ~]# kubectl -n kube-system exec etcd-k8s-master  which etcdctl
/usr/local/bin/etcdctl
[root@k8s-master ~]# kubectl cp -n kube-system etcd-k8s-master:/usr/local/bin/etcdctl  /usr/local/bin/etcdctl
tar: Removing leading `/' from member names
[root@k8s-master ~]# chmod +x /usr/local/bin/etcdctl 
```

##  1.2 查看etcd集群的成员节点
```
[root@k8s-master ~]# export ETCDCTL_API=3
[root@k8s-master ~]# etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key member list -w table
+------------------+---------+------------+----------------------------+----------------------------+
|        ID        | STATUS  |    NAME    |         PEER ADDRS         |        CLIENT ADDRS        |
+------------------+---------+------------+----------------------------+----------------------------+
| 802d87cb431bcba5 | started | k8s-master | https://172.17.176.31:2380 | https://172.17.176.31:2379 |
+------------------+---------+------------+----------------------------+----------------------------+
[root@k8s-master ~]# alias etcdctl='etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key'
[root@k8s-master ~]# etcdctl member list -w table
+------------------+---------+------------+----------------------------+----------------------------+
|        ID        | STATUS  |    NAME    |         PEER ADDRS         |        CLIENT ADDRS        |
+------------------+---------+------------+----------------------------+----------------------------+
| 802d87cb431bcba5 | started | k8s-master | https://172.17.176.31:2380 | https://172.17.176.31:2379 |
+------------------+---------+------------+----------------------------+----------------------------+

```

##  1.3 查看etcd集群节点状态
```
[root@k8s-master ~]# etcdctl endpoint status -w table
+--------------------------+------------------+---------+---------+-----------+-----------+------------+
|         ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+--------------------------+------------------+---------+---------+-----------+-----------+------------+
| https://[127.0.0.1]:2379 |etcdctl get / --prefix --keys-only 802d87cb431bcba5 |  3.3.15 |  2.3 MB |      true |         2 |    1529335 |
+--------------------------+------------------+---------+---------+-----------+-----------+------------+
[root@k8s-master ~]# etcdctl endpoint health -w table
+--------------------------+--------+------------+-------+
|         ENDPOINT         | HEALTH |    TOOK    | ERROR |
+--------------------------+--------+------------+-------+
| https://[127.0.0.1]:2379 |   true | 7.656205ms |       |
+--------------------------+--------+------------+-------+
```

##  1.4 查看所有key值
```
etcdctl get / --prefix --keys-only
```

##  1.5 定时备份
```
etcdctl snapshot save `hostname`-etcd_`date +%Y%m%d%H%M`.db
```
恢复快照：

1. 停止etcd和apiserver

2. 移走当前数据目录

   ```powershell
   $ mv /var/lib/etcd/ /tmp
   ```

3. 恢复快照

   ```powershell
   $ etcdctl snapshot restore `hostname`-etcd_`date +%Y%m%d%H%M`.db --data-dir=/var/lib/etcd/
   
   ```

#   2. Kubernetes调度
##  2.1 控制Pod应该如何调度
![image](./img/day03-1.jpg)
Kubernetes Scheduler 的作用是将待调度的 Pod 按照一定的调度算法和策略绑定到集群中一个合适的 Worker Node 上，并将绑定信息写入到 etcd 中，之后目标 Node 中 kubelet 服务通过 API Server 监听到 Scheduler 产生的 Pod 绑定事件获取 Pod 信息，然后下载镜像启动容器。

##  2.2 调度过程
Scheduler 提供的调度流程分为预选 (Predicates) 和优选 (Priorities) 两个步骤：

- 预选，K8S会遍历当前集群中的所有 Node，筛选出其中符合要求的 Node 作为候选
- 优选，K8S将对候选的 Node 进行打分

经过预选筛选和优选打分之后，K8S选择分数最高的 Node 来运行 Pod，如果最终有多个 Node 的分数最高，那么 Scheduler 将从当中随机选择一个 Node 来运行 Pod。

## 2.3 Cordon不可调度
-   本质也是使用了污点和容忍
```
kubectl cordon k8s-slave2
kubectl drain k8s-slave2
kubectl drain k8s-slave2 --delete-local-data=true --ignore-daemonsets=true  --force
kubectl uncordon  k8s-slave2
```
## 2.4 NodeSelector节点选择器

 `label`是`kubernetes`中一个非常重要的概念，用户可以非常灵活的利用 label 来管理集群中的资源，POD 的调度可以根据节点的 label 进行特定的部署。 

查看节点的label：

```powershell
$ kubectl get nodes --show-labels
```

为节点打label：

```powershell
$ kubectl label node k8s-master disktype=ssd
```

##  2.5 nodeAffinity节点亲和性
节点亲和性 ， 比上面的`nodeSelector`更加灵活，它可以进行一些简单的逻辑组合，不只是简单的相等匹配 。分为两种，硬策略和软策略。

requiredDuringSchedulingIgnoredDuringExecution ： 硬策略，如果没有满足条件的节点的话，就不断重试直到满足条件为止，简单说就是你必须满足我的要求，不然我就不会调度Pod。

preferredDuringSchedulingIgnoredDuringExecution：软策略，如果你没有满足调度要求的节点的话，Pod就会忽略这条规则，继续完成调度过程，说白了就是满足条件最好了，没有满足就忽略掉的策略。

```yaml
#要求 Pod 不能运行在128和132两个节点上，如果有节点满足disktype=ssd或者sas的话就优先调度到这类节点上
...
spec:
      containers:
      - name: demo
        image: 192.168.136.10:5000/demo/myblog:v1
        ports:
        - containerPort: 8002
      affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                    - key: kubernetes.io/hostname
                      operator: NotIn
                      values:
                        - 172.21.51.698
                        - 192.168.136.132
                        
            preferredDuringSchedulingIgnoredDuringExecution:
                - weight: 1
                  preference:
                    matchExpressions:
                    - key: disktype
                      operator: In
                      values:
                        - ssd
                        - sas
...
```

这里的匹配逻辑是 label 的值在某个列表中，现在`Kubernetes`提供的操作符有下面的几种：

- In：label 的值在某个列表中
- NotIn：label 的值不在某个列表中
- Gt：label 的值大于某个值
- Lt：label 的值小于某个值
- Exists：某个 label 存在
- DoesNotExist：某个 label 不存在

**如果nodeSelectorTerms下面有多个选项的话，满足任何一个条件就可以了；如果matchExpressions有多个选项的话，则必须同时满足这些条件才能正常调度 Pod**

##  2.6 污点与容忍
对于`nodeAffinity`无论是硬策略还是软策略方式，都是调度 Pod 到预期节点上，而`Taints`恰好与之相反，如果一个节点标记为 Taints ，除非 Pod 也被标识为可以容忍污点节点，否则该 Taints 节点不会被调度Pod。

Taints(污点)是Node的一个属性，设置了Taints(污点)后，因为有了污点，所以Kubernetes是不会将Pod调度到这个Node上的。于是Kubernetes就给Pod设置了个属性Tolerations(容忍)，只要Pod能够容忍Node上的污点，那么Kubernetes就会忽略Node上的污点，就能够(不是必须)把Pod调度过去。

场景一：私有云服务中，某业务使用GPU进行大规模并行计算。为保证性能，希望确保该业务对服务器的专属性，避免将普通业务调度到部署GPU的服务器。

场景二：用户希望把 Master 节点保留给 Kubernetes 系统组件使用，或者把一组具有特殊资源预留给某些 Pod，则污点就很有用了，Pod 不会再被调度到 taint 标记过的节点。taint 标记节点举例如下：

设置污点：
```powershell
$ kubectl taint node [node_name] key=value:[effect]   
      其中[effect] 可取值： [ NoSchedule | PreferNoSchedule | NoExecute ]
       NoSchedule：一定不能被调度。
       PreferNoSchedule：尽量不要调度。
       NoExecute：不仅不会调度，还会驱逐Node上已有的Pod。
  示例：kubectl taint node k8s-slave1 smoke=true:NoSchedule
```

**污点驱逐**
```
[root@k8s-master ~]# kubectl taint node k8s-slave-2 smoke=true:NoSchedule
node/k8s-slave-2 tainted
[root@k8s-master ~]# kubectl -n luffy get pod  -o wide 
NAME                      READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
myblog-84f966749f-64hnb   1/1     Running   0          4h1m    10.244.1.15   k8s-slave-1   <none>           <none>
myblog-84f966749f-8rs5p   1/1     Running   0          2m21s   10.244.1.18   k8s-slave-1   <none>           <none>
myblog-84f966749f-mlsmc   1/1     Running   0          4h1m    10.244.0.9    k8s-master    <none>           <none>
myblog-84f966749f-pgshm   1/1     Running   0          2m21s   10.244.0.10   k8s-master    <none>           <none>
myblog-84f966749f-qfhg7   1/1     Running   0          24h     10.244.1.14   k8s-slave-1   <none>           <none>
mysql-778f489b9-qhbqv     1/1     Running   0          7d      10.244.1.12   k8s-slave-1   <none>           <none>
```
**污点容忍**
-   容忍是或者的关系
```
...
spec:
      containers:
      - name: demo
        image: 192.168.136.10:5000/demo/myblog:v1
      tolerations: #设置容忍性
      - key: "smoke" 
        operator: "Equal"  #如果操作符为Exists，那么value属性可省略,不指定operator，默认为Equal
        value: "true"
        effect: "NoSchedule"
      - key: "drunk" 
        operator: "Exists"  #如果操作符为Exists，那么value属性可省略,不指定operator，默认为Equal
	  #意思是这个Pod要容忍的有污点的Node的key是smoke Equal true,效果是NoSchedule，
      #tolerations属性下各值必须使用引号，容忍的值都是设置Node的taints时给的值。
```

#   3. Kubernetes网络实现
##  3.1 CNI介绍及集群网络选型
容器网络接口（Container Network Interface），实现kubernetes集群的Pod网络通信及管理。包括：

- CNI Plugin负责给容器配置网络，它包括两个基本的接口：
  配置网络: AddNetwork(net NetworkConfig, rt RuntimeConf) (types.Result, error)
  清理网络: DelNetwork(net NetworkConfig, rt RuntimeConf) error
- IPAM Plugin负责给容器分配IP地址，主要实现包括host-local和dhcp。

以上两种插件的支持，使得k8s的网络可以支持各式各样的管理模式，当前在业界也出现了大量的支持方案，其中比较流行的比如flannel、calico等。
kubernetes配置了cni网络插件后，其容器网络创建流程为：

- kubelet先创建pause容器生成对应的network namespace
- 调用网络driver，因为配置的是CNI，所以会调用CNI相关代码，识别CNI的配置目录为/etc/cni/net.d
- CNI driver根据配置调用具体的CNI插件，二进制调用，可执行文件目录为/opt/cni/bin,[项目](https://github.com/containernetworking/plugins)
- CNI插件给pause容器配置正确的网络，pod中其他的容器都是用pause的网络

 可以在此查看社区中的CNI实现，https://github.com/containernetworking/cni 

## 3.2 Flannel网络模型实现剖析
-   https://github.com/coreos/flannel/blob/master/Documentation/backends.md

flannel实现overlay，underlay网络通常有多种实现：
- udp
- vxlan
- host-gw
- ...

不特殊指定的话，默认会使用vxlan技术作为Backend，可以通过如下查看：

```
$ kubectl -n kube-system exec  kube-flannel-ds-amd64-cb7hs cat /etc/kube-flannel/net-conf.json
{
  "Network": "10.244.0.0/16",
  "Backend": {
    "Type": "vxlan"
  }
}
#	阿里云使用的alloc
Defaulting container name to kube-flannel.
Use 'kubectl describe pod/kube-flannel-ds-tzpbj -n kube-system' to see all of the containers in this pod.
/ # cat /etc/kube-flannel/net-conf.json
{
  "Network": "172.20.0.0/16",
  "Backend": {
    "Type": "alloc"
  }
}
```

## 3.3 vxlan介绍及点对点通信的实现

![image](./img/day03-2.png)


VXLAN 全称是虚拟可扩展的局域网（ Virtual eXtensible Local Area Network），它是一种 overlay 技术，通过三层的网络来搭建虚拟的二层网络。

它创建在原来的 IP 网络（三层）上，只要是三层可达（能够通过 IP 互相通信）的网络就能部署 vxlan。在每个端点上都有一个 vtep 负责 vxlan 协议报文的封包和解包，也就是在虚拟报文上封装 vtep 通信的报文头部。物理网络上可以创建多个 vxlan 网络，这些 vxlan 网络可以认为是一个隧道，不同节点的虚拟机能够通过隧道直连。每个 vxlan 网络由唯一的 VNI 标识，不同的 vxlan 可以不相互影响。 

- VTEP（VXLAN Tunnel Endpoints）：vxlan 网络的边缘设备，用来进行 vxlan 报文的处理（封包和解包）。vtep 可以是网络设备（比如交换机），也可以是一台机器（比如虚拟化集群中的宿主机）
- VNI（VXLAN Network Identifier）：VNI 是每个 vxlan 的标识，一共有 2^24 = 16,777,216，一般每个 VNI 对应一个租户，也就是说使用 vxlan 搭建的公有云可以理论上可以支撑千万级别的租户

演示：在k8s-slave1和k8s-slave2两台机器间，利用vxlan的点对点能力，实现虚拟二层网络的通信

**k8s-slave1节点**
```
[root@k8s-slave-1 ~]# ip link add vxlan20 type vxlan id 20 remote 172.17.176.33 dstport 4789 dev eth0
[root@k8s-slave-1 ~]# ip -d link show vxlan20
26: vxlan20: <BROADCAST,MULTICAST> mtu 1450 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 26:23:d2:42:0e:41 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    vxlan id 20 remote 172.17.176.33 dev eth0 srcport 0 0 dstport 4789 ageing 300 noudpcsum noudp6zerocsumtx noudp6zerocsumrx addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
[root@k8s-slave-1 ~]# ip link set vxlan20 up 
[root@k8s-slave-1 ~]# ip addr add 10.0.136.11/24 dev vxlan20
```
**k8s-slave2节点**
```
[root@k8s-slave-2 ~]# ip link add vxlan20 type vxlan id 20 remote 172.17.176.32 dstport 4789 dev eth0
[root@k8s-slave-2 ~]# ip link set vxlan20 up 
[root@k8s-slave-2 ~]# ip addr add 10.0.136.12/24 dev vxlan20
```
-   验证
```
[root@k8s-slave-1 ~]# ping 10.0.136.12 -c 1 -t1
PING 10.0.136.12 (10.0.136.12) 56(84) bytes of data.
64 bytes from 10.0.136.12: icmp_seq=1 ttl=64 time=0.292 ms
[root@k8s-slave-1 ~]# route -n |grep 10.0.136.0
10.0.136.0      0.0.0.0         255.255.255.0   U     0      0        0 vxlan20
[root@k8s-slave-1 ~]# ip -d link show vxlan20
26: vxlan20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 26:23:d2:42:0e:41 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    vxlan id 20 remote 172.17.176.33 dev eth0 srcport 0 0 dstport 4789 ageing 300 noudpcsum noudp6zerocsumtx noudp6zerocsumrx addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
[root@k8s-slave-1 ~]# bridge fdb show|grep vxlan20
00:00:00:00:00:00 dev vxlan20 dst 172.17.176.33 via eth0 self permanent
```
![image](./img/day03-3.png)

隧道是一个逻辑上的概念，在 vxlan 模型中并没有具体的物理实体想对应。隧道可以看做是一种虚拟通道，vxlan 通信双方（图中的虚拟机）认为自己是在直接通信，并不知道底层网络的存在。从整体来说，每个 vxlan 网络像是为通信的虚拟机搭建了一个单独的通信通道，也就是隧道。

实现的过程：

虚拟机的报文通过 vtep 添加上 vxlan 以及外部的报文层，然后发送出去，对方 vtep 收到之后拆除 vxlan 头部然后根据 VNI 把原始报文发送到目的虚拟机。 


##  3.4 跨主机容器网络的通信
-   原先Docker网络结构
![image](./img/day03-4.png)
-   容器的流量通过vtep设备进行转发
![image](./img/day03-5.png)
**k8s-slave-1操作**
```
[root@k8s-slave-1 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
d79b495d6c95        bridge              bridge              local
d99d2bf230a6        host                host                local
2b3a0069c0f2        none                null                local
# 创建新网桥，指定cidr段
[root@k8s-slave-1 ~]# docker network create --subnet 172.20.0.0/16  network-luffy
9ff93a9f1543c7fc0d93958aa2074283ad5ff59f51a9a5830a1a1b1e5133c614
[root@k8s-slave-1 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
d79b495d6c95        bridge              bridge              local
d99d2bf230a6        host                host                local
9ff93a9f1543        network-luffy       bridge              local
2b3a0069c0f2        none                null                local
# 新建容器，接入到新网桥
[root@k8s-slave-1 ~]# docker run -d --name vxlan-test --net network-luffy --ip 172.20.0.2 nginx:alpine
ad0418c1b1780ab867741a509d1ff8a23722dea65f5f716aa8dd2d3777d8bc7c
[root@k8s-slave-1 ~]# docker exec vxlan-test ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:14:00:02  
          inet addr:172.20.0.2  Bcast:172.20.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
[root@k8s-slave-1 ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
br-9ff93a9f1543		8000.0242f7e4fdae	no		veth5e09997
cni0		8000.72f30299e263	no		veth87715ed8
							vethac6391dd
							vethb38cbac8
							vethcd980518
							vethd44f754b
							vethf1bc1704
docker0		8000.0242405b9cda	no		
```
**k8s-slave-2**
```
[root@k8s-slave-2 ~]# docker network create --subnet 172.20.0.0/16  network-luffy
574629858e91a02ea078f5b70b7278b74dd581c7b41b2479befa2f5ed136b378
[root@k8s-slave-2 ~]# docker run -d --name vxlan-test --net network-luffy --ip 172.20.0.3 nginx:alpine
Unable to find image 'nginx:alpine' locally
alpine: Pulling from library/nginx
df20fa9351a1: Pull complete 
091a6e3499e9: Pull complete 
b4bea01b9731: Pull complete 
62c992d61d2c: Pull complete 
b675ffa804eb: Pull complete 
Digest: sha256:5fcbe9a6b09b6525651d1e5d5a2df373eec1a13c75f0eaa724a369f43ce589f4
Status: Downloaded newer image for nginx:alpine
02dbb7a395fa091828ce8f470c023776b8f50489ccc6e5c7bf7cdb54e19d9291
[root@k8s-slave-2 ~]# docker exec vxlan-test ping 172.20.0.2
PING 172.20.0.2 (172.18.0.3): 56 data bytes
```
分析：数据到了网桥后，出不去。结合前面的示例，因此应该将流量由vtep设备转发，联想到网桥的特性，接入到桥中的端口，会由网桥负责转发数据，因此，相当于所有容器发出的数据都会经过到vxlan的端口，vxlan将流量转到对端的vtep端点，再次由网桥负责转到容器中。

**k8s-slave-1**
```
[root@k8s-slave-1 ~]# ip link del vxlan20
[root@k8s-slave-1 ~]# ip link add vxlan_docker type vxlan id 100 remote 172.17.176.33 dstport 4789 dev eth0
[root@k8s-slave-1 ~]# ip link set vxlan_docker up
[root@k8s-slave-1 ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
br-9ff93a9f1543		8000.0242f7e4fdae	no		veth5e09997
cni0		8000.72f30299e263	no		veth87715ed8
							vethac6391dd
							vethb38cbac8
							vethcd980518
							vethd44f754b
							vethf1bc1704
docker0		8000.0242405b9cda	no		
[root@k8s-slave-1 ~]# brctl addif br-9ff93a9f1543 vxlan_docker
[root@k8s-slave-1 ~]# ip -d link show vxlan_docker
30: vxlan_docker: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br-9ff93a9f1543 state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether ea:0c:44:c3:1a:32 brd ff:ff:ff:ff:ff:ff promiscuity 1 
    vxlan id 100 remote 172.17.176.33 dev eth0 srcport 0 0 dstport 4789 ageing 300 noudpcsum noudp6zerocsumtx noudp6zerocsumrx 
    bridge_slave state forwarding priority 32 cost 100 hairpin off guard off root_block off fastleave off learning on flood on port_id 0x8002 port_no 0x2 designated_port 32770 designated_cost 0 designated_bridge 8000.2:42:f7:e4:fd:ae designated_root 8000.2:42:f7:e4:fd:ae hold_timer    0.00 message_age_timer    0.00 forward_delay_timer    0.00 topology_change_ack 0 config_pending 0 proxy_arp off proxy_arp_wifi off mcast_router 1 mcast_fast_leave off mcast_flood on addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
```
**k8s-slave-2**
```
[root@k8s-slave-2 ~]# ip link add vxlan_docker type vxlan id 100 remote 172.17.176.32 dstport 4789 dev eth0
[root@k8s-slave-2 ~]# ip link set vxlan_docker up
[root@k8s-slave-2 ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
br-574629858e91		8000.0242968ff521	no		vethbb37701
cni0		8000.225793974e68	no		
docker0		8000.0242e35c691c	no		
[root@k8s-slave-2 ~]# brctl addif br-574629858e91 vxlan_docker
[root@k8s-slave-2 ~]# docker exec vxlan-test ping 172.20.0.2
PING 172.20.0.2 (172.20.0.2): 56 data bytes
64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.395 ms
```
##  3.5 Flannel的vxlan实现
![image](./img/day03-7.png)
flannel如何为每个节点分配Pod地址段
```
[root@k8s-master ~]# kubectl -n kube-system get pods|grep flannel
kube-flannel-ds-amd64-lgsl2          1/1     Running   0          11d
kube-flannel-ds-amd64-ql4t8          1/1     Running   0          7h42m
kube-flannel-ds-amd64-s2jx2          1/1     Running   0          11d
[root@k8s-master ~]# kubectl -n kube-system exec kube-flannel-ds-amd64-lgsl2 cat /etc/kube-flannel/net-conf.json
{
  "Network": "10.244.0.0/16",
  "Backend": {
    "Type": "vxlan"
  }
}
[root@k8s-master ~]# kubectl -n luffy get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
myblog-7fb9874dd9-2xsmd   1/1     Running   0          7h36m   10.244.1.21   k8s-slave-1   <none>           <none>
mysql-778f489b9-qhbqv     1/1     Running   0          7d8h    10.244.1.12   k8s-slave-1   <none>           <none>
#查看k8s-master主机分配的地址段
[root@k8s-master ~]# cat /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
# kubelet启动容器的时候就可以按照本机的网段配置来为pod设置IP地址
```
-   vtep设备在哪里：
```
# 没有remote ip，非点对点
[root@k8s-master ~]# ip -d link show flannel.1
56: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/ether c6:bc:5d:17:f8:2f brd ff:ff:ff:ff:ff:ff promiscuity 0 
    vxlan id 1 local 172.17.176.31 dev eth0 srcport 0 0 dstport 8472 nolearning ageing 300 noudpcsum noudp6zerocsumtx noudp6zerocsumrx addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
[root@k8s-master ~]# brctl show cni0
bridge name	bridge id		STP enabled	interfaces
cni0		8000.668464ccb780	no		veth849784a3
							vethaa77f80f
[root@k8s-master ~]# route -n|egrep 'flannel|cni0'
10.244.0.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.244.1.0      10.244.1.0      255.255.255.0   UG    0      0        0 flannel.1
10.244.2.0      10.244.2.0      255.255.255.0   UG    0      0        0 flannel.1
```
-   vtep封包的时候，如何拿到目的vetp端的IP及MAC信息

```powershell
# flanneld启动的时候会需要配置--iface=eth0,通过该配置可以将网卡的ip及Mac信息存储到ETCD中，
# 这样，flannel就知道所有的节点分配的IP段及vtep设备的IP和MAC信息，而且所有节点的flanneld都可以感知到节点的添加和删除操作，就可以动态的更新本机的转发配置
```
##  3.6 跨主机Pod通信的流量详细过程
-   pod1从桥接网卡cni出发,走到flannel1.1 vtep设备,去etcd中查找数据,发给对应的宿主机
```
[root@k8s-slave-1 ~]# ping 10.244.2.13 -c 1
PING 10.244.2.13 (10.244.2.13) 56(84) bytes of data.
64 bytes from 10.244.2.13: icmp_seq=1 ttl=63 time=0.381 ms

--- 10.244.2.13 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.381/0.381/0.381/0.000 ms
[root@k8s-slave-1 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.17.191.253  0.0.0.0         UG    0      0        0 eth0
10.244.0.0      10.244.0.0      255.255.255.0   UG    0      0        0 flannel.1
10.244.1.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.244.2.0      10.244.2.0      255.255.255.0   UG    0      0        0 flannel.1
169.254.0.0     0.0.0.0         255.255.0.0     U     1002   0        0 eth0
172.17.176.0    0.0.0.0         255.255.240.0   U     0      0        0 eth0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.20.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-9ff93a9f1543
[root@k8s-slave-1 ~]# ip -d link show flannel.1
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/ether 3e:3d:92:24:5f:22 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    vxlan id 1 local 172.17.176.32 dev eth0 srcport 0 0 dstport 8472 nolearning ageing 300 noudpcsum noudp6zerocsumtx noudp6zerocsumrx addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
[root@k8s-slave-1 ~]# bridge fdb show dev flannel.1
f2:d0:fb:4a:83:2e dst 172.17.176.33 self permanent
c6:bc:5d:17:f8:2f dst 172.17.176.31 self permanent
```


##  3.7 利用host-gw模式提升集群网络性能

vxlan模式适用于三层可达的网络环境，对集群的网络要求很宽松，但是同时由于会通过VTEP设备进行额外封包和解包，因此给性能带来了额外的开销。

网络插件的目的其实就是将本机的cni0网桥的流量送到目的主机的cni0网桥。实际上有很多集群是部署在同一二层网络环境下的，可以直接利用二层的主机当作流量转发的网关。这样的话，可以不用进行封包解包，直接通过路由表去转发流量。

![image](./img/day03-6.png)

为什么三层可达的网络不直接利用网关转发流量？

```powershell
内核当中的路由规则，网关必须在跟主机当中至少一个 IP 处于同一网段。
由于k8s集群内部各节点均需要实现Pod互通，因此，也就意味着host-gw模式需要整个集群节点都在同一二层网络内。
```

#   4. Kubernetes认证授权
##  4.1 APIServer安全控制
![image](./img/day03-8.png)

- Authentication：身份认证

  1. 这个环节它面对的输入是整个`http request`，负责对来自client的请求进行身份校验，支持的方法包括:

     - `basic auth`

     - `client证书验证（https双向验证）`

     - `jwt token`(用于serviceaccount)

  2. APIServer启动时，可以指定一种Authentication方法，也可以指定多种方法。如果指定了多种方法，那么APIServer将会逐个使用这些方法对客户端请求进行验证， 只要请求数据通过其中一种方法的验证，APIServer就会认为Authentication成功；

  3. 使用kubeadm引导启动的k8s集群，apiserver的初始配置中，默认支持`client证书`验证和`serviceaccount`两种身份验证方式。 证书认证通过设置`--client-ca-file`根证书以及`--tls-cert-file`和`--tls-private-key-file`来开启。

  4. 在这个环节，apiserver会通过client证书或 `http header`中的字段(比如serviceaccount的`jwt token`)来识别出请求的`用户身份`，包括”user”、”group”等，这些信息将在后面的`authorization`环节用到。

- Authorization：鉴权，你可以访问哪些资源

  1. 这个环节面对的输入是`http request context`中的各种属性，包括：`user`、`group`、`request path`（比如：`/api/v1`、`/healthz`、`/version`等）、 `request verb`(比如：`get`、`list`、`create`等)。

  2. APIServer会将这些属性值与事先配置好的访问策略(`access policy`）相比较。APIServer支持多种`authorization mode`，包括`Node、RBAC、Webhook`等。

  3. APIServer启动时，可以指定一种`authorization mode`，也可以指定多种`authorization mode`，如果是后者，只要Request通过了其中一种mode的授权， 那么该环节的最终结果就是授权成功。在较新版本kubeadm引导启动的k8s集群的apiserver初始配置中，`authorization-mode`的默认配置是`”Node,RBAC”`。

- Admission Control：[准入控制](http://docs.kubernetes.org.cn/144.html)，一个控制链(层层关卡)，用于拦截请求的一种方式。偏集群安全控制、管理方面。

  - 为什么需要？

    认证与授权获取 http 请求 header 以及证书，无法通过body内容做校验。

    Admission 运行在 API Server 的增删改查 handler 中，可以自然地操作 API resource 

  - 举个栗子

    - 以NamespaceLifecycle为例， 该插件确保处于Termination状态的Namespace不再接收新的对象创建请求，并拒绝请求不存在的Namespace。该插件还可以防止删除系统保留的Namespace:default，kube-system，kube-public。 

    - LimitRanger，若集群的命名空间设置了LimitRange对象，若Pod声明时未设置资源值，则按照LimitRange的定义来未Pod添加默认值

      ```yaml
      apiVersion: v1
      kind: LimitRange
      metadata:
        name: mem-limit-range
        namespace: luffy
      spec:
        limits:
        - default:
            memory: 512Mi
          defaultRequest:
            memory: 256Mi
          type: Container
      ---
      apiVersion: v1
      kind: Pod
      metadata:
        name: default-mem-demo-2
      spec:
        containers:
        - name: default-mem-demo-2-ctr
          image: nginx:alpine
      
      ```

    - NodeRestriction， 此插件限制kubelet修改Node和Pod对象，这样的kubelets只允许修改绑定到Node的Pod API对象，以后版本可能会增加额外的限制 。开启Node授权策略后，默认会打开该项

  - 怎么用？

    APIServer启动时通过 `--enable-admission-plugins --disable-admission-plugins` 指定需要打开或者关闭的 Admission Controller 

  - 场景

    - 自动注入sidecar容器或者initContainer容器
    - webhook admission，实现业务自定义的控制需求


##  4.2 kubectl的认证授权
kubectl的日志调试级别：

| 信息 | 描述                                                         |
| :--- | :----------------------------------------------------------- |
| v=0  | 通常，这对操作者来说总是可见的。                             |
| v=1  | 当您不想要很详细的输出时，这个是一个合理的默认日志级别。     |
| v=2  | 有关服务和重要日志消息的有用稳定状态信息，这些信息可能与系统中的重大更改相关。这是大多数系统推荐的默认日志级别。 |
| v=3  | 关于更改的扩展信息。                                         |
| v=4  | 调试级别信息。                                               |
| v=6  | 显示请求资源。                                               |
| v=7  | 显示 HTTP 请求头。                                           |
| v=8  | 显示 HTTP 请求内容。                                         |
| v=9  | 显示 HTTP 请求内容，并且不截断内容。                         |

```
[root@k8s-master ~]# kubectl get nodes -v=7
I1020 11:36:55.173590   11849 loader.go:375] Config loaded from file:  /root/.kube/config
I1020 11:36:55.178022   11849 round_trippers.go:420] GET https://172.17.176.31:6443/api/v1/nodes?limit=500
I1020 11:36:55.178035   11849 round_trippers.go:427] Request Headers:
I1020 11:36:55.178040   11849 round_trippers.go:431]     Accept: application/json;as=Table;v=v1beta1;g=meta.k8s.io, application/json
I1020 11:36:55.178044   11849 round_trippers.go:431]     User-Agent: kubectl/v1.16.2 (linux/amd64) kubernetes/c97fe50
I1020 11:36:55.184988   11849 round_trippers.go:446] Response Status: 200 OK in 6 milliseconds
```

- 当`kubectl`使用这种`kubeconfig`方式访问集群时，`Kubernetes`的`kube-apiserver`是如何对来自`kubectl`的访问进行身份验证(`authentication`)和授权(`authorization`)的呢？
- 为什么来自`kubectl`的请求拥有最高的管理员权限呢？ 

查看`/root/.kube/config`文件：

前面提到过apiserver的authentication支持通过`tls client certificate、basic auth、token`等方式对客户端发起的请求进行身份校验， 从kubeconfig信息来看，kubectl显然在请求中使用了`tls client certificate`的方式，即客户端的证书。 

证书base64解码：
```
[root@k8s-master ~]# echo LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDQWRxZ0F3SUJBZ0lJWUR4aVdVb0wzUlV3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TURFd01EZ3hNRFF6TVRWYUZ3MHlNVEV3TURneE1EUXpNVGRhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXR0WDM4YTVzekFvQUFxQisKcURSU0ZTSHhMOHdPbGMvQXF5K2drVEpCUVkvclVXRDcxa0xZWXBpL1Z5YkhTanY1bzJwR1gvNGFsb09XdmdPWQpGMmduODZnR3VsSk9INkFXdmF1K1BFbnBwRW1RYllzdjRIeGlNMmo1WVFLdWJ6SkE2MUw2QkNKOXlWR0dhZG1QCjhBZURkcUtsSGo4T3IvaEM3RlBVL2N3cnBUQzVqaTNrTWlFQS9TalJ2d0c3VGN0dXhWanJRMVp5RVNQc1Y1WWQKVFZ0Z0FSWVZyS2JnMjZXU25QK0VVVnE1UExxR3pLQVZramxHbzZIZWVWcXlxQnFmOEJSbExhUWFsMnVDMTNGMApzaXdtclpJWlI4YURpSnQycGtsa3dEazJxWlJCMkVzSXFudDZRczRXQVk4NjF1OEk2MVFwZ1Zla0F1VHhrSDdpCms1MW44UUlEQVFBQm95Y3dKVEFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFKdm04UTQrUjhIVjZxKyszbGJMM3A0NkxiSUtrUkdpcm0yawp2eXB2RmJTMjNlWjNkU2RXM0lJMk03RTVwclJEM1RPbENXbkRVTFVFNkNnWi9iT1VDaVJvSnBBYnBCSWNIRnE0CkNTUHp5OEgwSWM4UFNBa1dwMVJ5ZGpxWEV3T1VPalB3cHEwRFB3alFxUE1oVlJjWWdsYTdaN3E4N3hwWDU3R04KTXJLYkl2Z3BiRXJxaHBlaXVrSnNEck1jTThXRFFzM3NtRzR5ckt6RUE3UmVUK080bXZmN1k2TXFSNEpuNDlaZgorVXYyanFrbzVNcXFjWkN5Ni9pZzJMS3BOb1I5S3JvK3dWZ1FDOTBSdmU1Q1lKejJTSmdrdkNMUmZCVXB1T0hVClpxeVlONXY4akt6RXdqV2pxL0tGLzRHVUM5aGNtZUN4RGc5a2FxamdSeGRici9ZQzdCWT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo= |base64 -d > kubectl.crt
```
说明在认证阶段，`apiserver`会首先使用`--client-ca-file`配置的CA证书去验证kubectl提供的证书的有效性,基本的方式 
```
[root@k8s-master ~]# openssl verify -CAfile /etc/kubernetes/pki/ca.crt kubectl.crt
kubectl.crt: OK
```
除了认证身份，还会取出必要的信息供授权阶段使用，文本形式查看证书内容：
```
[root@k8s-master ~]# openssl x509 -in kubectl.crt -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 6934525661877624085 (0x603c62594a0bdd15)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Oct  8 10:43:15 2020 GMT
            Not After : Oct  8 10:43:17 2021 GMT
        Subject: O=system:masters, CN=kubernetes-admin
```
认证通过后，提取出签发证书时指定的CN(Common Name),`kubernetes-admin`，作为请求的用户名 (User Name), 从证书中提取O(Organization)字段作为请求用户所属的组 (Group)，`group = system:masters`，然后传递给后面的授权模块。

kubeadm在init初始引导集群启动过程中，创建了许多默认的RBAC规则， 在k8s有关RBAC的官方文档中，我们看到下面一些`default clusterrole`列表: 
![image](./img/day03-9.png)
其中第一个cluster-admin这个cluster role binding绑定了system:masters group，这和authentication环节传递过来的身份信息不谋而合。 沿着system:masters group对应的cluster-admin clusterrolebinding“追查”下去，真相就会浮出水面。

查看一下这一binding：
```
[root@k8s-master ~]# kubectl describe clusterrolebinding cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind   Name            Namespace
  ----   ----            ---------
  Group  system:masters  
```
看到在kube-system名字空间中，一个名为cluster-admin的clusterrolebinding将cluster-admin cluster role与system:masters Group绑定到了一起， 赋予了所有归属于system:masters Group中用户cluster-admin角色所拥有的权限。

再来查看一下cluster-admin这个role的具体权限信息：
```
[root@k8s-master ~]# kubectl describe clusterrole cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]
```
![image](./img/day03-10.png)

##  4.3 RBAC
Role-Based Access Control，基于角色的访问控制， apiserver启动参数添加--authorization-mode=RBAC 来启用RBAC认证模式，kubeadm安装的集群默认已开启。[官方介绍](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

查看开启：
```
[root@k8s-master ~]# ps aux |egrep [a]piserver
root      7244  2.5  4.0 552692 311192 ?       Ssl  Oct08 427:48 kube-apiserver --advertise-address=172.17.176.31 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --insecure-port=0 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

RBAC模式引入了4个资源类型：

- Role，角色

  一个Role只能授权访问单个namespace 

```
 ## 示例定义一个名为pod-reader的角色，该角色有读取default这个命名空间下的pods的权限
 kind: Role
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   namespace: default
   name: pod-reader
 rules:
 - apiGroups: [""] # "" indicates the core APIgroup
   resources: ["pods"]
   verbs: ["get", "watch", "list"]
   
 ## apiGroups: "","apps", "autoscaling","batch", kubectl api-versions
 ## resources: "services","pods","deployments"... kubectl api-resources
 ## verbs: "get", "list", "watch", "create","update", "patch", "delete", "exec"
 
 ## https://kubernetes.io/docs/reference/generted/kubernetes-api/v1.18/
  
```
- ClusterRole

  一个ClusterRole能够授予和Role一样的权限，但是它是集群范围内的。 

```yaml
## 定义一个集群角色，名为secret-reader，该可以读取所有的namespace中的secret资源
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # "namespace" omitted since ClusterRoles arnot namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]

# User,Group,ServiceAccount
```

- Rolebinding

  将role中定义的权限分配给用户和用户组。RoleBinding包含主题（users,groups,或service accounts）和授予角色的引用。对于namespace内的授权使用RoleBinding，集群范围内使用ClusterRoleBinding。

```yaml
## 定义一个角色绑定，将pod-reader这个role限授予给jane这个User，使得jane可以在读取deflt这个命名空间下的所有的pod数据
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User #这里可以是User,Group,ServiceAccount
  name: jane 
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Rol#这里可以是Role或者ClusterRole,若是Clusterle，则权限也仅限于rolebinding的内部
  name: pod-reader # match the name of thRole or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io

```

**注意：rolebinding既可以绑定role，也可以绑定clusterrole，当绑定clusterrole的时候，subject的权限也会被限定于rolebinding定义的namespace内部，若想跨namespace，需要使用clusterrolebinding**

```yaml
## 定义一个角色绑定，将dave这个用户和secret-ader这个集群角色绑定，虽然secret-reader是角色，但是因为是使用rolebinding绑定的，因此ve的权限也会被限制在development这个命名空间内
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "dave" to reasecrets in the "development" namespace.
# You need to already have a ClusterRole name"secret-reader".
kind: RoleBinding
metadata:
  name: read-secrets
  #
  # The namespace of the RoleBindindetermines where the permissions argranted.
  # This only grants permissions within th"development" namespace.
  namespace: development
subjects:
- kind: User
  name: dave # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: dave # Name is case sensitive
  namespace: luffy
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

考虑一个场景：  如果集群中有多个namespace分配给不同的管理员，每个namespace的权限是一样的，就可以只定义一个clusterrole，然后通过rolebinding将不同的namespace绑定到管理员身上，否则就需要每个namespace定义一个Role，然后做一次rolebinding。

- ClusterRolebingding

允许跨namespace进行授权

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# This cluster role binding allows anyone ithe "manager" group to read secrets in annamespace.
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```
  
## 4.4 kubelet的认证授权
```
[root@k8s-master ~]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Thu 2020-10-08 18:43:25 CST; 1 weeks 4 days ago
     Docs: https://kubernetes.io/docs/
 Main PID: 6641 (kubelet)
    Tasks: 20
   Memory: 45.4M
   CGroup: /system.slice/kubelet.service
           └─6641 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --conf...
```
查看`/etc/kubernetes/kubelet.conf`，解析证书
```
[root@k8s-master ~]# cat /etc/kubernetes/kubelet.conf
[root@k8s-master ~]# echo LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM5akNDQWQ2Z0F3SUJBZ0lJZFFBanMrWXNLQ2N3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TURFd01EZ3hNRFF6TVRWYUZ3MHlNVEV3TURneE1EUXpNVGhhTURneApGVEFUQmdOVkJBb1RESE41YzNSbGJUcHViMlJsY3pFZk1CMEdBMVVFQXhNV2MzbHpkR1Z0T201dlpHVTZhemh6CkxXMWhjM1JsY2pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBSzczT25UMzVHMzIKcldzeDB0dnZNZWJ0VGl6cnNGQXkrbUhhYWIvYWhKZXdFMHY3dHVoS052TWs5cjN4dzROajVKbmNmdW1GQjl6UwpkQnJNVHNwTC9PM2N2Vmp3Wk14MzVwZllDWE9tMzloRzFWK25QRVFJbUdQZDgrUmZ1TE9ySVQ4a0l3QUNFb0hECjdUMVpMVklZRm5Kd2xCVlZiYWt6TW5tY2g1bnhtVEhna3pTcEFpY3Z0QW4vQ3piMnB0NWdoY1BuR1NIQTJaZjUKdXgrR0x0ekhWZ2E0UlRxdkJrZ3I1ZElMNWdIeDRPaHRFVjFqdzFjeDBadVVRWUNnVFRkREJuOTcyNENpclBvQgp6MmZ0WTJzeTFucit4dUdnKzgxWUhUMHJndWtqTmszT1NpYVVJQVdCdVhncDJxZ2hxWlN3Qyt4clJsVVBqNFdnCm9lQlViajdsOU84Q0F3RUFBYU1uTUNVd0RnWURWUjBQQVFIL0JBUURBZ1dnTUJNR0ExVWRKUVFNTUFvR0NDc0cKQVFVRkJ3TUNNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUNjRkZYdlY1T0tmQ24yVkpmV2ZXeGxNQ1NyRVdLbgpkc0Q3Y0thQTZmQ0hZeUZ3VDREYlFaeGk4Z1Q4Qy9ic2lOVmhpOUR1WC85OFQ0c2VRZ2NQQ2RiTWtScmZFSGhmCnF2RFRONmg0c0l3clo1N3ZVS2RuM3dmdXErdDNaWGdPbXM2WTlUcUlJeG9Gc2FwRHpQa2NPZGlQaWNURHh5WEkKbnBmSEpYSUphTjlOaURtdzFXcERJcWdzK1cxUHFZbzFlWVYwWnF0b0g4UDFsOTUrS1NoNDkzamRycXY0K2VnbgphUEtmc29icm1FN3lqbUt1TnM0d05kQmtCdmxPaFZXcmd0eldBNTBKVnl3Y3lweW1EL1BQaTJiU0w0MEJUbzFPCnc5aVA3dTduaXg3RS8rOHh4MjlJRVl0TVdJVExUOW9xc3dabXNack01RGdKS0lQekd3VE1wMHpCCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K  |base64 -d >kubelet.crt
[root@k8s-master ~]# openssl x509 -in kubelet.crt -text
Subject: O=system:nodes, CN=system:node:k8s-master
```

我们知道，k8s会把O作为Group来进行请求，因此如果有权限绑定给这个组，肯定在clusterrolebinding的定义中可以找得到。因此尝试去找一下绑定了system:nodes组的clusterrolebinding
```
[root@k8s-master ~]# kubectl get clusterrolebinding -oyaml|grep -n10 system:nodes
94-    resourceVersion: "169"
95-    selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/kubeadm%3Anode-autoapprove-certificate-rotation
96-    uid: 89a06a00-4bdc-49ba-9caa-daebaef41998
97-  roleRef:
98-    apiGroup: rbac.authorization.k8s.io
99-    kind: ClusterRole
100-    name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
101-  subjects:
102-  - apiGroup: rbac.authorization.k8s.io
103-    kind: Group
104:    name: system:nodes
105-- apiVersion: rbac.authorization.k8s.io/v1
106-  kind: ClusterRoleBinding
107-  metadata:
108-    creationTimestamp: "2020-10-08T10:43:35Z"
109-    name: kubeadm:node-proxier
110-    resourceVersion: "185"
111-    selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/kubeadm%3Anode-proxier
112-    uid: 3da7a206-60d6-42ed-b040-9ff86ff32124
113-  roleRef:
114-    apiGroup: rbac.authorization.k8s.io
```
**查询权限**
```
[root@k8s-master ~]# kubectl describe clusterrole system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
Name:         system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources                                                      Non-Resource URLs  Resource Names  Verbs
  ---------                                                      -----------------  --------------  -----
  certificatesigningrequests.certificates.k8s.io/selfnodeclient  []                 []              [create]
```
 结局有点意外，除了`system:certificates.k8s.io:certificatesigningrequests:selfnodeclient`外，没有找到system相关的rolebindings，显然和我们的理解不一样。 尝试去找[资料](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#core-component-roles)，发现了这么一段 :

| Default ClusterRole            | Default ClusterRoleBinding          | Description                                                  |
| :----------------------------- | :---------------------------------- | :----------------------------------------------------------- |
| system:kube-scheduler          | system:kube-scheduler user          | Allows access to the resources required by the [scheduler](https://kubernetes.io/docs/reference/generated/kube-scheduler/)component. |
| system:volume-scheduler        | system:kube-scheduler user          | Allows access to the volume resources required by the kube-scheduler component. |
| system:kube-controller-manager | system:kube-controller-manager user | Allows access to the resources required by the [controller manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/) component. The permissions required by individual controllers are detailed in the [controller roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#controller-roles). |
| system:node                    | None                                | Allows access to resources required by the kubelet, **including read access to all secrets, and write access to all pod status objects**. You should use the [Node authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/) and [NodeRestriction admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction) instead of the `system:node` role, and allow granting API access to kubelets based on the Pods scheduled to run on them. The `system:node` role only exists for compatibility with Kubernetes clusters upgraded from versions prior to v1.8. |
| system:node-proxier            | system:kube-proxy user              | Allows access to the resources required by the [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)component. |

大致意思是说：之前会定义system:node这个角色，目的是为了kubelet可以访问到必要的资源，包括所有secret的读权限及更新pod状态的写权限。如果1.8版本后，是建议使用 [Node authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/) and [NodeRestriction admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction) 来代替这个角色的。


**查看一下策略**
![image](./img/day03-11.png)


## 4.5 ServiceAccount及K8SApi调用
前面说，认证可以通过证书，也可以通过使用ServiceAccount（服务账户）的方式来做认证。大多数时候，我们在基于k8s做二次开发时都是选择通过ServiceAccount + RBAC 的方式。我们之前访问dashboard的时候，是如何做的？
```
[root@k8s-master ~]# kubectl -n kubernetes-dashboard get sa admin -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2020-10-08T10:55:24Z"
  name: admin
  namespace: kubernetes-dashboard
  resourceVersion: "1643"
  selfLink: /api/v1/namespaces/kubernetes-dashboard/serviceaccounts/admin
  uid: 578f41c4-3dc7-4951-bca5-fd688d203838
secrets:
- name: admin-token-zljl7
```
**查看对应的token**
```
[root@k8s-master ~]# kubectl -n kubernetes-dashboard describe secret admin-token-zljl7
Name:         admin-token-zljl7
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin
              kubernetes.io/service-account.uid: 578f41c4-3dc7-4951-bca5-fd688d203838

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Il9iSUNSbkthVmNfdWo4eEFpVDgteThwMmZyQjdIMGFWUUJJT3plMW1XU2MifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi10b2tlbi16bGpsNyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjU3OGY0MWM0LTNkYzctNDk1MS1iY2E1LWZkNjg4ZDIwMzgzOCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDphZG1pbiJ9.GedwSDre6sKpi0JfrY4NCEzwuqY-tqU4wONQd2Fi0C2aWvZIwll9xDDTHlPd2sXLbCzCwM9JOKyZDoaHDAhoHggIyyyyrmt2PwsQKwwUM-3zOWI-Bzy0imviQqhsVeoWtZqV89Scw8x6VLwsv3470hxgrAL0Gtbafvg8jJG90vrlxe_G12XfCN7lGb1BYI5fY4oqUgalw3P8lqqzohEs9qVX3B7IHV3F1zjW1irqAcocIpHDG9VXKkVkgrNw85eoj92UdQTKVQJpkSxfB-OEoy3rnDfMHbVUsgW2nYCz4NZeJJI6OfgmseZSpgK9buD45B4FxkkjHfWJ93TZTT2Oag
```
**添加一个serviceaccount和ClusterRoleBinding**
```
[root@k8s-master ~]# cat test-sa.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test
  namespace: kubernetes-dashboard

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: test
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: test
  namespace: kubernetes-dashboard
```
**查看test的token**
```
[root@k8s-master ~]# kubectl -n kubernetes-dashboard get  sa  test -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2020-10-20T08:15:05Z"
  name: test
  namespace: kubernetes-dashboard
  resourceVersion: "1559507"
  selfLink: /api/v1/namespaces/kubernetes-dashboard/serviceaccounts/test
  uid: 1fadbe23-fb90-40e4-a003-d8479a902f45
secrets:
- name: test-token-ftrmr
[root@k8s-master ~]# kubectl -n kubernetes-dashboard describe  secret test-token-ftrmr 
[root@k8s-master ~]# curl -k  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Il9iSUNSbkthVmNfdWo4eEFpVDgteThwMmZyQjdIMGFWUUJJT3plMW1XU2MifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJ0ZXN0LXRva2VuLWZ0cm1yIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6InRlc3QiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIxZmFkYmUyMy1mYjkwLTQwZTQtYTAwMy1kODQ3OWE5MDJmNDUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6dGVzdCJ9.Rn9a_pJFCQaT9GWL0KWqgxjOKkW0MhJupBjke3SqD_pGtdV8dlabX3MbAtp3GvVW9K8CgeHrX77XyD3-ANOORsE_9Ry5xaCdCQHJFpbw86XeSK8jJ_nW2gx710kxj3ocrh1IJwZwOtua35VaniHzHGVsMauWoFsrTM5d_LCUBYNkkLDdG5ZSfCTR-wmco4AgMqAZAY5p3EHjgfiEttXiDEjHD7n30YESd0qrjWiM3ySKJopWy5oNTQVICJ2x0q5svCJ6Izhj-YCs_j5ap4QTvMkORRKQhofc_zeZmD2bZUXaBRhRKxuNtHa9jNTV93p2CEmjhZmP-OaWyB_gU7rvLA" https://172.17.176.31:6443/api/v1/namespaces/luffy/pods?limit=500
```
**验证结果**
```
[root@k8s-master ~]# curl -k  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Il9iSUNSbkthVmNfdWo4eEFpVDgteThwMmZyQjdIMGFWUUJJT3plMW1XU2MifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJ0ZXN0LXRva2VuLWZ0cm1yIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6InRlc3QiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIxZmFkYmUyMy1mYjkwLTQwZTQtYTAwMy1kODQ3OWE5MDJmNDUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6dGVzdCJ9.Rn9a_pJFCQaT9GWL0KWqgxjOKkW0MhJupBjke3SqD_pGtdV8dlabX3MbAtp3GvVW9K8CgeHrX77XyD3-ANOORsE_9Ry5xaCdCQHJFpbw86XeSK8jJ_nW2gx710kxj3ocrh1IJwZwOtua35VaniHzHGVsMauWoFsrTM5d_LCUBYNkkLDdG5ZSfCTR-wmco4AgMqAZAY5p3EHjgfiEttXiDEjHD7n30YESd0qrjWiM3ySKJopWy5oNTQVICJ2x0q5svCJ6Izhj-YCs_j5ap4QTvMkORRKQhofc_zeZmD2bZUXaBRhRKxuNtHa9jNTV93p2CEmjhZmP-OaWyB_gU7rvLA" https://172.17.176.31:6443/api/v1/namespaces/luffy/pods?limit=500
```

##  4.6 创建用户认证授权的kubeconfig文件
```
#   生成私钥
[root@k8s-master auth]# openssl genrsa -out luffy.key 2048
Generating RSA private key, 2048 bit long modulus
.....................+++
.....................................................................+++
e is 65537 (0x10001)
#   生成证书请求文件
[root@k8s-master auth]# openssl req -new -key luffy.key -out luffy.csr -subj "/O=demo:luffy/CN=luffy"
# 证书拓展属性
[root@k8s-master auth]# cat extfile.conf
[ v3_ca ]
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
# 生成luffy.crt证书
[root@k8s-master auth]# openssl x509 -req -in luffy.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -sha256 -out luffy.crt -extensions v3_ca -extfile extfile.conf -days 3650
Signature ok
subject=/O=demo:luffy/CN=luffy
Getting CA Private Key
```
**配置kuberconfig文件**
```
# 创建kubeconfig文件，指定集群名称和地址
[root@k8s-master auth]# kubectl config set-cluster luffy-cluster --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true --server=https://172.17.176.31:6443 --kubeconfig=luffy.kubeconfig
Cluster "luffy-cluster" set.
# 为kubeconfig文件添加认证信息
[root@k8s-master auth]# kubectl config set-credentials luffy --client-certificate=luffy.crt --client-key=luffy.key --embed-certs=true --kubeconfig=luffy.kubeconfig
User "luffy" set.
# 为kubeconfig添加上下文配置
[root@k8s-master auth]# kubectl config set-context luffy-context --cluster=luffy-cluster --user=luffy --kubeconfig=luffy.kubeconfig
Context "luffy-context" created.
# 设置默认的上下文
[root@k8s-master auth]# kubectl config use-context luffy-context --kubeconfig=luffy.kubeconfig
Switched to context "luffy-context".
```
**验证权限**
```
[root@k8s-master auth]# kubectl get po --kubeconfig=luffy.kubeconfig 
Error from server (Forbidden): pods is forbidden: User "luffy" cannot list resource "pods" in API group "" in the namespace "default"
```
**为luffy用户添加luffy命名空间访问权限**
```
[root@k8s-master auth]# cat luffy-role.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: luffy
  name: luffy-admin
rules:
- apiGroups: [""] # "" 指定核心 API 组
  resources: ["*"]
  verbs: ["*"]
[root@k8s-master auth]# cat luffy-rolebinding.yaml 
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定使得用户 "jane" 能够读取 "default" 命名空间中的 Pods
kind: RoleBinding
metadata:
  name: luffy-admin
  namespace: luffy
subjects:
- kind: User
  name: luffy # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: luffy-admin # 这里的名称必须与你想要绑定的 Role 或 ClusterRole 名称一致
  apiGroup: rbac.authorization.k8s.io
```
**重新校验权限**
```
[root@k8s-master auth]# kubectl get po --kubeconfig=luffy.kubeconfig 
Error from server (Forbidden): pods is forbidden: User "luffy" cannot list resource "pods" in API group "" in the namespace "default"
[root@k8s-master auth]# kubectl -n luffy get po --kubeconfig=luffy.kubeconfig 
NAME                      READY   STATUS    RESTARTS   AGE
myblog-7fb9874dd9-2xsmd   1/1     Running   0          6d23h
myblog-7fb9874dd9-knk7w   1/1     Running   0          6d15h
mysql-778f489b9-qhbqv     1/1     Running   0          13d
```

#   5. 通过HPA实现业务应用的动态扩缩容
##  5.1 HPA控制器介绍
当系统资源过高的时候，我们可以使用如下命令来实现 Pod 的扩缩容功能
```
$ kubectl -n luffy scale deployment myblog --replicas=2
```
但是这个过程是手动操作的。在实际项目中，我们需要做到是的是一个自动化感知并自动扩容的操作。Kubernetes 也为提供了这样的一个资源对象：Horizontal Pod Autoscaling（Pod 水平自动伸缩），简称[HPA](https://v1-14.docs.kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 

![image](./img/day03-12.png)

基本原理：HPA 通过监控分析控制器控制的所有 Pod 的负载变化情况来确定是否需要调整 Pod 的副本数量

HPA的实现有两个版本：

- autoscaling/v1，只包含了根据CPU指标的检测，稳定版本
- autoscaling/v2beta1，支持根据memory或者用户自定义指标进行伸缩

如何获取Pod的监控数据？

- k8s 1.8以下：使用heapster，1.11版本完全废弃
- k8s 1.8以上：使用metric-server

思考：为什么之前用 heapster ，现在废弃了项目，改用 metric-server ？

heapster时代，apiserver 会直接将metric请求通过apiserver proxy 的方式转发给集群内的 hepaster 服务，采用这种 proxy 方式是有问题的：
```html
  http://kubernetes_master_address/api/v1/namespaces/namespace_name/services/service_name[:port_name]/proxy
```

- proxy只是代理请求，一般用于问题排查，不够稳定，且版本不可控

- heapster的接口不能像apiserver一样有完整的鉴权以及client集成

- pod 的监控数据是核心指标（HPA调度），应该和 pod 本身拥有同等地位，即 metric应该作为一种资源存在，如metrics.k8s.io 的形式，称之为 Metric Api

于是官方从 1.8 版本开始逐步废弃 heapster，并提出了上边 Metric api 的概念，而 metrics-server 就是这种概念下官方的一种实现，用于从 kubelet获取指标，替换掉之前的 heapster。

 `Metrics Server` 可以通过标准的 Kubernetes API 把监控数据暴露出来，比如获取某一Pod的监控数据：

![image](./img/day03-13.png)

## 5.2 Metric-Server

[官方介绍](https://v1-14.docs.kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/#metrics-server)

```
...
Metric server collects metrics from the Summary API, exposed by Kubelet on each node.

Metrics Server registered in the main API server through Kubernetes aggregator, which was introduced in Kubernetes 1.7
...
```
## 5.3 安装
官方代码仓库地址：https://github.com/kubernetes-sigs/metrics-server

Depending on your cluster setup, you may also need to change flags passed to the Metrics Server container. Most useful flags:

- `--kubelet-preferred-address-types` - The priority of node address types used when determining an address for connecting to a particular node (default [Hostname,InternalDNS,InternalIP,ExternalDNS,ExternalIP])
- `--kubelet-insecure-tls` - Do not verify the CA of serving certificates presented by Kubelets. For testing purposes only.
- `--requestheader-client-ca-file` - Specify a root certificate bundle for verifying client certificates on incoming requests.

```powershell
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
```

修改args参数：

```powershell
...
 84       containers:
 85       - name: metrics-server
 86         image: registry.aliyuncs.com/google_containers/metrics-server-amd64:v0.3.6
 87         imagePullPolicy: IfNotPresent
 88         args:
 89           - --cert-dir=/tmp
 90           - --secure-port=4443
 91           - --kubelet-insecure-tls
 92           - --kubelet-preferred-address-types=InternalIP
...
```

执行安装：

```powershell
$ kubectl create -f components.yaml

$ kubectl -n kube-system get pods

$ kubectl top nodes
```
## 5.4 kubelet的指标采集
无论是 heapster还是 metric-server，都只是数据的中转和聚合，两者都是调用的 kubelet 的 api 接口获取的数据，而 kubelet 代码中实际采集指标的是 cadvisor 模块，你可以在 node 节点访问 10250 端口获取监控数据：

- Kubelet Summary metrics:  https://127.0.0.1:10250/metrics，暴露 node、pod 汇总数据
- Cadvisor metrics: https://127.0.0.1:10250/metrics/cadvisor，暴露 container 维度数据

```
[root@k8s-master auth]# kubectl describe secret admin-token-zljl7  -n kubernetes-dashboard
[root@k8s-master auth]#  curl -k  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Il9iSUNSbkthVmNfdWo4eEFpVDgteThwMmZyQjdIMGFWUUJJT3plMW1XU2MifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi10b2tlbi16bGpsNyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjU3OGY0MWM0LTNkYzctNDk1MS1iY2E1LWZkNjg4ZDIwMzgzOCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDphZG1pbiJ9.GedwSDre6sKpi0JfrY4NCEzwuqY-tqU4wONQd2Fi0C2aWvZIwll9xDDTHlPd2sXLbCzCwM9JOKyZDoaHDAhoHggIyyyyrmt2PwsQKwwUM-3zOWI-Bzy0imviQqhsVeoWtZqV89Scw8x6VLwsv3470hxgrAL0Gtbafvg8jJG90vrlxe_G12XfCN7lGb1BYI5fY4oqUgalw3P8lqqzohEs9qVX3B7IHV3F1zjW1irqAcocIpHDG9VXKkVkgrNw85eoj92UdQTKVQJpkSxfB-OEoy3rnDfMHbVUsgW2nYCz4NZeJJI6OfgmseZSpgK9buD45B4FxkkjHfWJ93TZTT2Oag" https://localhost:10250/metrics
```

kubelet虽然提供了 metric 接口，但实际监控逻辑由内置的cAdvisor模块负责，早期的时候，cadvisor是单独的组件，从k8s 1.12开始，cadvisor 监听的端口在k8s中被删除，所有监控数据统一由Kubelet的API提供。

cadvisor获取指标时实际调用的是 runc/libcontainer库，而libcontainer是对 cgroup文件 的封装，即 cadvsior也只是个转发者，它的数据来自于cgroup文件。

cgroup文件中的值是监控数据的最终来源，如

- mem usage的值，

  -  对于docker容器来讲，来源于`/sys/fs/cgroup/memory/docker/[containerId]/memory.usage_in_bytes`

  - 对于pod来讲，`/sys/fs/cgroup/memory/kubepods/besteffort/pod[podId]/memory.usage_in_bytes`或者

    `/sys/fs/cgroup/memory/kubepods/burstable/pod[podId]/memory.usage_in_bytes`

- 如果没限制内存，Limit = machine_mem，否则来自于
   `/sys/fs/cgroup/memory/docker/[id]/memory.limit_in_bytes`

- 内存使用率 = memory.usage_in_bytes/memory.limit_in_bytes

Metrics数据流：

![image](./img/day03-14.png) 
Metrics Server是独立的一个服务，只能服务内部实现自己的api，是如何做到通过标准的kubernetes 的API格式暴露出去的？

[kube-aggregator](https://github.com/kubernetes/kube-aggregator)

## 5.5 kube-aggregator聚合器及Metric-Server的实现
kube-aggregator是对 apiserver 的api的一种拓展机制，它允许开发人员编写一个自己的服务，并把这个服务注册到k8s的api里面，即扩展 API 。
![image](./img/day03-15.png) 
定义一个APIService对象：

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.luffy.k8s.io
spec:
  group: luffy.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: service-A       # 必须https访问
    namespace: luffy
    port: 443   
  version: v1beta1
  versionPriority: 100
```

k8s会自动帮我们代理如下url的请求：

```powershell
proxyPath := "/apis/" + apiService.Spec.Group + "/" + apiService.Spec.Version
```

即：https://192.168.136.10:6443/apis/luffy.k8s.io/v1beta1/xxxx转到我们的service-A服务中，service-A中只需要实现 `https://service-A/luffy.k8s.io/v1beta1/xxxx` 即可。



看下metric-server的实现：

```powershell
$ kubectl get apiservice 
NAME                       SERVICE                      AVAILABLE                      
v1beta1.metrics.k8s.io   kube-system/metrics-server		True

$ kubectl get apiservice v1beta1.metrics.k8s.io -oyaml
...
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server
    namespace: kube-system
    port: 443
  version: v1beta1
  versionPriority: 100
...

$ kubectl -n kube-system get svc metrics-server
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
metrics-server   ClusterIP   10.110.111.146   <none>        443/TCP   11h

$ curl -k  -H "Authorization: Bearer xxxx" https://10.110.111.146
{
  "paths": [
    "/apis",
    "/apis/metrics.k8s.io",
    "/apis/metrics.k8s.io/v1beta1",
    "/healthz",
    "/healthz/healthz",
    "/healthz/log",
    "/healthz/ping",
    "/healthz/poststarthook/generic-apiserver-start-informers",
    "/metrics",
    "/openapi/v2",
    "/version"
  ]
```

##  5.6 HPA实践
-   基于cpu的动态伸缩
```
[root@k8s-master hpa]# cat hpa-myblog.yaml 
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-myblog-cpu
  namespace: luffy
spec:
  maxReplicas: 3
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myblog
  targetCPUUtilizationPercentage: 10
```
> Deployment对象必须配置requests的参数，不然无法获取监控数据，也无法通过HPA进行动态伸缩

**验证**
```
[root@k8s-master hpa]# kubectl -n luffy get svc myblog
NAME     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
myblog   ClusterIP   10.106.250.146   <none>        80/TCP    14d
[root@k8s-master hpa]# kubectl -n luffy scale deploy myblog --replicas=1
deployment.apps/myblog scaled
[root@k8s-master hpa]# ab -n 100000 -c 1000 http://10.106.250.146/blog/index/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 10.106.250.146 (be patient)
apr_socket_recv: Connection refused (111)
Total of 1305 requests completed
[root@k8s-master ~]# kubectl -n luffy get pods 
NAME                      READY   STATUS    RESTARTS   AGE
myblog-7fb9874dd9-2xsmd   0/1     Running   1          7d
myblog-7fb9874dd9-qzvv9   1/1     Running   0          26s
myblog-7fb9874dd9-t2bh6   1/1     Running   0          26s
mysql-778f489b9-qhbqv     1/1     Running   0          14d
```

-   基于内存
```
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo-mem
  namespace: luffy
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo-mem
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 30
```
##  5.7 基于自定义指标的动态伸缩

 除了基于 CPU 和内存来进行自动扩缩容之外，我们还可以根据自定义的监控指标来进行。这个我们就需要使用 `Prometheus Adapter`，Prometheus 用于监控应用的负载和集群本身的各种指标，`Prometheus Adapter` 可以帮我们使用 Prometheus 收集的指标并使用它们来制定扩展策略，这些指标都是通过 APIServer 暴露的，而且 HPA 资源对象也可以很轻易的直接使用。 

![image](./img/day03-16.png)


#   6. kubernetes对接分部式存储
##  6.1 emptydir
k8s存储的目的就是保证Pod重建后，数据不丢失。简单的数据持久化的下述方式：

- emptyDir 

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: test-pd
  spec:
    containers:
    - image: k8s.gcr.io/test-webserver
      name: webserver
      volumeMounts:
      - mountPath: /cache
        name: cache-volume
    - image: k8s.gcr.io/test-redis
      name: redis
      volumeMounts:
      - mountPath: /data
        name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
  ```
- Pod内的容器共享卷的数据
- 存在于Pod的生命周期，Pod销毁，数据丢失
- Pod内的容器自动重建后，数据不会丢失

##  6.2 hostPath

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: test-pd
  spec:
    containers:
    - image: k8s.gcr.io/test-webserver
      name: test-container
      volumeMounts:
      - mountPath: /test-pd
        name: test-volume
    volumes:
    - name: test-volume
      hostPath:
        # directory location on host
        path: /data
        # this field is optional
        type: Directory
  ```

##  6.3 PV与PVC管理NFS存储卷实践
volume支持的种类众多（参考 https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes ），每种对应不同的存储后端实现，因此为了屏蔽后端存储的细节，同时使得Pod在使用存储的时候更加简洁和规范，k8s引入了两个新的资源类型，PV和PVC。

PersistentVolume（持久化卷），是对底层的存储的一种抽象，它和具体的底层的共享存储技术的实现方式有关，比如 Ceph、GlusterFS、NFS 等，都是通过插件机制完成与共享存储的对接。如使用PV对接NFS存储：
- capacity，存储能力， 目前只支持存储空间的设置， 就是我们这里的 storage=1Gi，不过未来可能会加入 IOPS、吞吐量等指标的配置。 
- accessModes，访问模式， 是用来对 PV 进行访问模式的设置，用于描述用户应用对存储资源的访问权限，访问权限包括下面几种方式： 
  - ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载
  - ReadOnlyMany（ROX）：只读权限，可以被多个节点挂载
  - ReadWriteMany（RWX）：读写权限，可以被多个节点挂载
- persistentVolumeReclaimPolicy，pv的回收策略, 目前只有 NFS 和 HostPath 两种类型支持回收策略 
  - Retain（保留）- 保留数据，需要管理员手工清理数据
  - Recycle（回收）- 清除 PV 中的数据，效果相当于执行 rm -rf /thevolume/*
  - Delete（删除）- 与 PV 相连的后端存储完成 volume 的删除操作，当然这常见于云服务商的存储服务，比如 ASW EBS。

因为PV是直接对接底层存储的，就像集群中的Node可以为Pod提供计算资源（CPU和内存）一样，PV可以为Pod提供存储资源。因此PV不是namespaced的资源，属于集群层面可用的资源。Pod如果想使用该PV，需要通过创建PVC挂载到Pod中。

```
[root@k8s-master ~]# cat pv-nfs.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity: 
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /data/k8s
    server: 158494a809-qpo3.cn-beijing.nas.aliyuncs.com
[root@k8s-master ~]# kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
nfs-pv   1Gi        RWO            Retain           Available                                   59s
```
一个 PV 的生命周期中，可能会处于4中不同的阶段：

- Available（可用）：表示可用状态，还未被任何 PVC 绑定
- Bound（已绑定）：表示 PV 已经被 PVC 绑定
- Released（已释放）：PVC 被删除，但是资源还未被集群重新声明
- Failed（失败）： 表示该 PV 的自动回收失败

```
[root@k8s-master ~]# cat pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
[root@k8s-master ~]# kubectl get pvc
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-nfs   Bound    nfs-pv   1Gi        RWO                           16s
```
**访问模式，storage大小（pvc大小需要小于pv大小），以及 PV 和 PVC 的 storageClassName 字段必须一样，这样才能够进行绑定。**
-   PersistentVolumeController会不断地循环去查看每一个 PVC，是不是已经处于 Bound（已绑定）状态。如果不是，那它就会遍历所有的、可用的 PV，并尝试将其与未绑定的 PVC 进行绑定，这样，Kubernetes 就可以保证用户提交的每一个 PVC，只要有合适的 PV 出现，它就能够很快进入绑定状态。而所谓将一个 PV 与 PVC 进行“绑定”，其实就是将这个 PV 对象的名字，填在了 PVC 对象的 spec.volumeName 字段上。
**与pod进行绑定**
```
[root@k8s-master ~]# cat deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-pvc
spec:
  replicas: 1
  selector:             #指定Pod的选择器
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: web
        volumeMounts:                        #挂载容器中的目录到pvc nfs中的目录
        - name: www
          mountPath: /usr/share/nginx/html
      volumes:
      - name: www
        persistentVolumeClaim:              #指定pvc
          claimName: pvc-nfs
```

## 6.4 storageClass实现动态挂载
创建pv及pvc过程是手动，且pv与pvc一一对应，手动创建很繁琐。因此，通过storageClass  +  provisioner的方式来实现通过PVC自动创建并绑定PV。

部署： https://github.com/kubernetes-retired/external-storage 

```
[root@k8s-master storageclass]# cat provisioner.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: luffy.com/nfs
            - name: NFS_SERVER
              value: 158494a809-qpo3.cn-beijing.nas.aliyuncs.com
            - name: NFS_PATH  
              value: /data/k8s
      volumes:
        - name: nfs-client-root
          nfs:
            server: 158494a809-qpo3.cn-beijing.nas.aliyuncs.com
            path: /data/k8s
[root@k8s-master storageclass]# cat rbac.yaml 
kind: ServiceAccount
apiVersion: v1
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
  namespace: nfs-provisioner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
  namespace: nfs-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: nfs-provisioner
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: nfs-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: nfs-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfs-provisioner
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
[root@k8s-master storageclass]# cat storage-class.yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs
provisioner: luffy.com/nfs
[root@k8s-master storageclass]# cat pvc.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim2
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
  storageClassName: nfs
```
**查看验证**
```
[root@k8s-master storageclass]# kubectl get pvc
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-nfs       Bound    nfs-pv                                     1Gi        RWO                           27h
test-claim2   Bound    pvc-d2cec10a-8d07-45fd-92a2-e6fb6e6f0c8b   1Mi        RWX            nfs            112s
```



#   7.  使用Helm3的部署
##  7.1 认识Helm
1. 为什么有helm？

2. Helm是什么？

   kubernetes的包管理器，“可以将Helm看作Linux系统下的apt-get/yum”。  

   - 对于应用发布者而言，可以通过Helm打包应用，管理应用依赖关系，管理应用版本并发布应用到软件仓库。

   - 对于使用者而言，使用Helm后不用需要了解Kubernetes的Yaml语法并编写应用部署文件，可以通过Helm下载并在kubernetes上安装需要的应用。

   除此以外，Helm还提供了kubernetes上的软件部署，删除，升级，回滚应用的强大功能。
3. Helm的重要概念

   - chart，应用的信息集合，包括各种对象的配置模板、参数定义、依赖关系、文档说明等
   - Repoistory，chart仓库，存储chart的地方，并且提供了一个该 Repository 的 Chart 包的清单文件以供查询。Helm 可以同时管理多个不同的 Repository。
   - release， 当 chart 被安装到 kubernetes 集群，就生成了一个 release ， 是 chart 的运行实例，代表了一个正在运行的应用 

helm 是包管理工具，包就是指 chart，helm 能够：

- 从零创建chart
- 与仓库交互，拉取、保存、更新 chart
- 在kubernetes集群中安装、卸载 release
- 更新、回滚、测试 release

##  7.2 安装与快速入门实践
下载最新的稳定版本：https://get.helm.sh/helm-v3.2.4-linux-amd64.tar.gz

更多版本可以参考： https://github.com/helm/helm/releases 

```powershell
# k8s-master节点
$ wget https://get.helm.sh/helm-v3.2.4-linux-amd64.tar.gz
$ tar -zxf helm-v3.2.4-linux-amd64.tar.gz

$ cp linux-amd64/helm /usr/local/bin/

# 验证安装
$ helm version
version.BuildInfo{Version:"v3.2.4", GitCommit:"0ad800ef43d3b826f31a5ad8dfbb4fe05d143688", GitTreeState:"clean", GoVersion:"go1.13.12"}
$ helm env

# 添加仓库
$ helm repo add stable http://mirror.azure.cn/kubernetes/charts/
# 同步最新charts信息到本地
$ helm repo update
```
##  7.3 Helm基本使用
-   创建一个chart
```
[root@k8s-master helm]# helm create hello-helm
[root@k8s-master helm]# ls -l
total 0
drwxr-xr-x 4 root root 93 Oct 25 10:53 hello-helm
#
```
-   编辑里面的hello-helm/values.yaml
```
[root@k8s-master helm]# helm install ./hello-helm
[root@k8s-master ~]# kubectl get pod,svc -o wide
NAME                                                 READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
pod/interesting-rabbit-hello-helm-5d9d68fd99-qj888   1/1     Running   0          42m   10.244.1.54   k8s-node1   <none>           <none>

NAME                                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE   SELECTOR
service/interesting-rabbit-hello-helm   NodePort    10.1.28.222   <none>        80:32584/TCP   42m   app.kubernetes.io/instance=interesting-rabbit,app.kubernetes.io/name=hello-helm
service/kubernetes                      ClusterIP   10.1.0.1      <none>        443/TCP        10d   <none>

#   查看版本
[root@k8s-master ~]# helm list
NAME              	REVISION	UPDATED                 	STATUS  	CHART           	APP VERSION	NAMESPACE
interesting-rabbit	1       	Fri Oct 25 10:54:01 2019	DEPLOYED	hello-helm-0.1.0	1.0        	defaul
```

##  7.4 Helmrepo更改
```
[root@k8s-master ~]# helm repo remove stable
[root@k8s-master ~]# helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
[root@k8s-master ~]# helm repo update
[root@k8s-master ~]# helm repo list
NAME  	URL                                                   
stable	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
local 	http://127.0.0.1:8879/charts 
```

##  7.5 Helm模板内置函数和Value
### 7.5.1 Helm 创建模板
```
[root@k8s-master helm]# helm  create mychart
Creating mychart
[root@k8s-master helm]# cat mychart/templates/configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
[root@k8s-master helm]# helm install ./mychart/
NAME:   ungaged-lizard
LAST DEPLOYED: Fri Oct 25 14:50:39 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME               DATA  AGE
mychart-configmap  1     0s

#   查看
[root@k8s-master helm]# helm  list
NAME              	REVISION	UPDATED                 	STATUS  	CHART           	APP VERSION	NAMESPACE
interesting-rabbit	1       	Fri Oct 25 10:54:01 2019	DEPLOYED	hello-helm-0.1.0	1.0        	default  
ungaged-lizard    	1       	Fri Oct 25 14:50:39 2019	DEPLOYED	mychart-0.1.0   	1.0        	default  
[root@k8s-master helm]# helm get manifest ungaged-lizard

---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
```

###  7.5.2 添加简单的模板
我们可以看到上⾯我们定义的 ConfigMap 的名字是固定的，但往往这并不是⼀种很好的做法，我们可
以通过插⼊ release 的名称来⽣成资源的名称，⽐如这⾥ ConfigMap 
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```
-   调试 
```
[root@k8s-master helm]# helm install --dry-run --debug ./mychart
```

###  7.5.3 内置对象
-   Release：这个对象描述了 release 本身。它⾥⾯有⼏个对象
```
Release.Name：release 名称
Release.Time：release 的时间
Release.Namespace：release 的 namespace（如果清单未覆盖）
Release.Service：release 服务的名称（始终是 Tiller）。
Release.Revision：此 release 的修订版本号，从1开始累加。
Release.IsUpgrade：如果当前操作是升级或回滚，则将其设置为 true。
Release.IsInstall：如果当前操作是安装，则设置为 true。
```
-   Values：从 values.yaml ⽂件和⽤户提供的⽂件传⼊模板的值。默认情况下，Values 是空的.：
-   Chart： Chart.yaml ⽂件的内容。所有的 Chart 对象都将从该⽂件中获取。chart 指南中Charts Guide列出了可⽤字段，可以前往查看。
-   Files：这提供对chart中所有⾮特殊⽂件的访问。虽然⽆法使⽤它来访问模板，但可以使⽤它来访问 chart 中的其他⽂件。请参阅 "访问⽂件" 部分。
```
Files.Get 是⼀个按名称获取⽂件的函数（.Files.Get config.ini）
Files.GetBytes 是将⽂件内容作为字节数组⽽不是字符串获取的函数。这对于像图⽚这样的东
⻄很有⽤。
```
-   Capabilities：这提供了关于 Kubernetes 集群⽀持的功能的信息。
```
Capabilities.APIVersions 是⼀组版本信息。
Helm 模板之内置函数和Values
280
Capabilities.APIVersions.Has $version 指示是否在群集上启⽤版本（batch/v1）。
Capabilities.KubeVersion 提供了查找 Kubernetes 版本的⽅法。它具有以下值：Major，
Minor，GitVersion，GitCommit，GitTreeState，BuildDate，GoVersion，Compiler，和
Platform。
Capabilities.TillerVersion 提供了查找 Tiller 版本的⽅法。它具有以下值：SemVer，
GitCommit，和 GitTreeState。
```
-   Template：包含有关正在执⾏的当前模板的信息
-   Name：到当前模板的⽂件路径（例如 mychart/templates/mytemplate.yaml）
-   BasePath：当前 chart 模板⽬录的路径（例如 mychart/templates）.
修改configmap.yaml
```
[root@k8s-master helm]# cat mychart/templates/configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  course: {{ .Values.course }}
[root@k8s-master helm]# helm install --dry-run --debug ./mychart
```
临时设置：
```
[root@k8s-master helm]# helm install --dry-run --debug --set course=python ./mychart
```

### 7.5.4 Helm模板之模板函数与管道
-   quote函数：转换成字符串
```
course: {{ quote .Values.course }}
```
-   repeat函数：
```
k8s: {{ .Values.course | repeat 3 | quote }}
```
-   default函数：
```
myvalue: {{ .Values.hello | default "Hello World" | quote }}
```
-   管道：
```
course: {{ .Values.course|upper |quote }}
```

### 7.5.5 Helm模板之控制流程
```
[root@k8s-master mychart]# cat templates/configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: {{ .Values.hello | default "Hello World" | quote }}
  course: {{ .Values.course|upper |quote }}
  project: {{ .Values.python | repeat 3 | quote }}
  {{- if eq .Values.python "django" }}
  web: true
  {{- end }}
```

### 7.5.6 使用方法
```
hfq-staging-5机器
cd /data/helm/
安装命令
helm install --name app-market-dev ./mycharts --set "project=app-market,env=dev,port=12334,tag=39"
更新命令
devops@hfq-staging-5:/data/helm$ helm upgrade app-market-dev ./mycharts --set "project=app-market,env=dev,port=12334,tag=39"
测试生成的yaml
helm install --dry-run  --debug --name adsdv ./mycharts --set "project=app-market,env=dev,port=12334,tag=39"
```
