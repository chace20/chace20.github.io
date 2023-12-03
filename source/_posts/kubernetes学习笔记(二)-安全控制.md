---
title: kubernetes学习笔记(二)-安全控制
date: 2018-09-11 09:30:58
tags:
 - Docker
 - kubernetes
---

## 1. 端口问题
``` yaml
pod:
 - containerPort: 容器端口
   hostPort: 主机端口

service:
 - port: 监听端口
   targetPort: 转发到pod的端口
   nodePort:>30001 外部访问的端口
```


## 2. 容器健康检查探针
``` yaml
livenessProbe:
    httpGet:
        path: /
        port: 9090
    initialDelaySeconds: 30
    timeoutSeconds: 30
```

## 3. 安全控制
### Authentication认证
 - CA
 - token
 - http base64
 
### Authorization授权
通过指定`--authorization_policay_file=SOME_FILENAME`来确定
 
### Admission Control准入控制
调用这些插件拦截通过上面认证和授权的请求。
 
``` ini
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
```
 
### Secret
私密信息保存到secret中，可以挂载到pod中。

``` yaml
apiVersion: v1
kind: Secret
metadata: 
    name: mysecret
type:
    Opaque
data:
    password: ODgyMAo=
    username: Y2hhY2UK
```

包含三种类型：
 - Opaque
 - kubernetes.io/dockercfg
 - kubernetes.io/service-account-token
 
Secret有三种使用方式：
 - 创建pod时，指定Service Account自动使用该secret
  - 挂载secret到pod使用
  - 创建pod时，指定pod的spc.ImagePullSecrets来引用它

### Service Account
Service Account是多个secret的集合。
 - 普通secret，用于访问API Secret
 - imagePullSecret，用于下载镜像

在`--admission-control=ServiceAccount`启用。

除了namespace，也可以通过切换上下文的方式来隔离pod。
