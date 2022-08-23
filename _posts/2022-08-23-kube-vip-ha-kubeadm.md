---
layout: post
title: 基于Kube-VIP实现Kubernetes集群高可用
date: 2022-08-23
tags: KUBEADM
---


基于Kubeadm部署Kubernetes集群，使用kube-vip实现集群高可用。
- System: Ubuntu 18.04.6 LTS(bionic)
- Kubernetes: v1.23.7
- Docker: 20.10.10
- kube-vip: v0.5.0
- cni: Calico

# 主机基础准备

1. 主机名解析

```
cat >> /etc/hosts <<EOF
192.168.1.111 k8s-master01
192.168.1.112 k8s-master02
192.168.1.113 k8s-master03
192.168.1.121 k8s-node01
192.168.1.122 k8s-node02
192.168.1.188 k8s-vip
EOF
```

2. 禁用swap分区

```
sed -i 's@^/swap@#/swap@g' /etc/fstab 
swapoff -a && sysctl -w vm.swappiness=0
```
3. 安装相关软件包

```
apt remove -y ufw lxd lxd-client lxcfs lxc-common
apt install -y bash-completion conntrack ipset ipvsadm jq libseccomp2 nfs-common psmisc rsync socat  
```
4. 加载内核模块

```
modprobe br_netfilter
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack
modprobe nf_conntrack_ipv4
```

```
cat > /etc/modules-load.d/10-k8s-modules.conf <<EOF
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
nf_conntrack_ipv4
EOF
```

5. 设置系统参数

```
cat >> /etc/sysctl.d/95-k8s-sysctl.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-arptables = 1
net.ipv4.tcp_tw_reuse = 0
net.core.somaxconn = 32768
net.netfilter.nf_conntrack_max=1000000
vm.swappiness = 0
vm.max_map_count=655360
fs.file-max=6553600
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
EOF

sysctl -p /etc/sysctl.d/95-k8s-sysctl.conf
```

6. 设置系统 ulimits

```
mkdir /etc/systemd/system.conf.d

cat >> /etc/systemd/system.conf.d/30-k8s-ulimits.conf << EOF
[Manager]
DefaultLimitCORE=infinity
DefaultLimitNOFILE=100000
DefaultLimitNPROC=100000
EOF
```

7. 时间同步

```
*/3 * * * * /usr/sbin/ntpdate ntp.aliyun.com  > /dev/null
```

# 安装容器运行时-Docker

1. 安装依赖包

```
sudo apt-get remove docker docker-engine docker.io containerd runc -y
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

2. 添加存储库

```
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

sudo apt update
```

3. 安装指定版本的docekr

```
sudo apt install  docker-ce=5:20.10.10~3-0~ubuntu-bionic -y
```

4. 配置并启动docker

```
cat <<EOF | sudo tee /etc/docker/daemon.json
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 10,
    "log-driver": "json-file",
    "storage-driver": "overlay2",
    "storage-opts": [
    "overlay2.override_kernel_check=true"
    ],
    "log-opts":{
        "max-size": "300m",
        "max-file": "2" 
    },
    "registry-mirrors": ["https://o4uba187.mirror.aliyuncs.com"],
    "live-restore": true
}
EOF

for i in enable restart ;do systemctl  $i docker;done
```

# 安装Kubernetes集群

1. 添加kubeadm存储库

```
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
```

2. 安装kubeadm相关软件包

```
sudo apt update 
sudo apt-cache madison kubeadm

sudo apt install -y kubeadm=1.23.7-00 kubelet=1.23.7-00 kubectl=1.23.7-00

sudo apt-mark hold kubelet kubeadm kubectl
```

3. 在所有master节点生成kube-vip的配置文件

```
export VIP=192.168.1.188
export INTERFACE=eth0
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name") # 获取最新版本
echo $KVVERSION # v0.5.0
alias kube-vip="docker run --network host --rm ghcr.io/kube-vip/kube-vip:$KVVERSION"
kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --services \
    --arp \
    --leaderElection | tee /etc/kubernetes/manifests/kube-vip.yaml 
