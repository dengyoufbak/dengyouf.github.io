---
layout: post
title: RKE 构建高可用的 KUBERNETES 集群
tag: RKE
---
## RKE 概述

> Rancher Kubernetes Engine (RKE) 是一个 CNCF 认证的 Kubernetes 发行版，完全在 Docker 容器中运行。它适用于裸机和虚拟化服务器。RKE 解决了安装复杂的问题，这是 Kubernetes 社区的一个常见问题。使用 RKE，Kubernetes 的安装和操作既简单又容易自动化，并且完全独立于您运行的操作系统和平台。只要你能运行受支持的 Docker 版本，你就可以使用 RKE 部署和运行 Kubernetes。


## Kubernetes 集群环境介绍

> 本次使用 CentOS7.9 最小化安装,集群机器规划信息如下表,软件版本信息如下:

- rke: v1.3.11
- kubernetes: v1.23.6
- docker: 20.10.0

| 主机名 | IP 地址 | 机器配置 | 用途 |
| --- | --- | --- | ---|
| rke-lb | 192.168.56.101 | 1C1G | Nginx 
| rke-master-1 | 192.168.56.111 | 4C4G | controlplane,etcd,worker|
| rke-master-2 | 192.168.56.112 | 4C4G | controlplane,etcd,worker|
| rke-master-3 | 192.168.56.113 | 4C4G | controlplane,etcd,worker|
| rke-worker-1 | 192.168.56.211 | 4C4G | worker |
| rke-worker-1 | 192.168.56.212 | 4C4G | worker |

### 机器环境设置

- 主机名解析

```
~]$ sudo cat /etc/hosts
192.168.56.101  rke-lb   rancher-ui.linux.io
192.168.56.111  rke-master-1
192.168.56.112  rke-master-2
192.168.56.113  rke-master-3
192.168.56.211  rke-worker-1
192.168.56.212  rke-worker-2
```

- 安装基础软件包

```
{
 sudo yum remove -y  firewalld python-firewall firewalld-filesystem
 sudo yum install -y bash-completion conntrack-tools ipset ipvsadm libseccomp nfs-utils psmisc socat wget vim net-tools ntpdate
}
```

- 时间同步

```
{
echo '*/5 * * * * /usr/sbin/ntpdate ntp.aliyun.com &> /dev/null' |sudo tee /var/spool/cron//root
}
```

- 关闭Selinux、Firewalld 已以及 Swap 分区

```
{
sudo setenforce 0
sudo sed -i 's@SELINUX=enforcing@SELINUX=disabled@' /etc/selinux/config
sudo swapoff -a && sysctl -w vm.swappiness=0
sudo sed -i '/swap/d' /etc/fstab
sudo systemctl stop firewalld && sudo systemctl disable firewalld
}
```

- 加载内核模块

```
{
for i in br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack;do
    sudo modprobe $i
done
sudo modprobe nf_conntrack_ipv4 || echo "NoFound"
sudo echo 'br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
nf_conntrack_ipv4'|sudo tee  /etc/modules-load.d/10-k8s-modules.conf
sudo systemctl  enable systemd-modules-load --now
}
```

- 修改/添加内核参数

```
{
sudo echo 'net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-arptables = 1
# KERNEL_VER < 4.12
#net.ipv4.tcp_tw_recycle = 0
#net.ipv4.tcp_tw_reuse = 0
net.core.somaxconn = 32768
net.netfilter.nf_conntrack_max=1000000
vm.swappiness = 0
vm.max_map_count=655360
fs.file-max=6553600
# PROXY_MODE == "ipvs"
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10'|sudo tee -a /etc/sysctl.d/95-k8s-sysctl.conf
sudo sysctl -p /etc/sysctl.d/95-k8s-sysctl.conf
}
# 禁用sctp
{
sudo echo '# put sctp into blacklist
install sctp /bin/true'|sudo tee /etc/modprobe.d/sctp.conf
}
```

- # 修改 ulimits 参数

```
{
sudo mkdir -pv /etc/systemd/system.conf.d/
echo '[Manager]
DefaultLimitCORE=infinity
DefaultLimitNOFILE=100000
DefaultLimitNPROC=100000'|sudo tee /etc/systemd/system.conf.d/30-k8s-ulimits.conf
}
```

## 安装容器运行时

```
{
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
sudo yum makecache fast
sudo yum -y install docker-ce-20.10.0
}
```

