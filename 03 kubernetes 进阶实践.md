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
#   4. Kubernetes认证授权
