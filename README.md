# kubernetes-tutorial

## 简介

k8s是谷歌咋2014年发布的一个开源项目。是谷歌开发的borg系统的开源版本。  
谷歌十年前就开始适应容器技术，数据中心里运行着20多亿个容器。

官网最小测试环境：https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/cluster-interactive/  
- minikube version  
- minikube start  
- kubectl get po  
- kubectl cluster-info  

## 重要概念

Cluster：是计算、存储和网络资源的集合，k8s利用这些资源运行各种基于容器的应用。  
Master：是cluster的大脑，主要责任是**调度**，即决定将应用放在那里运行。可以运行多个mster来实现高可用。  
Node：运行容器应用，由master管理，node负责监控并汇报容器的状态，同时管理容器的生命周期。  
Pod：是k8s最小工作单元，每个pod包含一个或多个容器，pod作为一个整体被master调度到某个node上运行。  
- k8s管理的是pod，而不是容器。
- pod中的容器 有相同的ip地址+端口号，它们之间使用localhost之间通信  
- pod中的容器 共享存储，k8s挂载volume到pod，本质上是将volume挂载到pod中的每一个容器。
问：哪些容器应该放到一个pod中？  
答：联系非常紧密，而且需要共享资源；比如两个容器，一个是写文件到共享存储，另外一个是从存储读文件后展示。  
Controller：管理pod的生命周期，定义pod部署特性，如几个副本，在哪个node上运行等；类型很多以满足不同应用场景，如：  
- Deployment：最常用，可以管理pod的多个副本  
- ReplicaSet：实现pod的多副本管理，不直接使用，而是通过被Deployment调用  
- DaemonSet：每个node上最多运行一个pod副本场景  
- StateflueSet：保证pod的副本在整个生命周期中名称不变，其他controller不提供改功能，并保证副本按固定顺序启动、更新和删除。  
-Job：运行结束就删除的pod，其他controller中的pod通常是长期运行的。   
Service：定义了外界访问pod的方式。有自己的ip和端口，并提供负载均衡。
Namespace：将一个物理上cluster逻辑上划分成多个虚拟cluster，每个cluster就是一个namespace，资源完全隔离；默认创建两个：defalut、kube-system。  
## k8s架构

k8s = master + node + etcd（可共用master/node机器）  
- master = API Server + Controller Manager + Scheduler  
- node = kubelet + kube-proxy + pod网络  

API Server：kube-apiserver，提供http/https restful api，作为前端接口，供其他调用管理k8s资源。  
Scheduer：kube-scheduler，决定将pod放在哪个node上运行，会考虑node负载以及高可用要求等。  
Controller Manager：kube-controller-manager，管理资源，保证资源处于预期状态。  
etcd：保存集群的配置信息和资源状态信息；数据变化后会通知集群相关组件。  
kubelet：根据scheduler提供的配置信息，创建并允许容器，并想master汇报运行状态。
kube-proxy：转发service的访问pod请求，将tcp/udp数据流转发到后端容器；若多个副本，则实现负载均衡。  
pod网络：保证pod直接的网络通信，如flannel。

## 运行应用

pod创建过程：kubectl创建deployment，deployment创建relplicaSet，relplicaSet创建pod  
子对象的名字 = 父对象名字 + 随机字符串/数字

pod创建方式：命令行直接创建，二是yaml配置文件 + kubeclt apply  

运行：kubectl run nginx-deployment --image=nginx:1.7.9 --replicas=2  
查看：kubectl get deployment xxx  
详情：kubectl descripe deployment xxx  

kubectl apply -f nginx.yml  
#### nginx.yml  
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





kubectl：k8s命令行工具，  
kubectl get pod --all-namespaces  
kubectl get pod --all-namespaces -o wide  
kubectl descrbe pod xxx  


