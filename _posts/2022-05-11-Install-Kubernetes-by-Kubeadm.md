---
layout: post
title: 基于Kubeadm搭建Kubernetes集群
date: 2022-05-11
tags: KUBEADM
---

## 主机规划

使用 kubeadm 初始化 kubernetes 集群，操作系统为 CentOS 7.6.1810 x86_64，用到的各相关程序版本如下：

- ubuntu: v18.04.6 LTS
- kubeadm: v1.22.6
- docker: v20.10.6
- flannel: v1.0.1

| 主机名 | IP 地址 | 配置 |
| --- | --- | --- |
| kubeadm-master-01 | 192.168.124.200 | 8GB Ram, 2vcpus |
| kubeadm-node-01 | 192.168.124.211 | 8GB Ram, 2vcpus |
| kubeadm-node-02 | 192.168.124.212 | 8GB Ram, 2vcpus |
| kubeadm-node-03 | 192.168.124.212 | 8GB Ram, 2vcpus |

## 准备

- 升级软件包

```
apt update
apt -y upgrade
```

- 主机名解析

```
cat >> /etc/hosts <<EOF

192.168.124.200 kubeadm-master-01
192.168.124.211 kubeadm-node-01
192.168.124.212 kubeadm-node-02
192.168.124.213 kubeadm-node-03

# 用来扩展集群为高可用集群
192.168.124.200 kubeadm-vip.linux.io
EOF
```

- 时间同步

```
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

apt install ntpdate -y
ntpdate time1.aliyun.com
echo "*/3 * * * * /usr/sbin/ntpdate time1.aliyun.com &> /dev/null" >> /var/spool/cron/crontabs/root
```

- 关闭防火墙和 swap 分区

```
ufw disable
sed -i 's@^/swap@# &1@g' /etc/fstab  && swapoff  -a
```

- 调整内核参数

```
modprobe overlay && modprobe br_netfilter

cat >> /etc/sysctl.d/99-kubernetes-cri.conf << EOF
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
user.max_user_namespaces=28633
vm.swappiness=0
EOF

sysctl  --system
```

- 配置 ipvs 功能

```
apt install ipvsadm ipset -y

modprobe ip_vs && modprobe ip_vs_rr && modprobe ip_vs_wrr && modprobe ip_vs_sh
cat <<EOF >> /etc/modules
ip_vs_rr
ip_vs_wrr
ip_vs_sh
ip_vs
EOF

reboot

lsmod | grep -e ip_vs -e nf_conntrack_ipv4
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 151552  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_defrag_ipv6         20480  1 ip_vs
nf_conntrack          135168  1 ip_vs
libcrc32c              16384  4 nf_conntrack,xfs,raid456,ip_vs
```

## 安装容器运行时

```
apt-get remove docker docker-engine docker.io
apt-get install apt-transport-https ca-certificates curl gnupg2 software-properties-common

curl -fsSL https://repo.huaweicloud.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

add-apt-repository "deb [arch=amd64] https://repo.huaweicloud.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

apt update
```

```
apt-cache madison docker-ce
apt install docker-ce=5:20.10.6~3-0~ubuntu-bionic -y
```

```
mkdir /etc/docker
cat >> /etc/docker/daemon.json << EOF 
{
    "registry-mirrors": ["https://o4uba187.mirror.aliyuncs.com"],
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
    "max-size": "100m"
    },
    "storage-driver": "overlay2",
    "graph":  "/var/lib/docker"
}
EOF

systemctl  enable docker --now
```

## 安装 Kubeadm

```
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
```

```
apt-get update 
apt-get install kubeadm=1.22.5-00 kubelet=1.22.5-00 kubectl=1.22.5-00 
systemctl enable kubelet.service
```

## 初始化 Master 节点

```
kubeadm init --kubernetes-version=v1.22.5 \
    --control-plane-endpoint=kubeadm-vip.linux.io \
    --apiserver-advertise-address=0.0.0.0 \
    --pod-network-cidr=10.244.0.0/16   \
    --service-cidr=10.96.0.0/12 \
    --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers \
    --ignore-preflight-errors=Swap | tee kubeadm-init.log

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join kubeadm-vip.linux.io:6443 --token pa5fyb.mcoy474qbznp8cvd \
	--discovery-token-ca-cert-hash sha256:5c1deff287d32725b8c3eaab49a32afc4f046ff1514056b15f11535c1fb45df5 \
	--control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join kubeadm-vip.linux.io:6443 --token pa5fyb.mcoy474qbznp8cvd \
	--discovery-token-ca-cert-hash sha256:5c1deff287d32725b8c3eaab49a32afc4f046ff1514056b15f11535c1fb45df5
```
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 部署 Flannel

```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```


## Join worker 节点

```
kubeadm join kubeadm-vip.linux.io:6443 --token pa5fyb.mcoy474qbznp8cvd \
	--discovery-token-ca-cert-hash sha256:5c1deff287d32725b8c3eaab49a32afc4f046ff1514056b15f11535c1fb45df5
```

## 启用 IPVS 模式


```
kubectl  edit cm kube-proxy -n kube-system
    mode: "ipvs" 

kubectl  delete pod -n kube-system -l k8s-app=kube-proxy
```
```
ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 192.168.124.200:6443         Masq    1      0          0
TCP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0
  -> 10.244.0.3:53                Masq    1      0          0
TCP  10.96.0.10:9153 rr
  -> 10.244.0.2:9153              Masq    1      0          0
  -> 10.244.0.3:9153              Masq    1      0          0
UDP  10.96.0.10:53 rr
  -> 10.244.0.2:53                Masq    1      0          0
  -> 10.244.0.3:53                Masq    1      0          0
```


## 验证集群

```
kubectl  create deployment myapp --image=ikubernetes/myapp:v1 --replicas=3
kubectl  expose deployment  myapp  --port=80 --target-port=80
```
```
kubectl  get pod -o wide && kubectl get svc
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE              NOMINATED NODE   READINESS GATES
myapp-7d4b7b84b-98cx7   1/1     Running   0          83s   10.244.3.3   kubeadm-node-03   <none>           <none>
myapp-7d4b7b84b-pfkqq   1/1     Running   0          83s   10.244.2.3   kubeadm-node-02   <none>           <none>
myapp-7d4b7b84b-zg5dc   1/1     Running   0          83s   10.244.1.4   kubeadm-node-01   <none>           <none>
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   60m
myapp        ClusterIP   10.101.62.58   <none>        80/TCP    2m40s
```

```
kubectl  exec -it myapp-7d4b7b84b-98cx7 -- wget -O  - 10.244.1.4
Connecting to 10.244.1.4 (10.244.1.4:80)
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>

kubectl  exec -it myapp-7d4b7b84b-98cx7 -- ping 192.168.124.212

 kubectl  exec -it myapp-7d4b7b84b-98cx7 -- nslookup myapp
nslookup: can't resolve '(null)': Name does not resolve

Name:      myapp
Address 1: 10.101.62.58 myapp.default.svc.cluster.local
```

