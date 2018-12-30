# kubernetes-tutorial

## 简介

k8s是谷歌咋2014年发布的一个开源项目。是谷歌开发的borg系统的开源版本。  
谷歌十年前就开始适应容器技术，数据中心里运行着20多亿个容器。

官网最小测试环境：https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/cluster-interactive/  
- minikube version  
- minikube start  
- kubectl get po  
kubectl cluster-info  

## 重要概念

Cluster：是计算、存储和网络资源的集合，k8s利用这些资源运行各种基于容器的应用。  
Master：是cluster的大脑，主要责任是**调度**，即决定将应用放在那里运行。可以运行多个mster来实现高可用。  
Node：运行容器应用，由master管理，node负责监控并汇报容器的状态，同时管理容器的生命周期。  
Pod：是k8s最小工作单元，每个pod包含一个或多个容器，pod作为一个整体被master调度到某个node上运行。  
- k8s管理的是pod，而不是容器。
- pod中的容器 有相同的ip地址+端口号，它们之间使用localhost之间通信  
- pod中的容器 共享存储，k8s挂载volume到pod，本质上是将volume挂载到pod中的每一个容器。

问：哪些容器应该放到一个pod中？  
答：联系非常紧密，而且需要共享资源；比如两个容器，一个是写