```
{
sudo mkdir /etc/docker  && sudo mkdir -pv /data/docker-root
echo '{
    "oom-score-adjust": -1000,
    "log-driver": "json-file",
    "log-opts": {
    "max-size": "100m",
    "max-file": "3"
    },
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 10,
    "bip": "172.20.1.0/16",
    "storage-driver": "overlay2",
    "storage-opts": [
    "overlay2.override_kernel_check=true"
    ],
    "registry-mirrors": ["https://o4uba187.mirror.aliyuncs.com"],
    "data-root":  "/data/docker-root",
    "exec-opts": ["native.cgroupdriver=systemd"]
}'|sudo tee /etc/docker/daemon.json
sudo systemctl enable docker --now
}
```

## 安装 kubernetes 集群

- 创建普通用户

```
{
useradd vagrant
echo "vagrant"|passwd --stdin vagrant
echo "vagrant  ALL=(ALL)       NOPASSWD: ALL" >> /etc/sudoers.d/vagrant
}
```
```
{
sudo usermod -aG docker vagrant
sudo chown  vagrant.vagrant /var/run/docker.sock
}
```


- 主机互信

```
sudo yum install -y sshpass
cat ssh-copy.sh
#!/bin/bash
#
IP="192.168.56.111
192.168.56.112
192.168.56.113
192.168.56.211
192.168.56.212"

#ssh-keygen -t rsa -b 2048 -N '' -f ~/.ssh/id_rsa

for node in ${IP};do
    sshpass -p vagrant ssh-copy-id  ${node}  -o StrictHostKeyChecking=no
    ssh $node "sudo chown vagrant.vagrant /var/run/docker.sock "
    if [ $? -eq 0 ];then
        echo -e "\033[46;31m ${node} 秘钥copy成功 \033[0m"
    else
        echo -e "\033[43;31m ${node} 秘钥copy失败 \033[0m"
    fi
done

bash ssh-copy.sh
```

- 安装 rke

```
{
sudo wget https://rancher-mirror.rancher.cn/rke/v1.3.11/rke_linux-amd64  -o /usr/local/bin/rke
sudo chown vagrant.vagrant  /usr/local/bin/rke && chmod +x /usr/local/bin/rke
}
```

- 初始化集群

```
]$ rke config
[+] Cluster Level SSH Private Key Path [~/.ssh/id_rsa]:
[+] Number of Hosts [1]: 5
[+] SSH Address of host (1) [none]: 192.168.56.111
[+] SSH Port of host (1) [22]:
[+] SSH Private Key Path of host (192.168.56.111) [none]:
[-] You have entered empty SSH key path, trying fetch from SSH key parameter
[+] SSH Private Key of host (192.168.56.111) [none]:
[-] You have entered empty SSH key, defaulting to cluster level SSH key: ~/.ssh/id_rsa
[+] SSH User of host (192.168.56.111) [ubuntu]: vagrant
[+] Is host (192.168.56.111) a Control Plane host (y/n)? [y]: y
[+] Is host (192.168.56.111) a Worker host (y/n)? [n]: y
[+] Is host (192.168.56.111) an etcd host (y/n)? [n]: y
[+] Override Hostname of host (192.168.56.111) [none]: rke-master-1
[+] Internal IP of host (192.168.56.111) [none]: 192.168.56.111
[+] Docker socket path on host (192.168.56.111) [/var/run/docker.sock]:
[+] SSH Address of host (2) [none]:
...
[+] Docker socket path on host (192.168.56.212) [/var/run/docker.sock]:
[+] Network Plugin Type (flannel, calico, weave, canal, aci) [canal]: canel
[+] Authentication Strategy [x509]:
[+] Authorization Mode (rbac, none) [rbac]:
[+] Kubernetes Docker image [rancher/hyperkube:v1.23.6-rancher1]: registry.cn-hangzhou.aliyuncs.com/rancher/hyperkube:v1.23.6-rancher1
[+] Cluster domain [cluster.local]:
[+] Service Cluster IP Range [10.43.0.0/16]:
[+] Enable PodSecurityPolicy [n]:
[+] Cluster Network CIDR [10.42.0.0/16]:
[+] Cluster DNS Service IP [10.43.0.10]:
[+] Add addon manifest URLs or YAML files [no]:
```

