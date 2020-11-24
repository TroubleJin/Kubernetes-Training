- [1. 基于Kubernetes的Devops](#1-基于Kubernetes的Devops)
  * [1.1 DevOps、CI、CD介绍](#11-devops-ci-cd--)
  * [1.2 Jenkins初体验](#12-Jenkins初体验)
  * [1.3 安装gitlab](#13-安装gitlab)
  * [1.4 Master-Slaves模式](#14-Master-Slaves模式)
    + [1.4.1 添加slave节点](#141-添加slave节点)
    + [1.4.2 执行java命令启动agent服务](#142-执行java命令启动agent服务)
    + [1.4.3 Jenkins定制化容器](#143-Jenkins定制化容器)
  * [1.5 流水线](#15-流水线)
    + [1.5.1 优化流水线部署](#151-优化流水线部署)
    + [1.5.2 使用k8s部署服务](#152-使用k8s部署服务)
    + [1.5.3 使用凭据管理敏感信息](#153-使用凭据管理敏感信息)
  * [1.6 多分支流水线](#16-多分支流水线)
- [2. 工具集成与Jenkinsfile实践篇](#2-工具集成与Jenkinsfile实践篇)
  * [2.1 集成Kubernetes](#21-集成Kubernetes)
  * [2.2 制作tools镜像](#22-制作tools镜像)
  * [2.3 修改Jenkinsfile](#23-修改Jenkinsfile)
  * [2.4 集成sonarQube实现代码扫描](#24-集成sonarQube实现代码扫描)
    + [2.4.1 sonarqube on kubernetes环境搭建](#241-sonarqube在kubernetes环境搭建)
  * [2.5 集成到tools镜像中](#25-集成到tools镜像中)
  
#   1.  基于Kubernetes的Devops
持续集成工具：
- Jenkins
- gitlabci
- Tekton 

本章基于k8s集群部署gitlab、sonarQube、Jenkins等工具，并把上述工具集成到Jenkins中，以Django项目和SpringBoot项目为例，通过多分支流水线及Jenkinsfile实现项目代码提交到不同的仓库分支，实现自动代码扫描、单元测试、docker容器构建、k8s服务的自动部署。
- DevOps、CI、CD介绍
- Jenkins、sonarQube、gitlab的快速部署
- Jenkins初体验
- 流水线入门及Jenkinsfile使用
- Jenkins与Kubernetes的集成
- sonarQube代码扫描与Jenkins的集成
- 实践Django项目的基于Jenkinsfile实现开发、测试环境的CI/CD

##  1.1 DevOps、CI、CD介绍
![image](./img/day05-01.png)

devops不是某一个具体的技术，而是一种思想+自动化能力，来使得构建、测试、发布软件能够更加地便捷、频繁和可靠的落地实践。本次课程核心内容就是要教会大家如何利用工具和技术来实现完整的DevOps平台的建设。我们主要使用的工具有：
1. gitlab，代码仓库，企业内部使用最多的代码版本管理工具。
2. Jenkins， 一个可扩展的持续集成引擎，用于自动化各种任务，包括构建、测试和部署软件。
3. robotFramework， 基于Python的自动化测试框架
4. sonarqube，代码质量管理平台
5. maven，java包构建管理工具
6. Kubernetes
7. Docker


##  1.2 Jenkins初体验
注意点：
1. 第一次启动很慢
2. 因为后面Jenkins会与kubernetes集群进行集成，会需要调用kubernetes集群的api，因此安装的时候创建了ServiceAccount并赋予了cluster-admin的权限
3. 默认部署到jenkins=true的节点
4. 初始化容器来设置权限
5. ingress来外部访问
6. 数据存储通过hostpath挂载到宿主机中
```
[root@k8s-master jenkins]# ls -l
总用量 8
-rw-r--r-- 1 root root 2210 11月  8 09:51 jenkins-all.yaml
-rw-r--r-- 1 root root  182 11月  8 10:02 jenkins-pvc.yaml
[root@k8s-master jenkins]# cat *
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins-crb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-master
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      devops: jenkins-master
  template:
    metadata:
      labels:
        devops: jenkins-master
    spec:
      serviceAccount: jenkins #Pod 需要使用的服务账号
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /var/jenkins_home"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: jenkinshome
          mountPath: /var/jenkins_home
      containers:
      - name: jenkins
        image: jenkinsci/blueocean:1.23.2
        imagePullPolicy: IfNotPresent
        ports:
        - name: http #Jenkins Master Web 服务端口
          containerPort: 8080
        - name: slavelistener #Jenkins Master 供未来 Slave 连接的端口
          containerPort: 50000
        volumeMounts:
        - name: jenkinshome
          mountPath: /var/jenkins_home
        env:
        - name: JAVA_OPTS
          value: "-Xms4096m -Xmx5120m -Duser.timezone=Asia/Shanghai -Dhudson.model.DirectoryBrowserSupport.CSP="
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: jenkins-data
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: slavelistener
    port: 50000
    targetPort: 50000
  type: ClusterIP
  selector:
    devops: jenkins-master
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins-web
  namespace: jenkins
spec:
  rules:
  - host: jenkins.devops.com
    http:
      paths:
      - backend:
          serviceName: jenkins
          servicePort: 8080
        path: /
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  namespace: jenkins
  name: jenkins-data
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10G
  storageClassName: nfs
```
**查看日志，第一次启动提示需要完成初始化设置**
```
$ kubectl -n jenkins logs -f jenkins-master-767df9b574-lgdr5
......
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

5396b4e1c395450f8360efd8ee641b18

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
```

点击叉号，跳过选择安装推荐的插件环节，直接进入Jenkins。由于默认的插件地址安装非常慢，我们可以替换成国内清华的源，进入 jenkins 工作目录，目录下面有一个 `updates` 的目录，下面有一个 `default.json` 文件，我们执行下面的命令替换插件地址：

```powershell
$ cd /var/jenkins_home/updates
$ sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json 
$ sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json

```

> 暂时先不用重新启动pod，汉化后一起重启。

选择右上角admin->configure->password重新设置管理员密码，设置完后，会退出要求重新登录，使用admin/xxxxxx(新密码)，登录即可。

##  1.3 安装gitlab
**gitlab代码仓库搭建**
 https://github.com/sameersbn/docker-gitlab 

```powershell
## 全量部署的组件
$ gitlab-ctl status
run: alertmanager: (pid 1987) 27s; run: log: (pid 1986) 27s
run: gitaly: (pid 1950) 28s; run: log: (pid 1949) 28s
run: gitlab-exporter: (pid 1985) 27s; run: log: (pid 1984) 27s
run: gitlab-workhorse: (pid 1956) 28s; run: log: (pid 1955) 28s
run: logrotate: (pid 1960) 28s; run: log: (pid 1959) 28s
run: nginx: (pid 2439) 1s; run: log: (pid 1990) 27s
run: node-exporter: (pid 1963) 28s; run: log: (pid 1962) 28s
run: postgres-exporter: (pid 1989) 27s; run: log: (pid 1988) 27s
run: postgresql: (pid 1945) 28s; run: log: (pid 1944) 28s
run: prometheus: (pid 1973) 28s; run: log: (pid 1972) 28s
run: puma: (pid 1968) 28s; run: log: (pid 1966) 28s
run: redis: (pid 1952) 28s; run: log: (pid 1951) 28s
run: redis-exporter: (pid 1971) 28s; run: log: (pid 1964) 28s
run: sidekiq: (pid 1969) 28s; run: log: (pid 1967) 28s
```

部署分析：

1. 依赖postgres
2. 依赖redis

-   准备secret文件
```
[root@k8s-master gitlab]# kubectl -n jenkins create secret generic gitlab-secret --from-env-file=gitlab-secret.txt
secret/gitlab-secret created
[root@k8s-master gitlab]# cat gitlab-secret.txt
postgres.user.root=root
postgres.pwd.root=1qaz2wsx
```
-   部署postgres

注意点：

- 使用secret来引用账户密码
```
[root@k8s-master gitlab]# cat postgres-pvc.yaml postgres.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  namespace: jenkins
  name: postgredb-data
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10G
  storageClassName: nfs
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
  namespace: jenkins
spec:
  ports:
  - name: server
    port: 5432
    targetPort: 5432
    protocol: TCP
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: jenkins
  name: postgres
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image:  172.17.176.31:5000/postgres:11.4 #若本地没有启动该仓库，换成postgres:11.4
        imagePullPolicy: "IfNotPresent"
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER           #PostgreSQL 用户名
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.user.root
        - name: POSTGRES_PASSWORD       #PostgreSQL 密码
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.pwd.root
        resources:
          limits:
            cpu: 1000m
            memory: 2048Mi
          requests:
            cpu: 50m
            memory: 100Mi
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgredb
      volumes:
      - name: postgredb
        persistentVolumeClaim:
          claimName: postgredb-data
```

**创建gitlab库**
```
[root@k8s-master gitlab]# kubectl -n jenkins get pods
NAME                              READY   STATUS    RESTARTS   AGE
jenkins-master-6b87f5bfc8-qzg9r   1/1     Running   0          2d
postgres-9b964c44-kfqr4           1/1     Running   0          4m51s
[root@k8s-master gitlab]# kubectl -n jenkins exec -it postgres-9b964c44-kfqr4 sh
# psql
psql (11.4 (Debian 11.4-1.pgdg90+1))
Type "help" for help.

root=# create database gitlab;
CREATE DATABASE
```

**部署redis**
```
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
  namespace: jenkins
spec:
  ports:
  - name: server
    port: 6379
    targetPort: 6379
    protocol: TCP
  selector:
    app: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: jenkins
  name: redis
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      tolerations:
      - operator: "Exists"
      containers:
      - name: redis
        image:  sameersbn/redis:4.0.9-2
        imagePullPolicy: "IfNotPresent"
        ports:
        - containerPort: 6379
        resources:
          limits:
            cpu: 1000m
            memory: 2048Mi
          requests:
            cpu: 50m
            memory: 100Mi
```

**搭建gitlab**
```
[root@k8s-master gitlab]# cat gitlab.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gitlab
  namespace: jenkins
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  rules:
  - host: gitlab.devops.com
    http:
      paths:
      - backend:
          serviceName: gitlab
          servicePort: 80
        path: /
---
apiVersion: v1
kind: Service
metadata:
  name: gitlab
  labels:
    app: gitlab
  namespace: jenkins
spec:
  ports:
  - name: server
    port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: gitlab
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  namespace: jenkins
  name: gitlab-data
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10G
  storageClassName: nfs
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: jenkins
  name: gitlab
  labels:
    app: gitlab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitlab
  template:
    metadata:
      labels:
        app: gitlab
    spec:
      containers:
      - name: gitlab
        image:  sameersbn/gitlab:13.2.2
        imagePullPolicy: "IfNotPresent"
        env:
        - name: GITLAB_HOST
          value: "gitlab.luffy.com"
        - name: GITLAB_PORT
          value: "80"
        - name: GITLAB_SECRETS_DB_KEY_BASE
          value: "long-and-random-alpha-numeric-string"
        - name: GITLAB_SECRETS_DB_KEY_BASE
          value: "long-and-random-alpha-numeric-string"
        - name: GITLAB_SECRETS_SECRET_KEY_BASE
          value: "long-and-random-alpha-numeric-string"
        - name: GITLAB_SECRETS_OTP_KEY_BASE
          value: "long-and-random-alpha-numeric-string"
        - name: DB_HOST
          value: "postgres"
        - name: DB_NAME
          value: "gitlab"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.user.root
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.pwd.root
        - name: REDIS_HOST
          value: "redis"
        - name: REDIS_PORT
          value: "6379"
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 2000m
            memory: 5048Mi
          requests:
            cpu: 100m
            memory: 500Mi
        volumeMounts:
        - mountPath: /home/git/data
          name: data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: gitlab-data
```
**代码提交到仓库即可**
-   测试钉钉推送
[官方文档](https://ding-doc.dingtalk.com/doc#/serverapi2/qf2nxq)
- 配置机器人
- 试验发送消息

```
$ curl 'https://oapi.dingtalk.com/robot/send?access_token=b3a3e118a02e9223645659edec95f5aabfe57a3f2d339b5f43b10f3bb0e604c4' \
   -H 'Content-Type: application/json' \
   -d '{"msgtype": "text", 
        "text": {
             "content": "我就是我, 是不一样的烟火"
        }
      }'
```
1. 安装gitlab plugin

   插件中心搜索并安装gitlab，直接安装即可

2. 配置Gitlab

   系统管理->系统配置->Gitlab，其中的API Token，需要从下个步骤中获取

3. 获取AccessToken

   登录gitlab，选择user->Settings->access tokens新建一个访问token
![image](./img/day05-02.png)
4. 配置host解析

   由于我们的Jenkins和gitlab域名是本地解析，因此需要让gitlab和Jenkins服务可以解析到对方的域名。两种方式：

   - 在容器内配置hosts

   - 配置coredns的静态解析

     ```powershell
        hosts {
            172.17.176.31 jenkins.devops.com  gitlab.devops.com
            fallthrough
        }
     ```

5. 创建自由风格项目

   - gitlab connection 选择为刚创建的gitlab
   - 源码管理选择Git，填项项目地址
   - 新建一个 Credentials 认证，使用用户名密码方式，配置gitlab的用户和密码
   - 构建触发器选择 Build when a change is pushed to GitLab 
   - 生成一个Secret token
   - 保存

![image](./img/day05-03.png)
6. 到gitlab配置webhook

   - 进入项目下settings->Integrations
   - URL： http://jenkins.devops.com/project/myblog 
   - Secret Token 填入在Jenkins端生成的token
   - Add webhook
   - test push events，报错：Requests to the local network are not allowed 


![image](./img/day05-04.png)
7. 设置gitlab允许向本地网络发送webhook请求

   访问 Admin Aera -> Settings -> Network ，展开Outbound requests

   Collapse，勾选第一项即可。再次test push events，成功。

![image](./img/day05-05.png)   

8. 配置free项目，增加构建步骤，执行shell，将发送钉钉消息的shell保存

9. 提交代码到gitlab仓库，查看构建是否自动执行


##  1.4 Master-Slaves模式
### 1.4.1 添加slave节点

- 系统管理 -> 节点管理 -> 新建节点
- 比如添加172.17.176.32，选择固定节点，保存
- 远程工作目录/opt/jenkins_jobs
- 标签为任务选择节点的依据，如172.17.176.32
- 启动方式选择通过java web启动代理，代理是运行jar包，通过JNLP（是一种允许客户端启动托管在远程Web服务器上的应用程序的协议 ）启动连接到master节点服务中
执行java命令启动agent服务
![image](./img/day05-06.png)

### 1.4.2 执行java命令启动agent服务

```
[root@k8s-master python-demo]# kubectl -n jenkins get svc jenkins
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)              AGE
jenkins   ClusterIP   10.100.40.112   <none>        8080/TCP,50000/TCP   3d12h

#   回到slave-1节点
[root@k8s-slave-1 ~]# wget 10.100.40.112:8080/jnlpJars/agent.jar
[root@k8s-slave-1 ~]# mkdir -p /opt/jenkins_jobs
[root@k8s-slave-1 ~]# java -jar agent.jar -jnlpUrl http://jenkins.devops.com/computer/k8s-slave-1/slave-agent.jnlp -secret 28722ad74d0a4e77d92b6f304b8ff368c135c8722a2ba04255cde9ed8ac86613 -workDir "/opt/jenkins_jobs"
```

![image](./img/day05-07.png)

###  1.4.3 Jenkins定制化容器

由于每次新部署Jenkins环境，均需要安装很多必要的插件，因此考虑把插件提前做到镜像中

*Dockerfile*

```dockerfile
FROM jenkinsci/blueocean:1.23.2

## 用最新的插件列表文件替换默认插件文件
COPY plugins.txt /usr/share/jenkins/ref/

## 执行插件安装
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

```

*plugins.txt*

```python
ace-editor:1.1
allure-jenkins-plugin:2.28.1
ant:1.10
antisamy-markup-formatter:1.6
apache-httpcomponents-client-4-api:4.5.10-1.0
authentication-tokens:1.3
...

```

*get_plugin.sh*

> admin:123456@localhost 需要替换成Jenkins的用户名、密码及访问地址

```powershell
#!/usr/bin/env bash
curl -sSL  "http://admin:123456@localhost:8080/pluginManager/api/xml?depth=1&xpath=/*/*/shortName|/*/*/version&wrapper=plugins" | perl -pe 's/.*?<shortName>([\w-]+).*?<version>([^<]+)()(<\/\w+>)+/\1:\2\n/g'|sed 's/ /:/' > plugins.txt

```

```powershell
## 执行构建，定制jenkins容器
$ docker build . -t 172.21.51.67:5000/jenkins:v20200414 -f Dockerfile
$ docker push 172.21.51.67:5000/jenkins:v20200414

```

至此，我们可以使用定制化的镜像启动jenkins服务

```powershell
## 删掉当前服务
$ kubectl delete -f jenkins-all.yaml

## 删掉已挂载的数据
$ rm -rf /var/jenkins_home

## 替换使用定制化镜像
$ sed -i 's#jenkinsci/blueocean#172.21.51.67:5000/jenkins:v20200404#g' jenkins-all.yaml

## 重新创建服务
$ kubectl create -f jenkins-all.yaml
```

##  1.5 流水线
[官方文档](https://jenkins.io/zh/doc/book/pipeline/getting-started/)
两种语法类型：

- Scripted Pipeline，脚本式流水线，最初支持的类型
- Declarative Pipeline，声明式流水线，为Pipeline plugin在2.5版本之后新增的一种脚本类型，后续Open Blue Ocean所支持的类型。与原先的Scripted Pipeline一样，都可以用来编写脚本。Declarative Pipeline 是后续Open Blue Ocean所支持的类型，写法简单，支持内嵌Scripted Pipeline代码

*为与BlueOcean脚本编辑器兼容，通常建议使用Declarative Pipeline的方式进行编写,从jenkins社区的动向来看，很明显这种语法结构也会是未来的趋势。*

```
pipeline { 
    agent {label '172.17.176.32'}
    environment { 
        PROJECT = 'myblog'
    }
    stages {
        stage('Checkout') { 
            steps { 
                checkout scm 
            }
        }
        stage('Build') { 
            steps { 
                sh 'make' 
            }
        }
        stage('Test'){
            steps {
                sh 'make check'
                junit 'reports/**/*.xml' 
            }
        }
        stage('Deploy') {
            steps {
                sh 'make publish'
            }
        }
    }
	post {
        success { 
            echo 'Congratulations!'
        }
		failure { 
            echo 'Oh no!'
        }
        always { 
            echo 'I will always say Hello again!'
        }
    }
}
```

脚本解释：

- `checkout`步骤为检出代码; `scm`是一个特殊变量，指示`checkout`步骤克隆触发此Pipeline运行的特定修订 

- agent：指明使用哪个agent节点来执行任务，定义于pipeline顶层或者stage内部

  - any，可以使用任意可用的agent来执行

  - label，在提供了标签的 Jenkins 环境中可用的代理上执行流水线或阶段。 例如: `agent { label 'my-defined-label' }`，最常见的使用方式

  - none，当在 `pipeline` 块的顶部没有全局代理， 该参数将会被分配到整个流水线的运行中并且每个 `stage` 部分都需要包含他自己的 `agent` 部分。比如: `agent none`

  - docker， 使用给定的容器执行流水线或阶段。 在指定的节点中，通过运行容器来执行任务

    ```json
    agent {
        docker {
            image 'maven:3-alpine'
            label 'my-defined-label'
            args  '-v /tmp:/tmp'
        }
    }
    
    ```

- options:  允许从流水线内部配置特定于流水线的选项。  

  -  buildDiscarder , 为最近的流水线运行的特定数量保存组件和控制台输出。例如: `options { buildDiscarder(logRotator(numToKeepStr: '10')) }` 
  -  disableConcurrentBuilds ,不允许同时执行流水线。 可被用来防止同时访问共享资源等。 例如: `options { disableConcurrentBuilds() }`
  -  timeout ,设置流水线运行的超时时间, 在此之后，Jenkins将中止流水线。例如: `options { timeout(time: 1, unit: 'HOURS') }`
  -  retry，在失败时, 重新尝试整个流水线的指定次数。 For example: `options { retry(3) }`

- environment:  指令制定一个 键-值对序列，该序列将被定义为所有步骤的环境变量 

- stages: 包含一系列一个或多个 [stage](https://jenkins.io/zh/doc/book/pipeline/syntax/#stage)指令, `stages` 部分是流水线描述的大部分"work" 的位置。 建议 `stages` 至少包含一个 [stage](https://jenkins.io/zh/doc/book/pipeline/syntax/#stage) 指令用于连续交付过程的每个离散部分,比如构建, 测试, 和部署。 

  ```groovy
  pipeline {
      agent any
      stages { 
          stage('Example') {
              steps {
                  echo 'Hello World'
              }
          }
      }
  }
  
  ```

- steps: 在给定的 `stage` 指令中执行的定义了一系列的一个或多个[steps](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-steps)。

- post:  定义一个或多个[steps](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-steps) ，这些阶段根据流水线或阶段的完成情况而运行`post` 支持以下 [post-condition](https://jenkins.io/zh/doc/book/pipeline/syntax/#post-conditions) 块中的其中之一: `always`, `changed`, `failure`, `success`, `unstable`, 和 `aborted`。

  -  always, 无论流水线或阶段的完成状态如何，都允许在 `post` 部分运行该步骤
  -  changed, 当前流水线或阶段的完成状态与它之前的运行不同时，才允许在 `post` 部分运行该步骤
  -  failure,  当前流水线或阶段的完成状态为"failure"，才允许在 `post` 部分运行该步骤, 通常web UI是红色
  -  success,  当前流水线或阶段的完成状态为"success"，才允许在 `post` 部分运行该步骤, 通常web UI是蓝色或绿色
  -  unstable,  当前流水线或阶段的完成状态为"unstable"，才允许在 `post` 部分运行该步骤, 通常由于测试失败,代码违规等造成。通常web UI是黄色 
  -  aborted， 只有当前流水线或阶段的完成状态为"aborted"，才允许在 `post` 部分运行该步骤, 通常由于流水线被手动的aborted。通常web UI是灰色 
  -

**测试Jenkinsfile**
```powershell
pipeline {
   agent { label '172.17.176.32'}

   stages {
      stage('printenv') {
         steps {
            echo 'Hello World'
            sh 'printenv'
         }
      }
      stage('check') {
         steps {
            checkout([$class: 'GitSCM', branches: [[name: 'master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'git', url: 'http://gitlab.devops.com/root/myblog.git']]])
         }
      }
      stage('build-image') {
         steps {
            retry(2) { sh 'docker build . -t myblog:latest'}
         }
      }
      stage('send-msg') {
         steps {
            sh """
            curl 'https://oapi.dingtalk.com/robot/send?access_token=67e81175c6ebacb1307e83f62680f36fbcf4524e8f43971cf2fb2049bc58723d' \
   -H 'Content-Type: application/json' \
   -d '{"msgtype": "text", 
        "text": {
             "content": "我就是我, 是不一样的烟火"
        }
      }'
      """
         }
      }
   }
}

```

###  1.5.1   优化流水线部署
- 优化代码检出阶段

  由于目前已经配置了使用git仓库地址，且使用SCM来检测项目，因此代码检出阶段完全没有必要再去指定一次 

- 构建镜像的tag使用git的commit id

- 增加post阶段的消息通知，丰富通知内容

- 配置webhook，实现myblog代码推送后，触发Jenkinsfile任务执行

![image](./img/day05-08.png)

```
pipeline {
    agent { label '172.17.176.32'}

    stages {
        stage('printenv') {
            steps {
            echo 'Hello World'
            sh 'printenv'
            }
        }
        stage('check') {
            steps {
                checkout scm
            }
        }
        stage('build-image') {
            steps {
            	retry(2) { sh 'docker build . -t myblog:${GIT_COMMIT}'}
            }
        }
    }
    post {
        success {
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=b3a3e118a02e9223645659edec95f5aabfe57a3f2d339b5f43b10f3bb0e604c4' \
                    -H 'Content-Type: application/json' \
                    -d '{"msgtype": "text",
                            "text": {
                                "content": "😄👍构建成功👍😄\n 关键字：luffdevops\n 项目名称: ${JOB_BASE_NAME}\n Commit Id: ${GIT_COMMIT}\n 构建地址：${RUN_DISPLAY_URL}"
                        }
                }'
            """
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=b3a3e118a02e9223645659edec95f5aabfe57a3f2d339b5f43b10f3bb0e604c4' \
                    -H 'Content-Type: application/json' \
                    -d '{"msgtype": "text",
                            "text": {
                                "content": "😖❌构建失败❌😖\n 关键字：luffdevops\n 项目名称: ${JOB_BASE_NAME}\n Commit Id: ${GIT_COMMIT}\n 构建地址：${RUN_DISPLAY_URL}"
                        }
                }'
            """
        }
        always {
            echo 'I will always say Hello again!'
        }
    }
}
```

### 1.5.2 使用k8s部署服务
- 新建deploy目录，将k8s所需的文件放到deploy目录中
- 将镜像地址改成模板，在pipeline中使用新构建的镜像进行替换
- 执行kubectl apply -f deploy应用更改，需要配置kubectl认证
```
pipeline {
    agent { label '172.17.176.32'}

    environment {
        IMAGE_REPO = "172.17.176.31:5000/myblog"
    }

    stages {
        stage('printenv') {
            steps {
              echo 'Hello World'
              sh 'printenv'
            }
        }
        stage('check') {
            steps {
                checkout scm
            }
        }
        stage('build-image') {
            steps {
                retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
            }
        }
        stage('push-image') {
            steps {
                retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
            }
        }
        stage('deploy') {
            steps {
                sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' deploy/*"
                timeout(time: 1, unit: 'MINUTES') {
                    sh "kubectl apply -f deploy/"
                }
            }
        }
    }
    post {
        success {
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=b3a3e118a02e9223645659edec95f5aabfe57a3f2d339b5f43b10f3bb0e604c4' \
                    -H 'Content-Type: application/json' \
                    -d '{"msgtype": "text",
                            "text": {
                                "content": "😄👍构建成功👍😄\n 关键字：myblog\n 项目名称: ${JOB_BASE_NAME}\n Commit Id: ${GIT_COMMIT}\n 构建地址：${RUN_DISPLAY_URL}"
                        }
                }'
            """
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=b3a3e118a02e9223645659edec95f5aabfe57a3f2d339b5f43b10f3bb0e604c4' \
                    -H 'Content-Type: application/json' \
                    -d '{"msgtype": "text",
                            "text": {
                                "content": "😖❌构建失败❌😖\n 关键字：luffy\n 项目名称: ${JOB_BASE_NAME}\n Commit Id: ${GIT_COMMIT}\n 构建地址：${RUN_DISPLAY_URL}"
                        }
                }'
            """
        }
        always {
            echo 'I will always say Hello again!'
        }
    }
}
```

### 1.5.3 使用凭据管理敏感信息
上述Jenkinsfile中存在的问题是敏感信息使用明文，暴漏在代码中，如何管理流水线中的敏感信息（包含账号密码），之前我们在对接gitlab的时候，需要账号密码，已经使用过凭据来管理这类敏感信息，同样的，我们可以使用凭据来存储钉钉的token信息，那么，创建好凭据后，如何在Jenkinsfile中获取已有凭据的内容？

 Jenkins 的声明式流水线语法有一个 `credentials()` 辅助方法（在[`environment`](https://jenkins.io/zh/doc/book/pipeline/jenkinsfile/#../syntax#environment) 指令中使用），它支持 [secret 文本](https://jenkins.io/zh/doc/book/pipeline/jenkinsfile/##secret-text)，[带密码的用户名](https://jenkins.io/zh/doc/book/pipeline/jenkinsfile/##usernames-and-passwords)，以及 [secret 文件](https://jenkins.io/zh/doc/book/pipeline/jenkinsfile/##secret-files)凭据。 

下面的流水线代码片段展示了如何创建一个使用带密码的用户名凭据的环境变量的流水线。

在该示例中，带密码的用户名凭据被分配了环境变量，用来使你的组织或团队以一个公用账户访问 Bitbucket 仓库；这些凭据已在 Jenkins 中配置了凭据 ID `jenkins-bitbucket-common-creds`。
这实际设置了下面的三个环境变量：
- `BITBUCKET_COMMON_CREDS` - 包含一个以冒号分隔的用户名和密码，格式为 `username:password`。
- `BITBUCKET_COMMON_CREDS_USR` - 附加的一个仅包含用户名部分的变量。
- `BITBUCKET_COMMON_CREDS_PSW` - 附加的一个仅包含密码部分的变量。

```
pipeline {
    agent { label '172.17.176.32'}

    environment {
        IMAGE_REPO = "172.17.176.31:5000/myblog"
        DINGTALK_CREDS = credentials('dingding-ops-token')
    }

    stages {
        stage('printenv') {
            steps {
              echo 'Hello World'
              sh 'printenv'
            }
        }
        stage('check') {
            steps {
                checkout scm
            }
        }
        stage('build-image') {
            steps {
                retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
            }
        }
        stage('push-image') {
            steps {
                retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
            }
        }
        stage('deploy') {
            steps {
                sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' deploy/*"
                timeout(time: 1, unit: 'MINUTES') {
                    sh "kubectl apply -f deploy/"
                }
            }
        }
    }
    post {
        success {
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS}' \
                    -H 'Content-Type: application/json' \
                    -d '{"msgtype": "text",
                            "text": {
                                "content": "😄👍构建成功👍😄\n 关键字：myblog\n 项目名称: ${JOB_BASE_NAME}\n Commit Id: ${GIT_COMMIT}\n 构建地址：${RUN_DISPLAY_URL}"
                        }
                }'
            """
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS}' \
                    -H 'Content-Type: application/json' \
                    -d '{"msgtype": "text",
                            "text": {
                                "content": "😖❌构建失败❌😖\n 关键字：luffy\n 项目名称: ${JOB_BASE_NAME}\n Commit Id: ${GIT_COMMIT}\n 构建地址：${RUN_DISPLAY_URL}"
                        }
                }'
            """
        }
        always {
            echo 'I will always say Hello again!'
        }
    }
}
```

##  1.6 多分支流水线
Jenkins端创建多分支流水线项目
- 增加git分支源
- 发现标签
- 根据名称过滤，develop|master|v.*
- 高级克隆，设置浅克隆
保存后，会自动检索项目中所有存在Jenkinsfile文件的分支和标签，若匹配我们设置的过滤正则表达式，则会添加到多分支的构建视图中。所有添加到视图中的分支和标签，会默认执行一次构建任务。

![image](./img/day05-09.png)
```
pipeline {
    agent { label '172.17.176.32'}

    options {
		buildDiscarder(logRotator(numToKeepStr: '10'))
		disableConcurrentBuilds()
		timeout(time: 20, unit: 'MINUTES')
		gitLabConnection('gitlab')
	}

    environment {
        IMAGE_REPO = "172.17.176.31:5000/demo/myblog"
        DINGTALK_CREDS = credentials('dingding-ops-token')
        TAB_STR = "\n                    \n&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"
    }

    stages {
        stage('printenv') {
            steps {
                script{
                    sh "git log --oneline -n 1 > gitlog.file"
                    env.GIT_LOG = readFile("gitlog.file").trim()
                }
                sh 'printenv'
            }
        }
        stage('checkout') {
            steps {
                checkout scm
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS = env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('build-image') {
            steps {
                retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('push-image') {
            steps {
                retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('deploy') {
            steps {
                sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' deploy/*"
                timeout(time: 1, unit: 'MINUTES') {
                    sh "kubectl apply -f deploy/"
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
    }
    post {
        success {
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😄👍 构建成功 👍😄  \n**项目名称**：luffdevops**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}   \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😖❌ 构建失败 ❌😖  \n**项目名称**：luffdevops**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}  \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """
        }
        always {
             cleanWs()
        }
    }
}
```

#  2. 工具集成与Jenkinsfile实践篇
1. Jenkins如何对接kubernetes集群
2. 使用kubernetes的Pod-Template来作为动态的agent执行Jenkins任务
3. 如何制作agent容器实现不同类型的业务的集成
4. 集成代码扫描、docker镜像自动构建、k8s服务部署、自动化测试

## 2.1 集成Kubernetes

[插件官方文档](https://plugins.jenkins.io/kubernetes/)

1. [系统管理] -> [插件管理] -> [搜索kubernetes]->直接安装

   若安装失败，请先更新[ bouncycastle API Plugin](https://plugins.jenkins.io/bouncycastle-api)并重新启动Jenkins

2. [系统管理] -> [系统配置] -> [Add a new cloud]

3. 配置地址信息

   -  Kubernetes 地址: https://kubernetes.default（或者https://172.17.176.31:6443）
   -  Kubernetes 命名空间：jenkins
   -  服务证书不用写（我们在安装Jenkins的时候已经指定过serviceAccount），均使用默认
   -  连接测试，成功会提示：Connection test successful
   -  Jenkins地址：http://jenkins:8080
   -  Jenkins 通道 ：jenkins:50000

4. 配置Pod Template

   - 名称：jnlp-slave
   - 命名空间：jenkins
   - 标签列表：jnlp-slave，作为agent的label选择用
   - 连接 Jenkins 的超时时间（秒） ：300，设置连接jenkins超时时间
   - 节点选择器：agent=true
   - 工作空间卷：选择hostpath，设置/opt/jenkins_jobs/,注意需要设置chown -R 1000:1000 /opt/jenkins_jobs/权限，否则Pod没有权限
 
![image](./img/day05-10.png)

##  2.2 制作tools镜像
```
[root@k8s-master tools]# cp `which kubectl` .
[root@k8s-master tools]# cp ~/.kube/config .
[root@k8s-master tools]# vim Dockerfile
[root@k8s-master tools]# cat Dockerfile
FROM alpine
USER root

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories && \
    apk update && \
    apk add  --no-cache openrc docker git curl tar gcc g++ make \
    bash shadow openjdk8 python2 python2-dev py-pip python3-dev openssl-dev libffi-dev \
    libstdc++ harfbuzz nss freetype ttf-freefont && \
    mkdir -p /root/.kube && \
    usermod -a -G docker root

COPY config /root/.kube/

RUN rm -rf /var/cache/apk/*
#-----------------安装 kubectl--------------------#
COPY kubectl /usr/local/bin/
RUN chmod +x /usr/local/bin/kubectl
# ------------------------------------------------#

[root@k8s-master tools]# docker build . -t 172.17.176.31:5000/devops/tools:v1
[root@k8s-master tools]# docker push 172.17.176.31:5000/devops/tools:v1
#   测试镜像
docker run -v /var/run/docker.sock:/var/run/docker.sock --rm -ti 172.17.176.31:5000/devops/tools:v1 bash
```


![image](./img/day05-11.png)

##  2.3 修改Jenkinsfile
```
pipeline {
    agent { label 'jnlp-slave'}

    options {
		buildDiscarder(logRotator(numToKeepStr: '10'))
		disableConcurrentBuilds()
		timeout(time: 20, unit: 'MINUTES')
		gitLabConnection('gitlab')
	}

    environment {
        IMAGE_REPO = "172.17.176.31:5000/demo/myblog"
        DINGTALK_CREDS = credentials('dingding-ops-token')
        TAB_STR = "\n                    \n&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"
    }

    stages {
        stage('printenv') {
            steps {
                script{
                    sh "git log --oneline -n 1 > gitlog.file"
                    env.GIT_LOG = readFile("gitlog.file").trim()
                }
                sh 'printenv'
            }
        }
        stage('checkout') {
            steps {
                container('tools') {
                    checkout scm
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS = env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('build-image') {
            steps {
                container('tools') {
                     retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('push-image') {
            steps {
                 container('tools') {
                    retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('deploy') {
            steps {
                container('tools') {
                    sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' deploy/*"
                    timeout(time: 1, unit: 'MINUTES') {
                        sh "kubectl apply -f deploy/"
                    }
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
    }
    post {
        success {
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😄👍 构建成功 👍😄  \n**项目名称**：luffdevops**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}   \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😖❌ 构建失败 ❌😖  \n**项目名称**：luffdevops**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}  \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """
        }
        always {
             cleanWs()
        }
    }
}
```

##  2.4 集成sonarQube实现代码扫描
Sonar可以从以下七个维度检测代码质量，而作为开发人员至少需要处理前5种代码质量问题。

1. 不遵循代码标准
   sonar可以通过PMD,CheckStyle,Findbugs等等代码规则检测工具规范代码编写。
2. 潜在的缺陷
   sonar可以通过PMD,CheckStyle,Findbugs等等代码规则检测工具检 测出潜在的缺陷。
3. 糟糕的复杂度分布
   文件、类、方法等，如果复杂度过高将难以改变，这会使得开发人员 难以理解它们, 且如果没有自动化的单元测试，对于程序中的任何组件的改变都将可能导致需要全面的回归测试。
4. 重复
   显然程序中包含大量复制粘贴的代码是质量低下的，sonar可以展示 源码中重复严重的地方。
5. 注释不足或者过多
   没有注释将使代码可读性变差，特别是当不可避免地出现人员变动 时，程序的可读性将大幅下降 而过多的注释又会使得开发人员将精力过多地花费在阅读注释上，亦违背初衷。
6. 缺乏单元测试
   sonar可以很方便地统计并展示单元测试覆盖率。
7. 糟糕的设计
   通过sonar可以找出循环，展示包与包、类与类之间的相互依赖关系，可以检测自定义的架构规则 通过sonar可以管理第三方的jar包，可以利用LCOM4检测单个任务规则的应用情况， 检测耦合。


![image](./img/day05-13.png)

1. CS架构
   - sonarqube scanner
   - sonarqube server
2. SonarQube Scanner 扫描仪在本地执行代码扫描任务
3. 执行完后，将分析报告被发送到SonarQube服务器进行处理
4. SonarQube服务器处理和存储分析报告导致SonarQube数据库，并显示结果在UI中

### 2.4.1 sonarqube在kubernetes环境搭建
- 和gitlab共享postgres数据库
- 使用ingress地址 `sonar.devops.com` 进行访问
- 使用initContainers进行系统参数调整

**创建sonar数据库**
```
[root@k8s-master sonar]# kubectl -n jenkins get pods
NAME                              READY   STATUS    RESTARTS   AGE
gitlab-68d45b9dd5-pg4hr           1/1     Running   0          9d
jenkins-master-6b87f5bfc8-jhmqv   1/1     Running   0          5d
postgres-9b964c44-kfqr4           1/1     Running   0          9d
redis-87b46d999-w8hpl             1/1     Running   0          9d
[root@k8s-master sonar]# kubectl -n jenkins exec -it postgres-9b964c44-kfqr4 bash
root@postgres-9b964c44-kfqr4:~# psql
psql (11.4 (Debian 11.4-1.pgdg90+1))
Type "help" for help.

root=# create database sonar;
CREATE DATABASE
```

**准备好sonar.yaml文件**
```
[root@k8s-master sonar]# cat sonar.yaml
apiVersion: v1
kind: Service
metadata:
  name: sonarqube
  namespace: jenkins
  labels:
    app: sonarqube
spec:
  ports:
  - name: sonarqube
    port: 9000
    targetPort: 9000
    protocol: TCP
  selector:
    app: sonarqube
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: jenkins
  name: sonarqube
  labels:
    app: sonarqube
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sonarqube
  template:
    metadata:
      labels:
        app: sonarqube
    spec:
      initContainers:
      - command:
        - /sbin/sysctl
        - -w
        - vm.max_map_count=262144
        image: alpine:3.6
        imagePullPolicy: IfNotPresent
        name: elasticsearch-logging-init
        resources: {}
        securityContext:
          privileged: true
      containers:
      - name: sonarqube
        image: 172.17.176.31:5000/sonarqube:7.9-community
        ports:
        - containerPort: 9000
        env:
        - name: SONARQUBE_JDBC_USERNAME
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.user.root
        - name: SONARQUBE_JDBC_PASSWORD
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.pwd.root
        - name: SONARQUBE_JDBC_URL
          value: "jdbc:postgresql://postgres:5432/sonar"
        livenessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
          failureThreshold: 6
        resources:
          limits:
            cpu: 2000m
            memory: 4096Mi
          requests:
            cpu: 300m
            memory: 512Mi
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: sonarqube
  namespace: jenkins
spec:
  rules:
  - host: sonar.devops.com
    http:
      paths:
      - backend:
          serviceName: sonarqube
          servicePort: 9000
        path: /
status:
  loadBalancer: {}
[root@k8s-master sonar]# kubectl create -f sonar.yaml
service/sonarqube created
deployment.apps/sonarqube created
ingress.extensions/sonarqube created
```

**sonar-scanner安装**
下载地址： https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.2.0.1873-linux.zip

```
[root@k8s-master sonar]# ls -l
drwxr-xr-x 6 root root     4096 10月  1 2019 sonar-scanner-4.2.0.1873-linux
-rw------- 1 root root 42397119 11月 20 14:10 sonar-scanner-cli-4.2.0.1873-linux.zip
[root@k8s-master python-demo]# cat sonar-project.properties
sonar.projectKey=myblog
sonar.projectName=myblog
# if you want disabled the DTD verification for a proxy problem for example, true by default
sonar.coverage.dtdVerification=false
# JUnit like test report, default value is test.xml
sonar.sources=blog,myblog
```
配置sonarqube服务器地址

由于sonar-scanner需要将扫描结果上报给sonarqube服务器做质量分析，因此我们需要在sonar-scanner中配置sonarqube的服务器地址：

在集群宿主机中测试，先配置一下hosts文件，然后配置sonar的地址：
```
$ cat /etc/hosts
172.17.176.31  sonar.devops.com

$ cat sonar-scanner/conf/sonar-scanner.properties
#----- Default SonarQube server
#sonar.host.url=http://localhost:9000
sonar.host.url=http://sonar.devops.com
#----- Default source code encoding
#sonar.sourceEncoding=UTF-8
```
-   更改coredns的静态解析
-   在项目根目录下扫描
```
sonar-scanner -X
```
![image](./img/day05-14.png)

##  2.5 集成到tools镜像中
由于tools已经集成了java环境，因此可以直接剔除scanner自带的jre

- 删掉sonar-scanner/jre目录
- 修改sonar-scanner/bin/sonar-scanner

`use_embedded_jre=false`

重新构建tools镜像
```
[root@k8s-master tools]# ls -l
总用量 45608
-rw------- 1 root root     5453 11月 18 09:50 config
-rw-r--r-- 1 root root      950 11月 20 14:53 Dockerfile
-rwxr-xr-x 1 root root 46682408 11月 18 09:50 kubectl
-rwxr-xr-x 1 root root     1772 11月 20 14:52 sonar-scanner
[root@k8s-master tools]# cat Dockerfile
FROM alpine
USER root

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories && \
    apk update && \
    apk add  --no-cache openrc docker git curl tar gcc g++ make \
    bash shadow openjdk8 python2 python2-dev py-pip python3-dev openssl-dev libffi-dev \
    libstdc++ harfbuzz nss freetype ttf-freefont && \
    mkdir -p /root/.kube && \
    usermod -a -G docker root

COPY config /root/.kube/

RUN rm -rf /var/cache/apk/*
#-----------------安装 kubectl--------------------#
COPY kubectl /usr/local/bin/
RUN chmod +x /usr/local/bin/kubectl
# ------------------------------------------------#

#---------------安装 sonar-scanner-----------------#
COPY sonar-scanner /usr/lib/sonar-scanner
RUN ln -s /usr/lib/sonar-scanner/bin/sonar-scanner /usr/local/bin/sonar-scanner && chmod +x /usr/local/bin/sonar-scanner
ENV SONAR_RUNNER_HOME=/usr/lib/sonar-scanner
# ------------------------------------------------#
```

重新构建镜像，并推送到仓库：
```
[root@k8s-master tools]# docker build . -t 172.17.176.31:5000/devops/tools:v1
[root@k8s-master tools]# docker push 172.17.176.31:5000/devops/tools:v1
```
安装并配置sonar插件

由于sonarqube的扫描的结果需要进行Quality Gates的检测，那么我们在容器中执行完代码扫描任务后，如何知道本次扫描是否通过了Quality Gates，那么就需要借助于sonarqube实现的jenkins的插件。

- 安装插件

  插件中心搜索sonarqube，直接安装

- 配置插件

  系统管理->系统配置-> **SonarQube servers** ->Add SonarQube

  - Name：sonarqube

  - Server URL：http://sonar.luffy.com

  - Server authentication token

    ① 登录sonarqube -> My Account -> Security -> Generate Token

    ② 登录Jenkins，添加全局凭据，类型为Secret text

- 如何在jenkinsfile中使用

  我们在 https://jenkins.io/doc/pipeline/steps/sonar/ 官方介绍中可以看到：
  
**修改Jenkinsfile**
```
[root@k8s-master python-demo]# cat Jenkinsfile
pipeline {
    agent { label 'jnlp-slave'}

    options {
		buildDiscarder(logRotator(numToKeepStr: '10'))
		disableConcurrentBuilds()
		timeout(time: 20, unit: 'MINUTES')
		gitLabConnection('gitlab')
	}

    environment {
        IMAGE_REPO = "172.17.176.31:5000/demo/myblog"
        DINGTALK_CREDS = credentials('dingding-ops-token')
        TAB_STR = "\n                    \n&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"
    }

    stages {
        stage('checkout') {
            steps {
                container('tools') {
                    checkout scm
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS = env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('printenv') {
            steps {
                script{
                    sh "git log --oneline -n 1 > gitlog.file"
                    env.GIT_LOG = readFile("gitlog.file").trim()
                }
                sh 'printenv'
            }
        }
        stage('CI'){
            failFast true
            parallel {
                stage('Unit Test') {
                    steps {
                        echo "Unit Test Stage Skip..."
                    }
                }
                stage('Code Scan') {
                    steps {
                        container('tools') {
                            withSonarQubeEnv('sonarqube') {
                                sh 'sonar-scanner -X'
                                sleep 3
                            }
                            script {
                                timeout(1) {
                                    def qg = waitForQualityGate('sonarqube')
                                    if (qg.status != 'OK') {
                                        error "未通过Sonarqube的代码质量阈检查，请及时修改！failure: ${qg.status}"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('build-image') {
            steps {
                container('tools') {
                     retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('push-image') {
            steps {
                 container('tools') {
                    retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('deploy') {
            steps {
                container('tools') {
                    sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' deploy/*"
                    timeout(time: 1, unit: 'MINUTES') {
                        sh "kubectl apply -f deploy/"
                    }
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
    }
    post {
        success {
            echo 'Congratulations!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😄👍 构建成功 👍😄  \n**项目名称**：luffdevops**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}   \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """
        }
        failure {
            echo 'Oh no!'
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😖❌ 构建失败 ❌😖  \n**项目名称**：luffdevops**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}  \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
            """
        }
        always {
             cleanWs()
        }
    }
}
```