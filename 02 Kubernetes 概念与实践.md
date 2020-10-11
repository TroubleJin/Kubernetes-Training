- [1. Kubernetes 概念](#1-Kubernetes概念)
  * [1.1 纯容器模式的问题](#11-纯容器模式的问题)
  * [1.2 容器调度管理平台](#12-容器调度管理平台)
  * [1.3 架构图](#13-架构图)
  * [1.4 核心组件](#14-核心组件)
  * [1.5 工作流程](#15-工作流程)
  * [1.6 架构设计的几点思考](#16-架构设计的几点思考)
- [2. Kubernetes 实践](#2-kubernetes实践)
  * [2.1 Kubernetes  查看组件](#21-kubernetes查看组件)
  * [2.2 理解集群资源](#22-理解集群资源)
  * [2.3 namespace](#23-namespace)
  * [2.4 实践--使用k8s管理业务应用](#24-实践--使用k8s管理业务应用)
  * [2.5 使用yaml格式定义Pod](#25-使用yaml格式定义Pod)

#   1. Kubernetes概念
##  1.1 纯容器模式的问题

1. 业务容器数量庞大，哪些容器部署在哪些节点，使用了哪些端口，如何记录、管理，需要登录到每台机器去管理？
2. 跨主机通信，多个机器中的容器之间相互调用如何做，iptables规则手动维护？
3. 跨主机容器间互相调用，配置如何写？写死固定IP+端口？
4. 如何实现业务高可用？多个容器对外提供服务如何实现负载均衡？
5. 容器的业务中断了，如何可以感知到，感知到以后，如何自动启动新的容器?
6. 如何实现滚动升级保证业务的连续性？

##  1.2 容器调度管理平台
三驾马车：
Docker Swarm       Mesos        Google Kubernetes

2017年开始Kubernetes凭借强大的容器集群管理功能, 逐步占据市场,目前在容器编排领域一枝独秀

 https://kubernetes.io/ 
 
##  1.3 架构图
![image](F5D7D12EE73349448C124AE9706E2205)

##  1.4 核心组件
- ETCD：分布式高性能键值数据库,存储整个集群的所有元数据
- ApiServer:  API服务器,集群资源访问控制入口,提供restAPI及安全访问控制
- Scheduler：调度器,负责把业务容器调度到最合适的Node节点
- Controller Manager：控制器管理,确保集群资源按照期望的方式运行
  - Replication Controller
  - Node controller
  - ResourceQuota Controller
  - Namespace Controller
  - ServiceAccount Controller
  - Token Controller
  - Service Controller
  - Endpoints Controller
- kubelet：运行在每个节点上的主要的“节点代理”，脏活累活
  - pod 管理：kubelet 定期从所监听的数据源获取节点上 pod/container 的期望状态（运行什么容器、运行的副本数量、网络或者存储如何配置等等），并调用对应的容器平台接口达到这个状态。
  - 容器健康检查：kubelet 创建了容器之后还要查看容器是否正常运行，如果容器运行出错，就要根据 pod 设置的重启策略进行处理.
  - 容器监控：kubelet 会监控所在节点的资源使用情况，并定时向 master 报告，资源使用数据都是通过 cAdvisor 获取的。知道整个集群所有节点的资源情况，对于 pod 的调度和正常运行至关重要
- kube-proxy：维护节点中的iptables或者ipvs规则
- kubectl: 命令行接口，用于对 Kubernetes 集群运行命令  https://kubernetes.io/zh/docs/reference/kubectl/ 

##  1.5 工作流程
![image](915E1D603CBF4D7285B8DBF9243BA15F)
1. 用户准备一个资源文件（记录了业务应用的名称、镜像地址等信息），通过调用APIServer执行创建Pod
2. APIServer收到用户的Pod创建请求，将Pod信息写入到etcd中
3. 调度器通过list-watch的方式，发现有新的pod数据，但是这个pod还没有绑定到某一个节点中
4. 调度器通过调度算法，计算出最适合该pod运行的节点，并调用APIServer，把信息更新到etcd中
5. kubelet同样通过list-watch方式，发现有新的pod调度到本机的节点了，因此调用容器运行时，去根据pod的描述信息，拉取镜像，启动容器，同时生成事件信息
6. 同时，把容器的信息、事件及状态也通过APIServer写入到etcd中

##  1.6 架构设计的几点思考
1. 系统各个组件分工明确(APIServer是所有请求入口，CM是控制中枢，Scheduler主管调度，而Kubelet负责运行)，配合流畅，整个运行机制一气呵成。
2. 除了配置管理和持久化组件ETCD，其他组件并不保存数据。意味`除ETCD外`其他组件都是无状态的。因此从架构设计上对kubernetes系统高可用部署提供了支撑。
3. 同时因为组件无状态，组件的升级，重启，故障等并不影响集群最终状态，只要组件恢复后就可以从中断处继续运行。
4. 各个组件和kube-apiserver之间的数据推送都是通过list-watch机制来实现。

#   2. Kubernetes实践
##  2.1 Kubernetes查看组件
```
[root@k8s-master ~]# kubectl get pod -n kube-system -o wide
NAME                                 READY   STATUS    RESTARTS   AGE     IP              NODE          NOMINATED NODE   READINESS GATES
coredns-58cc8c89f4-9fbh2             1/1     Running   0          2d16h   10.244.0.3      k8s-master    <none>           <none>
coredns-58cc8c89f4-gw2vj             1/1     Running   0          2d16h   10.244.0.2      k8s-master    <none>           <none>
etcd-k8s-master                      1/1     Running   0          2d16h   172.17.176.31   k8s-master    <none>           <none>
kube-apiserver-k8s-master            1/1     Running   0          2d16h   172.17.176.31   k8s-master    <none>           <none>
kube-controller-manager-k8s-master   1/1     Running   0          2d16h   172.17.176.31   k8s-master    <none>           <none>
kube-flannel-ds-amd64-lgsl2          1/1     Running   0          2d16h   172.17.176.31   k8s-master    <none>           <none>
kube-flannel-ds-amd64-s2jx2          1/1     Running   0          2d16h   172.17.176.32   k8s-slave-1   <none>           <none>
kube-flannel-ds-amd64-zvp55          1/1     Running   0          2d16h   172.17.176.33   k8s-slave-2   <none>           <none>
kube-proxy-2xhgb                     1/1     Running   0          2d16h   172.17.176.31   k8s-master    <none>           <none>
kube-proxy-4skw7                     1/1     Running   0          2d16h   172.17.176.33   k8s-slave-2   <none>           <none>
kube-proxy-txcz8                     1/1     Running   0          2d16h   172.17.176.32   k8s-slave-1   <none>           <none>
kube-scheduler-k8s-master            1/1     Running   0          2d16h   172.17.176.31   k8s-master    <none>           <none>
```

## 2.2 理解集群资源

组件是为了支撑k8s平台的运行，安装好的软件。

资源是如何去使用k8s的能力的定义。比如，k8s可以使用Pod来管理业务应用，那么Pod就是k8s集群中的一类资源，集群中的所有资源可以提供如下方式查看：
```
kubectl api-resources
```

##  2.3 namespace
命名空间，集群内一个虚拟的概念，类似于资源池的概念，一个池子里可以有各种资源类型，绝大多数的资源都必须属于某一个namespace。集群初始化安装好之后，会默认有如下几个namespace：

```powershell
$ kubectl get namespaces
NAME                   STATUS   AGE
default                Active   84m
kube-node-lease        Active   84m
kube-public            Active   84m
kube-system            Active   84m
kubernetes-dashboard   Active   71m
```
- 所有NAMESPACED的资源，在创建的时候都需要指定namespace，若不指定，默认会在default命名空间下
- 相同namespace下的同类资源不可以重名，不同类型的资源可以重名
- 不同namespace下的同类资源可以重名
- 通常在项目使用的时候，我们会创建带有业务含义的namespace来做逻辑上的整合


##  2.4 使用k8s管理业务应用        
**最小调度单元 Pod**

docker调度的是容器，在k8s集群中，最小的调度单元是Pod（豆荚）

**为什么引入Pod**

- 与容器引擎解耦

  Docker、Rkt。平台设计与引擎的具体的实现解耦

- 多容器共享网络|存储|进程 空间, 支持的业务场景更加灵活

##  2.5 使用yaml格式定义Pod
```
apiVersion: v1
kind: Pod
metadata:
  name: myblog
  namespace: luffy
  labels:
    component: myblog
spec:
  containers:
  - name: myblog
    image: 192.168.136.10:5000/myblog:v1
    env:
    - name: MYSQL_HOST   #  指定root用户的用户名
      value: "127.0.0.1"
    - name: MYSQL_PASSWD
      value: "123456"
    ports:
    - containerPort: 8002
  - name: mysql
    image: 192.168.136.10:5000/mysql:5.7-utf8
    ports:
    - containerPort: 3306
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "123456"
    - name: MYSQL_DATABASE
      value: "myblog"
```
| apiVersion | 含义                                                         |
| :--------- | :----------------------------------------------------------- |
| alpha      | 进入K8s功能的早期候选版本，可能包含Bug，最终不一定进入K8s    |
| beta       | 已经过测试的版本，最终会进入K8s，但功能、对象定义可能会发生变更。 |
| stable     | 可安全使用的稳定版本                                         |
| v1         | stable 版本之后的首个版本，包含了更多的核心对象              |
| apps/v1    | 使用最广泛的版本，像Deployment、ReplicaSets都已进入该版本    |


**资源类型与apiVersion对照表**

| Kind                  | apiVersion                              |
| :-------------------- | :-------------------------------------- |
| ClusterRoleBinding    | rbac.authorization.k8s.io/v1            |
| ClusterRole           | rbac.authorization.k8s.io/v1            |
| ConfigMap             | v1                                      |
| CronJob               | batch/v1beta1                           |
| DaemonSet             | extensions/v1beta1                      |
| Node                  | v1                                      |
| Namespace             | v1                                      |
| Secret                | v1                                      |
| PersistentVolume      | v1                                      |
| PersistentVolumeClaim | v1                                      |
| Pod                   | v1                                      |
| Deployment            | v1、apps/v1、apps/v1beta1、apps/v1beta2 |
| Service               | v1                                      |
| Ingress               | extensions/v1beta1                      |
| ReplicaSet            | apps/v1、apps/v1beta2                   |
| Job                   | batch/v1                                |
| StatefulSet           | apps/v1、apps/v1beta1、apps/v1beta2     |