```
# 完整配置文件
cat cluster.yml                                                   
# If you intended to deploy Kubernetes in an air-gapped environment
# please consult the documentation on how to configure custom RKE images.
nodes:
- address: 192.168.56.111
  port: "22"
  internal_address: 192.168.56.111
  role:
  - controlplane
  - worker
  - etcd
  hostname_override: rke-master-1
  user: vagrant
  docker_socket: /var/run/docker.sock
  ssh_key: ""
  ssh_key_path: ~/.ssh/id_rsa
  ssh_cert: ""
  ssh_cert_path: ""
  labels: {}
  taints: []
- address: 192.168.56.112
  port: "22"
  internal_address: 192.168.56.112
  role:
  - controlplane
  - worker
  - etcd
  hostname_override: rke-master-2
  user: vagrant
  docker_socket: /var/run/docker.sock
  ssh_key: ""
  ssh_key_path: ~/.ssh/id_rsa
  ssh_cert: ""
  ssh_cert_path: ""
  labels: {}
  taints: []
- address: 192.168.56.113
  port: "22"
  internal_address: 192.168.56.113
  role:
  - controlplane
  - worker
  - etcd
  hostname_override: rke-master-3
  user: vagrant
  docker_socket: /var/run/docker.sock
  ssh_key: ""
  ssh_key_path: ~/.ssh/id_rsa
  ssh_cert: ""
  ssh_cert_path: ""
  labels: {}
  taints: []
- address: 192.168.56.211
  port: "22"
  internal_address: 192.168.56.211
  role:
  - worker
  hostname_override: rke-worker-1
  user: vagrant
  docker_socket: /var/run/docker.sock
  ssh_key: ""
  ssh_key_path: ~/.ssh/id_rsa
  ssh_cert: ""
  ssh_cert_path: ""
  labels:
    app: ingress
  taints: []
- address: 192.168.56.212
  port: "22"
  internal_address: 192.168.56.212
  role:
  - worker
  hostname_override: rke-worker-2
  user: vagrant
  docker_socket: /var/run/docker.sock
  ssh_key: ""
  ssh_key_path: ~/.ssh/id_rsa
  ssh_cert: ""
  ssh_cert_path: ""
  labels:
    app: ingress
  taints: []
services:
  etcd:
    image: ""
    extra_args: {}
    extra_binds: []
    extra_env: []
    win_extra_args: {}
    win_extra_binds: []
    win_extra_env: []
    external_urls: []
    ca_cert: ""
    cert: ""
    key: ""
    path: ""
    uid: 0
    gid: 0
    snapshot: null
    retention: ""
    creation: ""
    backup_config: null
  kube-api:
    image: ""
    extra_args: {}
    extra_binds: []
    extra_env: []
    win_extra_args: {}
    win_extra_binds: []
    win_extra_env: []
    service_cluster_ip_range: 10.43.0.0/16
    service_node_port_range: ""
    pod_security_policy: false
    always_pull_images: false
    secrets_encryption_config: null
    audit_log: null
    admission_configuration: null
    event_rate_limit: null
  kube-controller:
    image: ""
    extra_args: {}
    extra_binds: []
    extra_env: []
    win_extra_args: {}
    win_extra_binds: []
    win_extra_env: []
    cluster_cidr: 10.42.0.0/16
    service_cluster_ip_range: 10.43.0.0/16
  scheduler:
    image: ""
    extra_args: {}
    extra_binds: []
    extra_env: []
    win_extra_args: {}
    win_extra_binds: []
    win_extra_env: []
  kubelet:
    image: ""
    extra_args: {}
    extra_binds: []
    extra_env: []
    win_extra_args: {}
    win_extra_binds: []
    win_extra_env: []
    cluster_domain: cluster.local
    infra_container_image: ""
    cluster_dns_server: 10.43.0.10
    fail_swap_on: false
    generate_serving_certificate: false
  kubeproxy:
    image: ""
    extra_args: {}
    extra_binds: []
    extra_env: []
    win_extra_args: {}
    win_extra_binds: []
    win_extra_env: []
network:
  plugin: canal
  # 指定网络接口
  options:
    canal_iface: eth1
  mtu: 0
  node_selector: {}
  update_strategy: null
  tolerations: []
authentication:
  strategy: x509
  sans: []
  webhook: null
addons: ""
addons_include: []
system_images:
  etcd: rancher/mirrored-coreos-etcd:v3.5.3
  alpine: rancher/rke-tools:v0.1.80
  nginx_proxy: rancher/rke-tools:v0.1.80
  cert_downloader: rancher/rke-tools:v0.1.80
  kubernetes_services_sidecar: rancher/rke-tools:v0.1.80
  kubedns: rancher/mirrored-k8s-dns-node-cache:1.21.1
  dnsmasq: rancher/mirrored-k8s-dns-dnsmasq-nanny:1.21.1
  kubedns_sidecar: rancher/mirrored-k8s-dns-sidecar:1.21.1
  kubedns_autoscaler: rancher/mirrored-cluster-proportional-autoscaler:1.8.5
  coredns: rancher/mirrored-coredns-coredns:1.9.0
  coredns_autoscaler: rancher/mirrored-cluster-proportional-autoscaler:1.8.5
  nodelocal: rancher/mirrored-k8s-dns-node-cache:1.21.1
  kubernetes: registry.cn-hangzhou.aliyuncs.com/rancher/hyperkube:v1.23.6-rancher1
  flannel: rancher/mirrored-coreos-flannel:v0.15.1
  flannel_cni: rancher/flannel-cni:v0.3.0-rancher6
  calico_node: rancher/mirrored-calico-node:v3.22.0
  calico_cni: rancher/mirrored-calico-cni:v3.22.0
  calico_controllers: rancher/mirrored-calico-kube-controllers:v3.22.0
  calico_ctl: rancher/mirrored-calico-ctl:v3.22.0
  calico_flexvol: rancher/mirrored-calico-pod2daemon-flexvol:v3.22.0
  canal_node: rancher/mirrored-calico-node:v3.22.0
  canal_cni: rancher/mirrored-calico-cni:v3.22.0
  canal_controllers: rancher/mirrored-calico-kube-controllers:v3.22.0
  canal_flannel: rancher/mirrored-flannelcni-flannel:v0.17.0
  canal_flexvol: rancher/mirrored-calico-pod2daemon-flexvol:v3.22.0
  weave_node: weaveworks/weave-kube:2.8.1
  weave_cni: weaveworks/weave-npc:2.8.1
  pod_infra_container: rancher/mirrored-pause:3.6
  ingress: rancher/nginx-ingress-controller:nginx-1.2.0-rancher1
  ingress_backend: rancher/mirrored-nginx-ingress-controller-defaultbackend:1.5-rancher1
  ingress_webhook: rancher/mirrored-ingress-nginx-kube-webhook-certgen:v1.1.1
  metrics_server: rancher/mirrored-metrics-server:v0.6.1
  windows_pod_infra_container: rancher/mirrored-pause:3.6
  aci_cni_deploy_container: noiro/cnideploy:5.1.1.0.1ae238a
  aci_host_container: noiro/aci-containers-host:5.1.1.0.1ae238a
  aci_opflex_container: noiro/opflex:5.1.1.0.1ae238a
  aci_mcast_container: noiro/opflex:5.1.1.0.1ae238a
  aci_ovs_container: noiro/openvswitch:5.1.1.0.1ae238a
  aci_controller_container: noiro/aci-containers-controller:5.1.1.0.1ae238a
  aci_gbp_server_container: noiro/gbp-server:5.1.1.0.1ae238a
  aci_opflex_server_container: noiro/opflex-server:5.1.1.0.1ae238a
ssh_key_path: ~/.ssh/id_rsa
ssh_cert_path: ""
ssh_agent_auth: false
authorization:
  mode: rbac
  options: {}
ignore_docker_version: null
enable_cri_dockerd: null
kubernetes_version: ""
private_registries: []
ingress:
  provider: nginx
  network_mode: hostNetwork
  use-forwarded-headers: 'true'
  options: {}
  node_selector:
    app: ingress
  extra_args: {}
  dns_policy: ""
  extra_envs: []
  extra_volumes: []
  extra_volume_mounts: []
  update_strategy: null
  http_port: 0
  https_port: 0
  tolerations: []
  default_backend: null
  default_http_backend_priority_class_name: ""
  nginx_ingress_controller_priority_class_name: ""
  default_ingress_class: null
cluster_name: ""
cloud_provider:
  name: ""
prefix_path: ""
win_prefix_path: ""
addon_job_timeout: 0
bastion_host:
  address: ""
  port: ""
  user: ""
  ssh_key: ""
  ssh_key_path: ""
  ssh_cert: ""
  ssh_cert_path: ""
  ignore_proxy_env_vars: false
monitoring:
  provider: ""
  options: {}
  node_selector: {}
  update_strategy: null
  replicas: null
  tolerations: []
  metrics_server_priority_class_name: ""
restore:
  restore: false
  snapshot_name: ""
rotate_encryption_key: false
dns: null
```

