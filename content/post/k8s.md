---
title: "K8s学习笔记"
date: 2021-11-11T13:54:38+08:00
draft: true
---

容器的本质是进程
namespace可以让容器内部看不到外部的进程，PID从1开始。从而实现环境隔离。
cgroup机制用来限制容器可以消耗的资源，比如内存，cpu，磁盘io等。
rootfs用来作为容器的文件系统，容器内部看不到外部的文件，宿主机也看不到容器内。通过结合使用 Mount Namespace 和 rootfs，容器就能够为进程构建出一个完善的文件系统隔离环境


由于 rootfs 里打包的不只是应用，而是整个操作系统的文件和目录，也就意味着，应用以及它运行所需要的所有依赖，都被封装在了一起。
对一个应用来说，操作系统本身才是它运行所需要的最完整的“依赖库”。

镜像理解其实就是rootfs，就是改操作系统的所有文件和目录。


Docker 镜像使用的 rootfs，往往由多个“层”组成：

只读层就是镜像文件
init层是一些系统配置文件，host啥的
可读写层就是容器层

通过AuFS机制，文件从上往下依次被覆盖。所以容器删除之后可读写层的东西就都没有了，想要保留必须挂载出来

镜像的可以理解为是exe程序，容器就是进程。



kubectl apply -f nginx.yaml    采用配置，创建或者修改资源

kubectl get pods  获取pods 列表
kubectl get deployments  获取deployment 列表
kubectl exec -it test-projected-volume -- /bin/sh   进入某个pod中
kubectl describe pod  查看pod 详情
kubectl rollout status deployment/nginx-deployment 实时查看 Deployment 对象的状态变化

pod相当于虚拟机，容器相当于进程，k8s就是操作系统。

pod可以让一组容器共享一些资源，比如ip地址，挂载同一份文件等。
pod就是一组强关联的容器的组合。


deployment 用来调度pod,比如pod挂掉的时候自动重启，保证几个pod在正常运行。




