---
layout: post
title: 高性能对象存储MinIO入门
date: 2022-06-13
tags: MINIO
---

## 单机版本

###  二进制部署

- 下载二进制程序

```
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/
```

- 启动 MinIO 服务器

```
export MINIO_ROOT_USER=minio
export MINIO_ROOT_PASSWORD=miniopasswd
/usr/local/bin/minio server  --address 192.168.33.11:9000   --console-address 192.168.33.11:9090 /data/minio-data/

API: http://192.168.33.11:9000
RootUser: minio
RootPass: miniopasswd

Console: http://192.168.33.11:9090
RootUser: minio
RootPass: miniopasswd

Command-line: https://docs.min.io/docs/minio-client-quickstart-guide
   $ mc alias set myminio http://192.168.33.11:9000 minio miniopasswd
Documentation: https://docs.min.io
```
- 提供服务启动脚本

```
cat > /etc/systemd/system/minio.service << EOF
[Unit]
Description=minio
After=network.target
After=syslog.target

[Install]
WantedBy=multi-user.target

[Service]
WorkingDirectory=/usr/local/
ExecStart=/usr/local/bin/minio server \
    --address 192.168.33.11:9000 \
    --console-address 192.168.33.11:9090 \
      /data/minio-data/
Restart=always
Environment=MINIO_ROOT_USER=minio
Environment=MINIO_ROOT_PASSWORD=miniopasswd
EOF
```

## 集群版本

> 生产环境强烈建议至少四台机器，这也是官方的建议要求，这样的话就可以做到挂掉一台机器集群 依然可以读写，挂掉两台机器集群依然可读。

###  二进制部署

| 主机名 | IP 地址|  磁盘路径 |
| --- | --- | --- |
| minio-1 | 192.168.33.11 | /data/disk{1,2,3,4}/minio|
| minio-2 | 192.168.33.12 | /data/disk{1,2,3,4}/minio|
| minio-3 | 192.168.33.12 |  /data/disk{1,2,3,4}/minio |
| minio-4 | 192.168.33.14 | /data/disk{1,2,3,4}/minio |
| nginx | 192.168.33.20 |  

- 准备数据目录

```
mkdir -pv /data/disk{1,2,3,4}/minio
groupadd -r minio-user
useradd -M -r -g minio-user minio-user
chown minio-user:minio-user /data/disk{1,2,3,4} -R
```

-  创建服务环境文件

```
cat > /etc/default/minio << EOF
MINIO_VOLUMES="http://192.168.33.{11...14}:9000/data/disk{1...4}/minio"
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=miniopasswd
MINIO_SERVER_URL="http://192.168.33.20:9000"
EOF
```

- 准备启动脚本

```
vim  /etc/systemd/system/minio.service
[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/usr/local/

User=minio-user
Group=minio-user
ProtectProc=invisible

EnvironmentFile=/etc/default/minio
ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

# Let systemd restart this service always
Restart=always

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=1048576

# Specifies the maximum number of threads this process can create
TasksMax=infinity

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=infinity
SendSIGKILL=no
[Install]
WantedBy=multi-user.target
```

- 启动服务

```
 systemctl  start minio
```

- 配置nginx代理

```
cat /etc/nginx/conf.d/minio.conf
upstream minio_cluster {
    server 192.168.33.11:9000;
    server 192.168.33.12:9000;
    server 192.168.33.13:9000;
    server 192.168.33.14:9000;
}

server {
    listen 9000;
    server_name 192.168.33.20 localhost;

    # To allow special characters in headers
    ignore_invalid_headers off;

    # Allow any size file to be uploaded.
    # Set to a value such as 1000m; to restrict file size to a specific value
    client_max_body_size 100m;

    # To disable buffering
    proxy_buffering off;

    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;

        proxy_connect_timeout 300;

        # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        chunked_transfer_encoding off;

        proxy_pass http://minio_cluster;
    }
}
```