- 启动集群

```
rke up --config cluster.yml
```

- 更新集群

```
rke up --update-only --config cluster.yml
```

## 验证 kubernete 集群

- 安装 kubectl 工具

```
sudo wget -O /usr/local/bin/kubectl  https://rancher-mirror.rancher.cn/kubectl/v1.23.6/linux-amd64-v1.23.6-kubectl
sudo chown vagrant.vagrant /usr/local/bin/kubectl
sudo chmod +x /usr/local/bin/kubectl
```

- 配置认证文件

```
mkdir -pv ~/.kube
cp kube_config_cluster.yml ~/.kube/config
```

- 查看资源详情

```

kubectl  get nodes
NAME           STATUS   ROLES                      AGE   VERSION
rke-master-1   Ready    controlplane,etcd,worker   10m   v1.23.6
rke-master-2   Ready    controlplane,etcd,worker   10m   v1.23.6
rke-master-3   Ready    controlplane,etcd,worker   10m   v1.23.6
rke-worker-1   Ready    worker                     10m   v1.23.6
rke-worker-2   Ready    worker                     10m   v1.23.6

kubectl  get pods -A
NAMESPACE       NAME                                      READY   STATUS      RESTARTS   AGE
ingress-nginx   ingress-nginx-admission-create-82prb      0/1     Completed   0          9m37s
ingress-nginx   ingress-nginx-admission-patch-x6zk5       0/1     Completed   3          9m37s
ingress-nginx   nginx-ingress-controller-p6z77            1/1     Running     0          9m37s
ingress-nginx   nginx-ingress-controller-zpdct            1/1     Running     0          9m37s
kube-system     calico-kube-controllers-fc7fcb565-bt6fp   1/1     Running     0          9m56s
kube-system     canal-27btn                               2/2     Running     0          9m56s
kube-system     canal-d6tff                               2/2     Running     0          9m57s
kube-system     canal-htqtg                               2/2     Running     0          9m56s
kube-system     canal-tg4wq                               2/2     Running     0          9m56s
kube-system     canal-zn8gr                               2/2     Running     0          9m56s
kube-system     coredns-548ff45b67-4d4mv                  1/1     Running     0          8m42s
kube-system     coredns-548ff45b67-llqkk                  1/1     Running     0          9m47s
kube-system     coredns-autoscaler-d5944f655-5rrmn        1/1     Running     0          9m47s
kube-system     metrics-server-5c4895ffbd-ws8mx           1/1     Running     0          9m42s
kube-system     rke-coredns-addon-deploy-job-rvmp5        0/1     Completed   0          9m49s
kube-system     rke-ingress-controller-deploy-job-sczs5   0/1     Completed   0          9m39s
kube-system     rke-metrics-addon-deploy-job-79mth        0/1     Completed   0          9m44s
kube-system     rke-network-plugin-deploy-job-vwjhc       0/1     Completed   0          9m59s
```

