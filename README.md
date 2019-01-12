# kubernetes-tutorial

## 简介

kubernetes是谷歌在2014年发布的一个开源项目，是其内部使用的borg系统的开源版本。  
传说谷歌十年前就开始使用容器技术，数据中心里运行着20多亿个容器。  
kubernetes（中间有8个字母，故称k8s，且读音相近）源自希腊语，译作 航海大师。  

官网最小测试环境：https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/cluster-interactive/  
- minikube version  
- minikube start  
- kubectl get po  
- kubectl cluster-info  

## 重要概念

Cluster：是计算、存储和网络资源的集合，k8s利用这些资源运行各种基于容器的应用。  
Master：是cluster的大脑，主要责任是调度，即决定将应用放在那里运行。可以运行多个master来实现高可用。  
Node：运行容器应用，由master管理，node负责监控并汇报容器的状态，同时管理容器的生命周期。  
Pod：是k8s最小工作单元，每个pod包含一个或多个容器，pod作为一个整体被master调度到某个node上运行。  
- k8s管理的是pod，而不是容器。
- pod中的容器 有相同的ip地址+port空间，它们之间使用localhost之间通信  
- pod中的容器 共享存储，k8s挂载volume到pod，本质上是将volume挂载到pod中的每一个容器

问：哪些容器应该放到一个pod中 ？  
答：联系非常紧密，而且需要共享资源；比如两个容器，一个是写文件到共享存储，另外一个是从存储读文件后展示。  

Controller：管理pod生命周期，定义pod部署特性，如几个副本，在那个node上运行等；类型很多以满足不同应用场景，如：  

- Deployment：最常用，可以管理pod的多个副本  
- ReplicaSet：实现pod的多副本管理，不直接使用，而是通过被Deployment调用  
- DaemonSet：每个node上最多运行一个pod副本场景  
- StateflueSet：保证pod的副本在整个生命周期中名称不变，其他controller不提供该功能，并保证副本按固定顺序启动、更新和删除  
- Job：运行结束就删除的pod，其他controller中的pod通常是长期运行的   

Service：定义了外界访问pod的方式。有自己的ip和端口，并提供负载均衡。  
Namespace：将一个物理上cluster逻辑上划分成多个虚拟cluster，每个cluster就是一个namespace，资源完全隔离。默认创建两个：defalut、kube-system。  
## k8s架构

k8s = master + node + etcd（可共用master/node机器）  
- master = API Server + Controller Manager + Scheduler  
- node = kubelet + kube-proxy + pod网络  

API Server：kube-apiserver，提供http/https restful api，作为前端接口，供其他调用管理k8s资源。  
Scheduer：kube-scheduler，决定将pod放在哪个node上运行，会考虑node负载以及高可用要求等。  
Controller Manager：kube-controller-manager，管理资源，保证资源处于预期状态。  
etcd：保存集群的配置信息和资源状态信息；数据变化后会通知集群相关组件。  
kubelet：根据scheduler提供的配置信息，创建并运行容器，并向master汇报运行状态。  
kube-proxy：转发service的pod访问请求，将tcp/udp数据流转发到后端容器；若多个副本，则实现负载均衡。  
pod网络：保证pod直接的网络通信，如flannel。

## 运行应用

### 创建
pod创建过程：kubectl创建deployment，deployment创建relplicaSet，relplicaSet创建pod  
子对象的名字 = 父对象名字 + 随机字符串/数字

pod创建方式：一是命令行直接创建，二是yaml配置文件 + kubeclt apply  

运行：kubectl run nginx-deployment --image=nginx:1.7.9 --replicas=2  
查看：kubectl get deployment xxx  
详情：kubectl descripe deployment xxx  

kubectl apply -f nginx.yml  
nginx.yml 内容：  
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: web_server
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.7
```
- apiVersion：当前配置文件的版本  
- kind：要创建的资源类型  
- metadata：该资源的元数据，name是必选项  
- spec：该资源的规格说明  
- replicas：副本数  
- template：定义pod的模板  
- metadata：定义pod的元数据，至少要定义一个label  
- spec：pod的规格说明，定义pod中每一个容器的属性，name和image是必选项  

### 伸缩
在线减少或增加pod的副本数  
方法：修改yaml文件中的replicas参数值后，重新执行 kubeclt apply 即可  

处于安全考虑，默认配置下，k8s不会将pod调度到mster节点  
设置方法：kubectl taint node k8s-master node-role.kubernetes.io/master-  
恢复方法：kubectl taint node k8s-master node-role.kubernetes.io/master="":NoSchedule  

### Failover
关闭集群中某一个node后，k8s检测到该node不可用，会把其上的pod标记为unknown状态，并在其他可用node上恢复这些pod；当该node恢复后，那些标记为unknown的pod会被删除，但是已经在其他node上运行的那些pod不会重新调度回来。  

### label控制pod位置
默认情况下，pod会被scheduler调度到所有可用的node上，特殊场景下需要指定pod运行的node。如把磁盘I/O高的pod部署到配置了ssd的node上，有些pod需要使用gpu等。k8s使用label来实现这个功能。  

方法：先给特殊资源添加bael，然后再创建pod时指定一些label来使用特殊资源。  
创建：kubectl label node node1 disktpye=ssd  
使用：在yaml文件中
```
spce:
  containers:
  - name: nginx
    image:ngingx:1.7.9
  nodeSelector:
    disktpye: ssd
