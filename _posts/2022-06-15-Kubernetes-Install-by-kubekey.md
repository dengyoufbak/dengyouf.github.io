---
layout: post
title: 基于KubeKey安装Kubernetes集群
date: 2020-07-11
tags: KUBEKEY
---

## 准备Linux主机

- 系统环境

| 主机系统 | 最低配置 | 内核版本 |
| --- | --- | ---|
|CentOS 7.x | CPU：2 核，内存：4 G，硬盘：40 G | 4.15+ |

| 主机 IP        |	主机名	|角色|
|--------------|---|---|
| 192.168.0.11 |	master|	master, etcd,worker|
| 192.168.0.12 |	node1|	worker|
| 192.168.0.12 |	node2|	worker|


> /var/lib/docker 路径主要用于存储容器数据，在使用和操作过程中数据量会逐渐增加。因此，在生产环境中，建议为 /var/lib/docker 单独挂载一个硬盘

- 节点要求

```
yum install socat conntrack ebtables ipset ipvsadm -y
```

## 下载KubeKey

```
export KKZONE=cn
curl -sfL https://get-kk.kubesphere.io | VERSION=v2.2.0 sh -
chmod +x kk
```

>运行 ./kk version --show-supported-k8s，查看能使用 KubeKey 安装的所有受支持的 Kubernetes 版本 

## 配置规划集群

```
./kk create config --with-kubernetes  v1.23.7  [--with-kubesphere v3.2.1] 
```
```
~]# cat config-sample.yaml

apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: master, address: 192.168.0.11, internalAddress: 192.168.0.11, user: root, password: "dy6545286@DY"}
  - {name: node1, address: 192.168.0.12, internalAddress: 192.168.0.12, user: root, password: "dy6545286@DY"}
  - {name: node2, address: 192.168.0.13, internalAddress: 192.168.0.13, user: root, password: "dy6545286@DY"}
  roleGroups:
    etcd:
    - master
    control-plane: 
    - master
    worker:
    - master
    - node1
    - node2
  controlPlaneEndpoint:
    ## Internal loadbalancer for apiservers 
    internalLoadbalancer: haproxy

    domain: lb.linux.io
    address: ""
    port: 6443
  kubernetes:
    version: version
    clusterName: linux.io
    autoRenewCerts: true
    containerManager: 
  etcd:
    type: kubekey
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
    ## multus support. https://github.com/k8snetworkplumbingwg/multus-cni
    multusCNI:
      enabled: false
  registry:
    privateRegistry: ""
    namespaceOverride: ""
    registryMirrors: []
    insecureRegistries: []
  addons: []
```

## 创建集群

```
 ./kk create cluster -f config-sample.yaml
```

## 验证集群

```
~]# kubectl get pod -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-785fcf8454-fxjc7   1/1     Running   0          11m
kube-system   calico-node-27rbk                          1/1     Running   0          11m
kube-system   calico-node-bx8n9                          1/1     Running   0          11m
kube-system   calico-node-j957l                          1/1     Running   0          11m
kube-system   coredns-757cd945b-7vrxd                    1/1     Running   0          11m
kube-system   coredns-757cd945b-bvzc6                    1/1     Running   0          11m
kube-system   haproxy-node1                              1/1     Running   0          11m
kube-system   haproxy-node2                              1/1     Running   0          11m
kube-system   kube-apiserver-master                      1/1     Running   0          11m
kube-system   kube-controller-manager-master             1/1     Running   0          11m
kube-system   kube-proxy-9cgwm                           1/1     Running   0          11m
kube-system   kube-proxy-gcmck                           1/1     Running   0          11m
kube-system   kube-proxy-hzqnl                           1/1     Running   0          11m
kube-system   kube-scheduler-master                      1/1     Running   0          11m
kube-system   nodelocaldns-97knk                         1/1     Running   0          11m
kube-system   nodelocaldns-ft2cq                         1/1     Running   0          11m
kube-system   nodelocaldns-vxqmz                         1/1     Running   0          11m
```
```
~]# kubectl  get nodes -o wide
NAME     STATUS   ROLES                         AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
master   Ready    control-plane,master,worker   11m   v1.23.7   192.168.0.11   <none>        CentOS Linux 7 (Core)   3.10.0-1127.19.1.el7.x86_64   docker://20.10.8
node1    Ready    worker                        11m   v1.23.7   192.168.0.12   <none>        CentOS Linux 7 (Core)   3.10.0-1127.19.1.el7.x86_64   docker://20.10.8
node2    Ready    worker                        11m   v1.23.7   192.168.0.13   <none>        CentOS Linux 7 (Core)   3.10.0-1127.19.1.el7.x86_64   docker://20.10.8
```
