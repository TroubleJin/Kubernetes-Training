[toc]
#   1. Kubernetes集群的日志及监控
##  1.1 k8s日志收集架构

https://kubernetes.io/docs/concepts/cluster-administration/logging/ 

![image](730825176875408AAD35144B4DF2F461)
总体分为三种方式：
- 使用在每个节点上运行的节点级日志记录代理。
- 在应用程序的 pod 中，包含专门记录日志的 sidecar 容器。
- 将日志直接从应用程序中推送到日志记录后端。

### 1.1.1 使用节点级日志代理

![image](73020318ECDC4C459DFFC85AF87AC26B)

容器日志驱动：

 https://docs.docker.com/config/containers/logging/configure/ 

查看当前的docker主机的驱动：

```powershell
$ docker info --format '{{.LoggingDriver}}'
```

json-file格式，docker会默认将标准和错误输出保存为宿主机的文件，路径为：

`/var/lib/docker/containers/<container-id>/<container-id>-json.log`

并且可以设置日志轮转：

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "labels": "production_status",
    "env": "os,customer"
  }
}
```

优势：

- 部署方便，使用DaemonSet类型控制器来部署agent即可
- 对业务应用的影响最小，没有侵入性

劣势:

- 只能收集标准和错误输出，对于容器内的文件日志，暂时收集不到

### 1.1.2 使用sidecar容器和日志代理
**方式一：sidecar** 容器将应用程序日志传送到自己的标准输出。

  思路：在pod中启动一个sidecar容器，把容器内的日志文件吐到标准输出，由宿主机中的日志收集agent进行采集。

![image](B4A1C5B0103A4D80AD4D1C7565B466FB)

  ```powershell
  $ cat count-pod.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: counter
  spec:
    containers:
    - name: count
      image: busybox
      args:
      - /bin/sh
      - -c
      - >
        i=0;
        while true;
        do
          echo "$i: $(date)" >> /var/log/1.log;
          echo "$(date) INFO $i" >> /var/log/2.log;
          i=$((i+1));
          sleep 1;
        done
      volumeMounts:
      - name: varlog
        mountPath: /var/log
    - name: count-log-1
      image: busybox
      args: [/bin/sh, -c, 'tail -n+1 -f /var/log/1.log']
      volumeMounts:
      - name: varlog
        mountPath: /var/log
    - name: count-log-2
      image: busybox
      args: [/bin/sh, -c, 'tail -n+1 -f /var/log/2.log']
      volumeMounts:
      - name: varlog
        mountPath: /var/log
    volumes:
    - name: varlog
      emptyDir: {}
      
  $ kubectl create -f counter-pod.yaml
  $ kubectl logs -f counter -c count-log-1
  ```
优势：
  - 可以实现容器内部日志收集
  - 对业务应用的侵入性不大
劣势：
  - 每个业务pod都需要做一次改造
  - 增加了一次日志的写入，对磁盘使用率有一定影响

**方式二：sidecar**

容器运行一个日志代理，配置该日志代理以便从应用容器收集日志。

![image](9A7E1E2DC4F140459323F87401F243F7)

  思路：直接在业务Pod中使用sidecar的方式启动一个日志收集的组件（比如fluentd），这样日志收集可以将容器内的日志当成本地文件来进行收取。

  优势：不用往宿主机存储日志，本地日志完全可以收集

  劣势：每个业务应用额外启动一个日志agent，带来额外的资源损耗 


### 1.1.3 从应用中直接暴露日志目录
![image](89822C49DF5847C8B853C84B8E91C4A7)

目前来讲，最建议的是采用节点级的日志代理。

方案一：自研方案，实现一个自研的日志收集agent，大致思路：

- 针对容器的标准输出及错误输出，使用常规的方式，监听宿主机中的容器输出路径即可
- 针对容器内部的日志文件
  - 在容器内配置统一的环境变量，比如LOG_COLLECT_FILES，指定好容器内待收集的日志目录及文件
  - agent启动的时候挂载docker.sock文件及磁盘的根路径
  - 监听docker的容器新建、删除事件，通过docker的api，查出容器的存储、环境变量、k8s属性等信息
  - 配置了LOG_COLLECT_FILES环境变量的容器，根据env中的日志路径找到主机中对应的文件路径，然后生成收集的配置文件
  - agent与开源日志收集工具（Fluentd或者filebeat等）配合，agent负责下发配置到收集工具中并对进程做reload


方案二：日志使用开源的Agent进行收集（EFK方案），适用范围广，可以满足绝大多数日志收集、展示的需求。

# 2. EFK架构工作流程
![image](1939425D12124A84B1FEF97FCD99DC55)

## 2.1 EFK架构工作流程
- Elasticsearch

  一个开源的分布式、Restful 风格的搜索和数据分析引擎，它的底层是开源库Apache Lucene。它可以被下面这样准确地形容：

  - 一个分布式的实时文档存储，每个字段可以被索引与搜索；
  - 一个分布式实时分析搜索引擎；
  - 能胜任上百个服务节点的扩展，并支持 PB 级别的结构化或者非结构化数据。

- Kibana

  Kibana是一个开源的分析和可视化平台，设计用于和Elasticsearch一起工作。可以通过Kibana来搜索，查看，并和存储在Elasticsearch索引中的数据进行交互。也可以轻松地执行高级数据分析，并且以各种图标、表格和地图的形式可视化数据。

- [Fluentd](https://docs.fluentd.org/)

  一个针对日志的收集、处理、转发系统。通过丰富的插件系统，可以收集来自于各种系统或应用的日志，转化为用户指定的格式后，转发到用户所指定的日志存储系统之中。 

 ![image](28A5913289574DF79AC8454D0A9D5332)

  Fluentd 通过一组给定的数据源抓取日志数据，处理后（转换成结构化的数据格式）将它们转发给其他服务，比如 Elasticsearch、对象存储、kafka等等。Fluentd 支持超过300个日志存储和分析服务，所以在这方面是非常灵活的。主要运行步骤如下

  1. 首先 Fluentd 从多个日志源获取数据
  2. 结构化并且标记这些数据
  3. 然后根据匹配的标签将数据发送到多个目标服务

##  2.2 Fluentd架构
![image](69E894319304485FB822A5904C13A3DE)
为什么推荐使用fluentd作为k8s体系的日志收集工具？

- 云原生：https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch 
- 将日志文件JSON化
![image](1D4E6224C47541DB80FA7C9AD71FF874)
- 可插拔架构设计
![image](745E6D3145734AA2B1D9B6532761C432)
- 极小的资源占用基于C和Ruby语言,30-40MB，13,000 events/second/core 
![image](52824281E24540E28015CE444FBD511A)
- 极强的可靠性
- 基于内存和本地文件的缓存
- 强大的故障转移

**fluentd事件流的生命周期及指令配置**
 https://docs.fluentd.org/v/0.12/quickstart/life-of-a-fluentd-event 

```
Input -> filter 1 -> ... -> filter N -> Buffer -> Output
```
启动命令

```
$ fluentd -c fluent.conf
```