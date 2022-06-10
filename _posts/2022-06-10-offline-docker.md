---
layout: post
title: Docker离线安装
date: 2022-06-10
tags: DOCKER
---

## DOCKER离线安装


### 1. 获取程序包

```
wget https://download.docker.com/linux/static/stable/x86_64/docker-20.10.8.tgz
```

### 2. 解压

```
tar -xf docker-20.10.8.tgz 
cp docker/* /usr/bin/
```

### 3. 配置docker

```
mkdir /etc/docker
cat > /etc/docker/daemon.json << EOF
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 10,
    "log-driver": "json-file",
    "storage-driver": "overlay2",
    "data-root": "/data/docker",
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
```

### 4.提供服务管理脚本

```
cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
[Install]
WantedBy=multi-user.target
EOF
```

### 5.启动服务

```
systemctl  daemon-reload
systemctl  enable docker --now
```
