# kubernetes-tutorial
Kubernetes 入门教程
---
# 简介

k8s是谷歌咋2014年发布的一个开源项目。是谷歌开发的borg系统的开源版本。
谷歌十年前就开始适应容器技术，数据中心里运行着20多亿个容器。

官网最小测试环境：https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/cluster-interactive/ 
  minikube version
  minikube start
  kubectl get po
  kubectl cluster-info

--------

# 重要概念

Cluster：是计算、存储和网络资源的集合，k8s利用这些资源运行各种基于容器的应用。
Master：是cluster的大脑，主要责任是**调度**，即决定将应用放在那里运行。可以运行多个mster来实现高可用。
Node：
