---
layout: post
title: Prometheus监控平台快速搭建
date: 2022-06-23
tags: PROMETHEUS
---

## Prometheus安装

- 解压程序

```
wget https://github.com/prometheus/prometheus/releases/download/v2.36.2/prometheus-2.36.2.linux-amd64.tar.gz
tar -xf prometheus-2.36.2.linux-amd64.tar.gz  -C /usr/local/
ln -sv /usr/local/prometheus-2.36.2.linux-amd64/ /usr/local/prometheus
```

- 启动服务

```
cd /usr/local/prometheus
./promtool check config prometheus.yml
./prometheus --config.file=./prometheus.yml \
  --web.listen-address="0.0.0.0:9090" \
  --web.enable-lifecycle \
  --storage.tsdb.path="data" \
  --storage.tsdb.retention.time=30d \
  --query.timeout=2m   --log.level=info \
  --log.format=logfmt \
  --storage.tsdb.retention.size=2TB \
  --storage.tsdb.no-lockfile \
  --storage.tsdb.wal-compression  \
  --rules.alert.resend-delay=5s
```

- 提供服务启动脚本

```
useradd  prometheus
chown -R prometheus.prometheus /usr/local/prometheus*
```
```
vim /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
Environment="GOMAXPROCS=1"
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/prometheus/prometheus \
  --config.file=/usr/local/prometheus/prometheus.yml \
  --storage.tsdb.path=/usr/local/prometheus/data \
  --web.console.templates=/usr/local/prometheus/prometheus/consoles \
  --web.console.libraries=/usr/local/prometheus/prometheus/console_libraries \
  --storage.tsdb.retention.time=30d \
  --storage.tsdb.retention.size=2TB \
  --rules.alert.resend-delay=5s \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
```
```
systemctl start prometheus&&systemctl enable prometheus
```
## 安装Node_Exporter

- 解压程序

```
wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
tar -xf node_exporter-1.3.1.linux-amd64.tar.gz  -C /usr/local/
ln -sv /usr/local/node_exporter-1.3.1.linux-amd64/ /usr/local/node_exporter
```

- 提供服务启动脚本

```
useradd node_exporter
mkdir -pv /var/lib/node_exporter/textfile_collector
```
```
vim  /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter

[Service]
User=node_exporter
ExecStart=/usr/local/node_exporter/node_exporter \
    --collectors.enabled meminfo,loadavg,filesystem \
    --collector.textfile.directory /var/lib/node_exporter/textfile_collector

[Install]
WantedBy=multi-user.target
```
```
systemctl start node_exporter && systemctl enable node_exporter
```
## 安装Grafana

- 解压程序

```
wget https://dl.grafana.com/oss/release/grafana-8.4.7.linux-amd64.tar.gz
tar -xf grafana-8.4.7.linux-amd64.tar.gz  -C /usr/local/
ln -sv /usr/local/grafana-8.4.7 /usr/local/grafana
```

- 启动服务

```
cd /usr/local/grafana
nohup ./bin/grafana-server &
```
访问地址：http://IP:3000, 默认访问密码是：admin/admin
