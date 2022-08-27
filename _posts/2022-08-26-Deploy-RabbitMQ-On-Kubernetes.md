---
layout: post
title: 使用Operator创建RabbitMQ集群
date: 2020-08-26
tags: Kubernetes
---

# 安装要求
- Kubernetes版本： 1.18+

```
kubectl  get nodes
NAME            STATUS   ROLES    AGE   VERSION
192.168.1.210   Ready    master   23h   v1.23.1
192.168.1.211   Ready    node     22h   v1.23.1
192.168.1.212   Ready    node     22h   v1.23.1
192.168.1.213   Ready    node     22h   v1.23.1
```

- 需要一个默认的StorageClass

```
kubectl  get sc
NAME                PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-csi (default)   nfs.csi.k8s.io   Delete          Immediate           false                  3m18s
```

# 安装步骤

1. 安装cluster-operator

```
 wget https://github.com/rabbitmq/cluster-operator/releases/download/v1.7.0/cluster-operator.yml
 kubectl  apply -f cluster-operator.yml 

 kubectl get all -n rabbitmq-system 
NAME                                             READY   STATUS    RESTARTS   AGE
pod/rabbitmq-cluster-operator-658b68747c-cpvpg   1/1     Running   0          3m48s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/rabbitmq-cluster-operator   1/1     1            1           3m48s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/rabbitmq-cluster-operator-658b68747c   1         1         1       3m48s
```

2. 使用cluster-operator创建RabbitMQ集群

```
wget  https://raw.githubusercontent.com/rabbitmq/cluster-operator/main/docs/examples/hello-world/rabbitmq.yaml 

cp rabbitmq.yaml{,.bak} && vim rabbitmq.yaml
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: dev-rabbitmq-cluster
  namespace: rabbitmq-system

kubectl  apply -f rabbitmq.yaml

kubectl get pod -n rabbitmq-system 
NAME                                         READY   STATUS    RESTARTS   AGE
dev-rabbitmq-cluster-server-0                1/1     Running   0          10m
rabbitmq-cluster-operator-658b68747c-cpvpg   1/1     Running   0          33m
```

3. 获取用户名和密码

```
username="$(kubectl get secret dev-rabbitmq-cluster-default-user -n rabbitmq-system -o jsonpath='{.data.username}' | base64 --decode)"
echo "username: $username"  

password="$(kubectl get secret dev-rabbitmq-cluster-default-user -n rabbitmq-system -o jsonpath='{.data.password}' | base64 --decode)"
echo "password: $password"
```
```
# 默认会创建一个类型是ClusterIP的svc
kubectl get svc -n rabbitmq-system
NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
dev-rabbitmq-cluster         ClusterIP   10.68.174.204   <none>        5672/TCP,15672/TCP,15692/TCP   9m
dev-rabbitmq-cluster-nodes   ClusterIP   None            <none>        4369/TCP,25672/TCP             9m
```

4. 登陆web UI

````
# 方法：使用NodePort的方式把集群映射到外部访问
````


- 参考文档：`https://www.rabbitmq.com/kubernetes/operator/quickstart-operator.html`