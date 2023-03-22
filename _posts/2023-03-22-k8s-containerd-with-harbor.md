---
layout: post
title: K8S搭配COntainerd拉取Habor镜像进行部署
date: 2023-03-22
tags: Kubernetes
---

## 一、实验环境

本次实验通过配置 containerd 支持 Harbor 实现 Kubernetes集成Habor镜像仓库，用到的相关程序版本如下:
- harbor：v2.5.5 
- kubernetes：v1.25.6
- cri: containerd

## 二、配置 Containerd 支持 Harbor 仓库

#### 2.1. 复制 Harbor证书文件到本机

```
mkdir -pv /etc/containerd/certs.d
scp reg.linux.io:/opt/harbor/ssl/{ca.crt,reg.linux.io.key,reg.linux.io.crt} /etc/containerd/certs.d
```

#### 2.2 添加私有CA证书为可信证书

```
cp /etc/containerd/certs.d/ca.crt /usr/local/share/ca-certificates
update-ca-certificates 
Updating certificates in /etc/ssl/certs...
rehash: warning: skipping ca-certificates.crt,it does not contain exactly one certificate or CRL
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
```

#### 2.3 修改配置文件
```
cp /etc/containerd/config.toml{,.bak}
vim /etc/containerd/config.toml
  ......
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        # 配置证书
        [plugins."io.containerd.grpc.v1.cri".registry.configs."reg.linux.io".tls]
          ca_file = "/etc/containerd/certs.d/ca.crt"
          cert_file = "/etc/containerd/certs.d/reg.linux.io.crt"
          key_file  = "/etc/containerd/certs.d/reg.linux.io.key"
        # 配置用户名密码,经过测试这个不配置也能成功
        [plugins."io.containerd.grpc.v1.cri".registry.configs."reg.linux.io".auth]
          username = "admin"
          password = "Harbor12345" 

      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        # 配置 Harbor地址
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."reg.linux.io"]
          endpoint = ["https://reg.linux.io"]
  ......
```
#### 2.4 重启containerd服务

```
systemctl daemon-reload && systemctl restart containerd.service
```

## 2.5 命令行登录Harbor

```
nerdctl  login reg.linux.io -u admin -p Harbor12345 
WARN[0000] WARNING! Using --password via the CLI is insecure. Use --password-stdin. 
WARNING: Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

## 2.6 推送镜像到 Harbor 仓库

```
nerdctl pull ikubernetes/demoapp:v1.0 --all-platforms 
nerdctl tag ikubernetes/demoapp:v1.0 reg.linux.io/dev/demoapp:v1.0
nerdctl  push reg.linux.io/dev/demoapp:v1.0
```
如果推送没问题，说明配置没问题了。

## 三、配置Kubernetes拉取镜像仓库

#### 3.1 生成 harbor-secret

```
cat ~/.docker/config.json | base64 -w 0
ewoJImF1dGhzIjogewoJCSJyZWcubGludXguaW8iOiB7CgkJCSJhdXRoIjogIllXUnRhVzQ2U0dGeVltOXlNVEl6TkRVPSIKCQl9Cgl9Cn0=

vim harbor-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: harbor-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: ewoJImF1dGhzIjogewoJCSJyZWcubGludXguaW8iOiB7CgkJCSJhdXRoIjogIllXUnRhVzQ2U0dGeVltOXlNVEl6TkRVPSIKCQl9Cgl9Cn0=

kubectl apply -f harbor-secret.yaml 
```

#### 3.2 配置Deployment使用secret 拉取镜像

```
cat > demoapp-deploy.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
   name: harbor-demoapp
   namespace: default
   labels:
     app: demoapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demoapp
  minReadySeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
     metadata:
        labels:
          app: demoapp
     spec:
       imagePullSecrets:
       - name: harbor-secret
       containers:
       - name: demoapp
         image: reg.linux.io/dev/demoapp:v1.0
         ports:
         - name: http
           protocol: TCP
           containerPort: 80

         resources:
           requests: 
               memory: 50Mi
               cpu: 50m
EOF
```

```
kubectl  get pod -l app=demoapp
NAME                              READY   STATUS    RESTARTS   AGE
harbor-demoapp-798d8cfd5b-jllgd   1/1     Running   0          43s
harbor-demoapp-798d8cfd5b-nnqgz   1/1     Running   0          43s
harbor-demoapp-798d8cfd5b-vkjt9   1/1     Running   0          43s
```