```

4. kubeadm初始化master01节点

```
kubeadm init --kubernetes-version=v1.23.7 \
    --control-plane-endpoint=k8s-vip \
    --apiserver-advertise-address=0.0.0.0 \
    --pod-network-cidr=10.244.0.0/16   \
    --service-cidr=10.96.0.0/12 \
    --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers \
    --ignore-preflight-errors=Swap \
    --token-ttl 0 \
    --upload-certs  | tee kubeadm-init.log
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

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8s-vip:6443 --token q8u6ol.845izym2xjgrp4ll \
        --discovery-token-ca-cert-hash sha256:dbedd3c30ca252ec5fb0d583e52c59bf328d07d76f001563c327e81646519a4a \
        --control-plane --certificate-key 1c4dbef71f5e56a97af1d6b7c74b6ac7ae5eeba0a7ec4303e3ad840011843d0d

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s-vip:6443 --token q8u6ol.845izym2xjgrp4ll \
        --discovery-token-ca-cert-hash sha256:dbedd3c30ca252ec5fb0d583e52c59bf328d07d76f001563c327e81646519a4a 
```

5. 配置kubectl客户端

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl  get nodes
NAME           STATUS     ROLES                  AGE     VERSION
k8s-master01   NotReady   control-plane,master   3m37s   v1.23.7
```

6. Join 其他master节点

```
kubeadm join k8s-vip:6443 --token q8u6ol.845izym2xjgrp4ll \
        --discovery-token-ca-cert-hash sha256:dbedd3c30ca252ec5fb0d583e52c59bf328d07d76f001563c327e81646519a4a \
        --control-plane --certificate-key 1c4dbef71f5e56a97af1d6b7c74b6ac7ae5eeba0a7ec4303e3ad840011843d0d
```

7. Join Worker节点

```
kubeadm join k8s-vip:6443 --token q8u6ol.845izym2xjgrp4ll \
        --discovery-token-ca-cert-hash sha256:dbedd3c30ca252ec5fb0d583e52c59bf328d07d76f001563c327e81646519a4a 
```

8. 修改网络模式为IPVS

```
kubectl edit configmap kube-proxy -n kube-system
    kind: KubeProxyConfiguration
    metricsBindAddress: ""
    mode: "ipvs"
kubectl  delete pod -n kube-system -l k8s-app=kube-proxy
```

9. 部署Calico网络插件

```
curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O calico.yaml
cp calico.yaml{,.bak}
diff calico.yaml calico.yaml.bak 
4546,4547c4546,4547
<             - name: CALICO_IPV4POOL_CIDR
<               value: "10.244.0.0/16"
---
>             # - name: CALICO_IPV4POOL_CIDR
>             #   value: "192.168.0.0/16"

kubectl  apply -f calico.yaml
```

# 验证集群

```
kubectl  create deployment myapp --image=ikubernetes/myapp:v1 --replicas=5
kubectl  get pods  -o wide
NAME                    READY   STATUS    RESTARTS        AGE   IP               NODE           NOMINATED NODE   READINESS GATES
myapp-9cbc4cf76-77gr6   1/1     Running   2 (8m10s ago)   28m   10.244.122.133   k8s-master02   <none>           <none>
myapp-9cbc4cf76-ccsp4   1/1     Running   2 (8m37s ago)   28m   10.244.58.203    k8s-node02     <none>           <none>
myapp-9cbc4cf76-pcb26   1/1     Running   2 (8m15s ago)   28m   10.244.32.133    k8s-master01   <none>           <none>
myapp-9cbc4cf76-t4pn9   1/1     Running   2 (8m52s ago)   28m   10.244.85.200    k8s-node01     <none>           <none>
myapp-9cbc4cf76-vkkwj   1/1     Running   3 (98s ago)     28m   10.244.195.6     k8s-master03   <none>           <none>
for i in 10.244.122.133 10.244.58.203 10.244.32.133 10.244.85.200 10.244.195.6;do curl $i;done
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>

kubectl  expose deployment  myapp --port=80 --target-port=80
kubectl  get svc myapp
NAME    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
myapp   ClusterIP   10.101.72.39   <none>        80/TCP    27s
for i in `seq 5`;do curl 10.101.72.39;done
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
for i in `seq 5`;do curl 10.101.72.39/hostname.html;done
myapp-9cbc4cf76-t4pn9
myapp-9cbc4cf76-ccsp4
myapp-9cbc4cf76-pcb26
myapp-9cbc4cf76-vkkwj
myapp-9cbc4cf76-77gr6

kubectl run -i -t busybox --image=busybox:1.28 -- sh
If you don't see a command prompt, try pressing enter.
/ # nslookup  myapp
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      myapp
Address 1: 10.101.72.39 myapp.default.svc.cluster.local
/ # curl myapp
sh: curl: not found
/ # wget -O - -q myapp
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
/ # wget -O - -q myapp/hostname.html
myapp-9cbc4cf76-ccsp4
```

