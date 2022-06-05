---
layout: post
title: Kubernetes部署NFS CSI存储卷
date: 2022-06-05
tags: Kubernetes
---



## 在 Kubernetes 集群中安装  NFS Server

```
kubectl create -f https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/deploy/example/nfs-provisioner/nfs-server.yaml
```
```
kubectl  get pod
NAME                          READY   STATUS    RESTARTS   AGE
nfs-server-594768d8b8-4tctg   1/1     Running   0          5m59s

kubectl  get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)            AGE
kubernetes   ClusterIP   10.43.0.1      <none>        443/TCP            7d23h
nfs-server   ClusterIP   10.43.21.132   <none>        2049/TCP,111/UDP   6m2s
```
## 在 Kubernetes 集群中安装  NFS CSI 驱动

```
git clone https://github.com/kubernetes-csi/csi-driver-nfs.git
cd csi-driver-nfs
cp deploy/rbac-csi-nfs.yaml deploy/v3.1.0
./deploy/install-driver.sh v3.1.0 local
```
```
kubectl -n kube-system get pod -o wide -l app=csi-nfs-controller
NAME                                  READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
csi-nfs-controller-54dc4c6b58-d4drf   3/3     Running   0          20m   192.168.56.113   rke-master-3   <none>           <none>
kubectl -n kube-system get pod -o wide -l app=csi-nfs-node
NAME                 READY   STATUS    RESTARTS        AGE   IP               NODE           NOMINATED NODE   READINESS GATES
csi-nfs-node-2vmtg   3/3     Running   0               20m   192.168.56.111   rke-master-1   <none>           <none>
csi-nfs-node-6f9gs   3/3     Running   0               20m   192.168.56.113   rke-master-3   <none>           <none>
csi-nfs-node-n8tqd   3/3     Running   1 (8m22s ago)   20m   192.168.56.112   rke-master-2   <none>           <none>
csi-nfs-node-qc9zq   3/3     Running   0               20m   192.168.56.212   rke-worker-2   <none>           <none>
csi-nfs-node-tr5jh   3/3     Running   0               20m   192.168.56.211   rke-worker-1   <none>           <none>
```

## 创建 StorageClass 存储类

```
kubectl  apply -f deploy/example/storageclass-nfs.yaml
kubectl  get sc
NAME      PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-csi   nfs.csi.k8s.io   Retain          Immediate           false                  6s
```

## 验证

```
kubectl  apply -f deploy/example/pvc-nfs-csi-dynamic.yaml
kubectl  get pvc
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-nfs-dynamic   Bound    pvc-fe418ef4-d3e1-4c10-b1f9-6b2b815be70d   10Gi       RWX            nfs-csi        19m
kubectl  get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS   REASON   AGE
pvc-fe418ef4-d3e1-4c10-b1f9-6b2b815be70d   10Gi       RWX            Retain           Bound    default/pvc-nfs-dynamic   nfs-csi                 63s
```

```
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-deployment-nfs
spec:
  accessModes:
    - ReadWriteMany  # In this example, multiple Pods consume the same PVC.
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs-csi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-nfs
spec:
  replicas: 1
  selector:
    matchLabels:
      name: deployment-nfs
  template:
    metadata:
      name: deployment-nfs
      labels:
        name: deployment-nfs
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
        - name: deployment-nfs
          image: mcr.microsoft.com/oss/nginx/nginx:1.19.5
          command:
            - "/bin/bash"
            - "-c"
            - set -euo pipefail; while true; do echo $(hostname) $(date) >> /mnt/nfs/outfile; sleep 1; done
          volumeMounts:
            - name: nfs
              mountPath: "/mnt/nfs"
      volumes:
        - name: nfs
          persistentVolumeClaim:
            claimName: pvc-deployment-nfs
```

> 需科学上网获取镜像


## 参考文档
https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/deploy/example/README.md

https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/install-nfs-csi-driver.md