- 部署应用

```
kubectl create deployment myapp --image=ikubernetes/myapp:v1 --replicas=3
kubectl  expose deployment myapp --port=80 --target-port=80 --type=ClusterIP

~]$ kubectl  get all   -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP          NODE           NOMINATED NODE   READINESS GATES
pod/myapp-9cbc4cf76-6g7gz   1/1     Running   0          2m26s   10.42.0.2   rke-master-3   <none>           <none>
pod/myapp-9cbc4cf76-b4vtp   1/1     Running   0          2m26s   10.42.1.3   rke-master-2   <none>           <none>
pod/myapp-9cbc4cf76-cp85t   1/1     Running   0          2m26s   10.42.2.3   rke-master-1   <none>           <none>

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE    SELECTOR
service/kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP   16m    <none>
service/myapp        ClusterIP   10.43.222.216   <none>        80/TCP    103s   app=myapp

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                 SELECTOR
deployment.apps/myapp   3/3     3            3           2m26s   myapp        ikubernetes/myapp:v1   app=myapp

NAME                              DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES                 SELECTOR
replicaset.apps/myapp-9cbc4cf76   3         3         3       2m26s   myapp        ikubernetes/myapp:v1   app=myapp,pod-template-hash=9cbc4cf76
```
```
[vagrant@rke-master-1 ~]$  curl 10.42.0.2
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
[vagrant@rke-master-1 ~]$ curl 10.42.1.3
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
[vagrant@rke-master-1 ~]$ curl 10.42.2.3
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
[vagrant@rke-master-1 ~]$ curl 10.43.222.216
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
```
```
~]$ kubectl  exec -it pod/myapp-9cbc4cf76-6g7gz -- nslookup myapp
nslookup: can't resolve '(null)': Name does not resolve

Name:      myapp
Address 1: 10.43.222.216 myapp.default.svc.cluster.local
```





## 参考

- 调优: https://docs.rancher.cn/docs/rancher2/best-practices/optimize/os/_index
- 国内: https://docs.rancher.cn/docs/rancher2/best-practices/use-in-china/_index
- Kubernetes 集群设置: https://docs.rancher.cn/docs/rke/config-options/_index/

