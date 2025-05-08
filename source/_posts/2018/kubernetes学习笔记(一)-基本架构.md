---
title: kubernetes学习笔记(一)-基础入门
date: 2018-09-11 09:24:39
tags:
 - Docker
 - kubernetes
---

## 1. 基本术语

Service: 唯一名字，拥有唯一虚拟IP和端口号

Pod: 通过label与service绑定，业务容器、基础设施容器

Node：物理机或者虚拟机

Master Node上运行: kube-apiserver、kube-controller-manager和kube-scheduler

Work Node上运行: kubelet、kube-proxy

## 2. 细节

### 网络部分

 - 每个Pod对应一个docker网桥的IP
 - 每个service通过endpoint(dockerIp:port)与select的Pod进行绑定
 - 每个service会分配一个Cluster IP(集群内使用的IP)
 - service可以通过NodePort或者LoadBalancer对外提供服务

``` yaml
# 外部通过30001访问Pod的80端口
type: NodePort
ports:
    - port: 80
      nodePort: 30001
```

### 存储部分

``` yaml
# 将主机上/data定义卷
volumns:
    - name: "persistent-storage"
      hostPath:
        path: "/data"
# 挂载到容器内部的/data
containers:
    volumeMounts:
        - name: "persistent-storage"
          mountPath: "/data"
```

## 3. 架构与组件

![基本架构](https://upload.wikimedia.org/wikipedia/commons/thumb/b/be/Kubernetes.png/600px-Kubernetes.png)

API Server: 提供资源对象的唯一操作入口

Controller Manager: 管理控制中心

Scheduler: 调度器

Kubelet: 负责本节点上的Pod的创建、修改等管理，向API Server上报状态信息

Proxy: Service的代理及负载均衡

## 4. 常用命令
``` sh
kubectl get <resource_type> <resource_name>
kubectl describe <resource_type> <resource _name>
kubectl scale rc <rc_name> --replicas=<num>
kubectl create/replace -f <file>
kubectl label pod <pod_name> <label_name=label_value>
kubectl rolling-update <rc> --image=<new_image>
```