```    
删除：kubectl label node node1 disktpye-  

### DaemonSet
Deployment部署的pod副本分布在多个node上，有可能分布一个node上，也可能均匀分布，也可能有的node上没有（node数大于pod副本数）。  
DaemonSet语法几乎与Deployment一模一样，只是将kind类型改为DaemonSet。  

有些场景需要每个node上都要分布该pod：  

- 集群中每个节点上都运行的存储pod 
- 集群中每个节点上都运行的日志pod
- 集群中每个节点上都运行的监控pod

### Job

容器按运行时间长短可分为：服务类容器（如http sever） 和 工作类容器（如批处理）  

查看：kubeclt get job (列desied和successful都为1，才表示启动成功一个pod)  
因为pod执行完毕后自动退出，可通过 kbuectl logs xxx 查看pod的结果输出。  

#### 并行job  
通过parallelism（每次运行多少个pod） + completions（总共运行多少个pod） 设置同时运行多个pod，来提高job的执行效率。  
通过查看AGE列值是否相同，判断是否是同时并行执行的。  

#### 定时jod
CronJob：定时job，类似linux中的cron定时任务。  

## Service

controller通过动态的创建和销毁pod来保证应用整体的健壮性，即pod是脆弱的，应用是健壮的。  

每个pod都有自己的ip，pod在发生故障后，新的pod会重新分配ip，这里产生一个问题，pod的ip在变化，用户怎么访问呢 ？  
答：service  
service有自己的cluster ip，且固定不变，k8s维护service和pod的映射关系，无论后端pod的ip如何变化，通过service访问pod，用户对后端变化无感知。  

pod的ip是在容器中配置的，那个service的cluser ip是在哪里设置的呢？这个cluster ip又是如果映射到pod的ip的呢 ？  
答：iptables  
cluster ip是一个虚拟ip，由k8s上的iptables规则来管理的。  
iptables将访问service的流量转发到后端的pod上，使用类似轮询的负载均衡策略。  

### service类型
- Cluster IP：默认类型，供集群内的节点和pod访问  
- NodePort：通过集群node的静态端口对外提供服务，外部通过NodeIP:NodePort访问service
- LoadBalancer：采用cloud provider特有的load balancer对外提供服务

## 网络

k8s采用基于扁平地址空间的网络模型，每个pod都有自己的ip，pod之间不需要配置nat就能直接通信，pod内的容器共享pod的ip，能够通过localhost通信。

pod内：不同的pod之间不存在端口冲突的情况，因为每个pod的ip不同，当容器使用localhost时，意味着使用的是其本身所在pod的地址空间。  
pod间：pod的ip是集群间可见的，也就是集群中的任何pod和节点都可以通话ip和pod通信。
pod于service：pod间可以直接通过ip通信，但是要提前知道pod的ip，而pod被重新创建时ip会发生变化，service提供了pod的抽象层，将请求转发给正确的pod，还实现了高可用和负载均衡。  
外部访问：无论是pod的ip还是service的cluster ip，都只能在k8s集群内可见，对外集群外来说，这些ip都是私有的。k8s提供了两种方式让外界于pod通信。

- NodePort：Service通过node的静态端口对外提供服务，外部可以通过 NodeIp：NodePort访问Service
- Loadbalancer:Service利用cloud provider提供的load balancer对外提供服务，如：AWS、Azure、GCP等。

## Volume

容器和pod是短暂的，它们的生命周期可能很短，会被频繁的销毁和创建，容器销毁时，保存在容器中的数据都会被清除。  
k8s使用volume持久化容器的数据，容器可能被销毁、但是volume数据库会被保留。  
本质上，volume是一个目录，会被mount到pod上，pod中所有的容器都可以访问这个目录。  

- emptyDir：是一个临时目录，生命周期于pod一致，不与pod中的容器一致。方便地为pod提供共享存储，适用于容器需要临时共享存储空间的场景。
- hostPath：是将文件系统上已经存在目录mount到pod，大部分应用不会使用，因为增加了pod于node的耦合，限制了pod的作用，不过安心需要访问k8s和docker内容数据的应用需要使用该方式。pod被销毁，对应的volume还存在。
- 外部storage：如aws、azure、gce等公有云上，也可以使用主流的分布式存储，如Ceph、GlusterFS等。最大特点是，volume不依赖k8s，由独立的存储系统管理，即使k8s崩溃，数据也不会受损。当然增加了运维的复杂性，对可靠性、可用性和可扩展要求高的场景。

## Secret & Configmap

应用启动过程中需要的一些敏感信息，如用户名、密码和密钥，将这些信息保留在容器镜像中显然不妥，k8s使用secret。  
Secret以密文的方式存储数据，避免了在配置文件中保存敏感信息，secret以volume的形式被mount到pod，容器可以以文件或者环境变量方式使用这些数据。

对于一些非敏感数据，比如应用的配置信息，可以使用ConfigMap。

## Helm 包管理器

每个成功的软件平台都有一个优秀的打包工具，debian、Ubuntu的apt，redhat、centos的yum，Helm是k8s的一个高层次的应用打包工具。


## Rolling Update

滚动更新：就是一次只更新一小部分副本，成功后再更新其他副本，最终更新完所有副本。 
最大好处就是零停机，整个更新过程中，始终有可用副本在运行，从而保证了业务的连续性。  

方法：修改yaml文件中image对应的版本号后执行，每次替换的pod数量可以定制，通过参数：masSurge、maxUnavailable。  

更新回滚：每次更新，k8s都会记录下当前的配置并保存为一个revision，以便失败回滚，版本的数量通过revisionHistoryLimit设置。
方法：kubectl rollout undo deployment xxx --to-revision=2  

## Health check

强大的自愈能力是k8s容器编排引擎的一个重要特性，默认是通过自动重启发生故障的日期来实现。

默认检查机制：每个容器启动时都会执行一个进程，此进程有dockerfile的cmd或者enttypoint指定。进程退出时返回非0，则认为发生故障，k8s根据restartPolicy（Nerver、OnFailure、Always）重启容器。  
查看：kubectl get pod healthcheck  可以看到重启的次数  

问：有些场景下，出现了故障，但是进程并不会退出的情况，该怎样检查 ？  
答：Liveness探测  

Liveness探测可以让用户自定义判断容器是否健康的条件，若探测失败，k8s就会重启容器。  
Readiness探测告诉k8s什么时候可以让容器加入到Service负载均衡池中。  

- 两种检测机制，若不特地配置，都采取默认行为，即通过判断容器进程返回值是否为零
- 两种检测机制，配置参数一样，不同之处在于探测失败后的行为，Liveness会重启容器，Readiness将容器设置为不可用，不接收service转发的请求  
- 两种检测机制，相互独立，没有依赖，可单独使用，也可同时使用 
- Liveness探测容器是否需要重启以实现自愈，Readiness探测容器是否准备好对外提供服务。  

使用场景1：scale up  
应用启动容器都需要一段时间，比如加载缓存时间、连接数据库等，从容器启动到真正能对外提供服务需要一段时间的，可以通过Redadiness探测容器是否就绪，避免发送到还没有准备好的backend。  
使用场景2：rolling update    
新副本需要时间完成准备工作，假如由于人为配置错误，副本始终无法完成准备工作（如无法连接数据库），由于新副本没有异常退出，默认的检测机制人为容器已经就绪，琢步将新副本替换旧副本，结果就是，所有都替换后，应用无法对外提供服务。如果使用Redadiness探测，新副本只有通过了探测后才会被加入到service，旧副本才会被替换。

## Dashboard

k8s提供了一个基于web的dashboard，可以部署容器化的应用，监控应用状态，排查故障，管理各种资源。

## 监控

当集群运行起来后，需要保证集群都是正常的，必要组件都各司其职，有足够的资源满足应用的需求。  

Scope：是docker和k8s的可视化监控工具。每个node上都会运行agent收集数据，scope app从agent获取数据并展示。  
Heapster：是k8s的原生监控方案，以pod形式运行，自动发现集群节点，从节点上的kubelet获取监控数据，kubelet从节点上的cAdvisor收集数据。收集到的监控数据保存在InfluxDB时序数据库中，并通过Grafana展示。  

Scope和Heapster的监控对象是node和pod，但是这些还不够，比如想监控api server、scheduler、controller等组件状态。  
Prometheus Operator：CoreOS基于prometheus开发的k8s监控方案，是目前功能最全的开源监控方案。

- exporter：收集目标对象数据，通过http接口暴露出来
- prometheus server：从exporter上拉取并储存监控数据
- grafana：prometheus自己开发了一套可视化组件但是废弃了，因为社区有更出色的grafana，可以无缝对接
- altermanager：告警组件

## 日志

EFK  
Elasticsearch是一个搜索引擎，存储日志并提供查询接口；Fluentd从k8s收集日志并发送给Elasticsearch存储；Kibana提供一个web gui，供用户浏览和搜索存储在Elasticsearch中日志。

## 打赏
如果您觉得这篇文档对您有所帮助，请微信扫描下方的二维码进行打赏  
加油后将会为您分享更多的作品，谢谢！
 ![image](https://github.com/yanchaozhong/kubernetes-tutorial/blob/master/images/payforme.png)
 
