---
layout: post
title: Calico配置网络策略管控Pod间流量
date: 2020-08-26
tags: Kubernetes
---

Kubernetes 集群中通过创建 NetworkPolicy 的方式来声明网络策略，以管理 Pod 之间的网络通信流量。

# 前提条件
- 一个已经存在的Kubernetes集群
- 网络插件需要支持Network Policy
    - Calico 
    - Cilium
    - Kube-router

# 创建一个deployment并配置Service

- 创建Deployment-myapp

```
 kubectl  create deployment myapp --image=ikubernetes/myapp:v1 --replicas=1
```

- 创建一个Serivce暴露Deployment

```
 kubectl expose deployment myapp --port=80 --target-port=80
```

- 查询结果

```
 kubectl  get svc,pod -o wide
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/kubernetes   ClusterIP   10.68.0.1    <none>        443/TCP   22h   <none>
service/myapp        ClusterIP   10.68.9.59   <none>        80/TCP    45s   app=myapp

NAME                        READY   STATUS    RESTARTS   AGE     IP              NODE            NOMINATED NODE   READINESS GATES
pod/myapp-9cbc4cf76-pvjfc   1/1     Running   0          2m26s   172.20.43.131   192.168.1.212   <none>           <none>
```

# 创建一个Pod访问myapp

- 创建pod

```
 kubectl  run busybox --image=busybox:1.28 -- sleep 3600
```

- 访问myapp，默认情况下，如果没有设置网络策略的情况下，各个pod间可以直接访问

```
 kubectl  exec -it busybox  -- wget -O - -q --timeout=1 myapp
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
```

# 添加网络策略，限制对myapp的访问

- 设置网络策略，声明只有带有access=true标签的pod才能访问myapp服务

```
vim network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-myapp
spec:
  podSelector:
    matchLabels:
      app: myapp
  ingress:
  - from:
      - podSelector:
          matchLabels:
            access: "true"
```

- 再次访问，发现busybox无法访问myapp服务

```
 kubectl  get pod --show-labels
NAME                    READY   STATUS    RESTARTS   AGE   LABELS
busybox                 1/1     Running   0          11m   run=busybox
myapp-9cbc4cf76-pvjfc   1/1     Running   0          16m   app=myapp,pod-template-hash=9cbc4cf76

 kubectl  exec -it busybox  -- wget -O - -q --timeout=1 myapp 
wget: download timed out
command terminated with exit code 1
```

- 给busybox添加access=true标签后，访问正常

```
 kubectl  get pod --show-labels
NAME                    READY   STATUS    RESTARTS   AGE   LABELS
busybox                 1/1     Running   0          12m   access=true,run=busybox
myapp-9cbc4cf76-pvjfc   1/1     Running   0          17m   app=myapp,pod-template-hash=9cbc4cf76

kubectl  exec -it busybox  -- wget -O - -q --timeout=1 myapp 
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
```