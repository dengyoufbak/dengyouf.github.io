---
layout: post
title: Docker 安装 Elasticsearch 7.X 集群.md
date: 2023-03-11
tags: ElasticSearch
---

## 一、Elasticsearch 简介

> Elasticsearch 是一个分布式搜索引擎，底层基于 Lucene 实现。Elasticsearch 屏蔽了 Lucene 的底层细节，提供了分布式特性，同时对外提供了 Restful API。Elasticsearch 以其易用性迅速赢得了许多用户，被用在网站搜索、日志分析等诸多方面。由于 ES 强大的横向扩展能力，甚至很多人也会直接把 ES 当做 NoSQL 来用。



## 二、实验环境

本次实验采用 docker-compose 的方式部署三节点的 elasticsearch 集群。

- OS：ubuntu20.04
- Docker: 20.10
- Docker-compose: 1.24.1
- Elasticsearch: 7.7.0

| 主机IP | 角色 |
| :--- | --- |
| 192.168.1.111 | master、data |
| 192.168.1.112 | master、data |
| 192.168.1.113 | master、data |

## 三、准备环境

```
# 增加文件描述符
cat >>/etc/security/limits.conf<< EOF
*               soft      nofile          65536
*               hard      nofile          65536
*               soft      nproc           65536
*               hard      nproc           65536
*               hard      memlock         unlimited
*               soft      memlock         unlimited
EOF
```
```
# 修改默认限制内存
cat >>/etc/systemd/system.conf<< EOF
DefaultLimitNOFILE=65536
DefaultLimitNPROC=32000
DefaultLimitMEMLOCK=infinity
EOF
```
```
# 优化内核
cat >>/etc/sysctl.conf<< EOF
# 关闭交换内存
vm.swappiness =0
# 影响java线程数量，建议修改为262144或者更高
vm.max_map_count= 262144
# 优化内核listen连接
net.core.somaxconn=65535
# 最大打开文件描述符数，建议修改为655360或者更高
fs.file-max=655360
# 开启ipv4转发
net.ipv4.ip_forward= 1
EOF
sysctl  --system
```
### 3.1 安装 docker

```
export DOWNLOAD_URL="https://mirrors.tuna.tsinghua.edu.cn/docker-ce"
export VERSION="20.10"
curl -fsSL https://get.docker.com/ | sh

mkdir -pv /opt/docker-root

vim /etc/docker/daemon.json
{
        "registry-mirrors": ["https://o4uba187.mirror.aliyuncs.com"],
        "graph":  "/opt/docker-root"
}

for i in enable restart; do systemctl restart docker ;done
```

### 3.2 安装 docker-compose 

```
curl -L https://get.daocloud.io/docker/compose/releases/download/1.24.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod  +x /usr/local/bin/docker-compose
```

### 3.3 配置 Elasticsearch 

#### 3.3.1 准备相关目录

```
mkdir -pv /data/elasticsearch/{data,logs,plugins,config}

cat >> /etc/security/limits.conf <<EOF
* soft nofile 65536
* hard nofile 65536
EOF

echo "vm.max_map_count=655360" >> /etc/sysctl.conf  && sysctl -p
```

#### 3.3.2 准备配置文件

```
# 节点 192.168.1.111
cat  > /data/elasticsearch/config/elasticsearch.yml <<EOF
cluster.name: dev-es-cls
node.name: es-node01
node.master: true
node.data: true

#network.host: 0.0.0.0
network.bind_host: 0.0.0.0
network.publish_host: 192.168.1.111
http.port: 9200
transport.tcp.port: 9300
http.cors.enabled: true
http.cors.allow-origin: "*"

discovery.zen.ping.unicast.hosts: ["es-node01:9300", "es-node02:9300", "es-node03:9300"]
discovery.zen.minimum_master_nodes: 2
discovery.zen.ping_timeout: 5s

bootstrap.memory_lock: true
action.destructive_requires_name: true
cluster.initial_master_nodes: ["es-node01"]
EOF
```
```
# 节点 192.168.1.112
cat  > /data/elasticsearch/config/elasticsearch.yml <<EOF
cluster.name: dev-es-cls
node.name: es-node02
node.master: true
node.data: true

#network.host: 0.0.0.0
network.bind_host: 0.0.0.0
network.publish_host: 192.168.1.112
http.port: 9200
transport.tcp.port: 9300
http.cors.enabled: true
http.cors.allow-origin: "*"

discovery.zen.ping.unicast.hosts: ["es-node01:9300", "es-node02:9300", "es-node03:9300"]
discovery.zen.minimum_master_nodes: 2
discovery.zen.ping_timeout: 5s

bootstrap.memory_lock: true
action.destructive_requires_name: true
cluster.initial_master_nodes: ["es-node01"]
EOF
```
```
# 节点 192.168.1.113
cat  > /data/elasticsearch/config/elasticsearch.yml <<EOF
cluster.name: dev-es-cls
node.name: es-node03
node.master: true
node.data: true

#network.host: 0.0.0.0
network.bind_host: 0.0.0.0
network.publish_host: 192.168.1.113
http.port: 9200
transport.tcp.port: 9300
http.cors.enabled: true
http.cors.allow-origin: "*"

discovery.zen.ping.unicast.hosts: ["es-node01:9300", "es-node02:9300", "es-node03:9300"]
discovery.zen.minimum_master_nodes: 2
discovery.zen.ping_timeout: 5s

bootstrap.memory_lock: true
action.destructive_requires_name: true
cluster.initial_master_nodes: ["es-node01"]
EOF
```

#### 3.3.3 准备 docker-compose 文件

```
# 节点 192.168.1.111
cat > /data/elasticsearch/docker-compose.yml << EOF
version: '3'
services:
  es-node01:
    image: elasticsearch:7.7.0
    container_name: es-node01
    environment:
      - TZ="Asia/Shanghai"
      - "ES_JAVA_OPTS=-Xms2048m -Xmx2048m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - /data/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
      - /data/elasticsearch/data:/usr/share/elasticsearch/data:rw
      - /data/elasticsearch/logs:/usr/share/elasticsearch/logs:rw
      - /data/elasticsearch/plugins:/usr/share/elasticsearch/plugins:rw
    ports:
      - 9200:9200
      - 9300:9300
    extra_hosts:                        # 设置容器 hosts
      - "es-node01:192.168.1.111"
      - "es-node02:192.168.1.112"
      - "es-node03:192.168.1.113"
  #kibana:
  #  image: kibana:7.7.0
  #  container_name: kibana
  #  restart: always
  #  environment:
  #    - TZ="Asia/Shanghai"
  #  ports:
  #    - 5601:5601
  #  volumes:
  #    - /data/kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml:ro
  #  depends_on:
  #    - es-master
  elasticsearch-head:
    image: mobz/elasticsearch-head:5
    container_name: head
    restart: always
    environment:
      - TZ="Asia/Shanghai"
    ports:
      - 9100:9100
EOF
```
```
# 节点 192.168.1.112
cat > /data/elasticsearch/docker-compose.yml << EOF
version: '3'
services:
  es-node1:
    image: elasticsearch:7.7.0
    container_name: es-node02
    environment:
      - TZ="Asia/Shanghai"
      - "ES_JAVA_OPTS=-Xms2048m -Xmx2048m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - /data/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
      - /data/elasticsearch/data:/usr/share/elasticsearch/data:rw
      - /data/elasticsearch/logs:/usr/share/elasticsearch/logs:rw
      - /data/elasticsearch/plugins:/usr/share/elasticsearch/plugins:rw
    ports:
      - 9200:9200
      - 9300:9300
    extra_hosts:                        # 设置容器 hosts
      - "es-node01:192.168.1.111"
      - "es-node02:192.168.1.112"
      - "es-node03:192.168.1.113"
EOF
```
```
# 节点 192.168.1.113
cat > /data/elasticsearch/docker-compose.yml << EOF
version: '3'
services:
  es-node1:
    image: elasticsearch:7.7.0
    container_name: es-node03
    environment:
      - TZ="Asia/Shanghai"
      - "ES_JAVA_OPTS=-Xms2048m -Xmx2048m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - /data/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
      - /data/elasticsearch/data:/usr/share/elasticsearch/data:rw
      - /data/elasticsearch/logs:/usr/share/elasticsearch/logs:rw
      - /data/elasticsearch/plugins:/usr/share/elasticsearch/plugins:rw
    ports:
      - 9200:9200
      - 9300:9300
    extra_hosts:                        # 设置容器 hosts
      - "es-node01:192.168.1.111"
      - "es-node02:192.168.1.112"
      - "es-node03:192.168.1.113"
EOF
```

## 四、启动服务并验证服务

### 4.1 启动集群

```
chown 1000:1000 /data/elasticsearch/* -R
cd /data/elasticsearch/
docker-compose up -d
```

### 4.2 验证集群状态

```
curl -XGET http://192.168.1.112:9200/_cat/health?v
epoch      timestamp cluster    status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1678509859 04:44:19  dev-es-cls green           3         3      0   0    0    0        0             0                  -                100.0%

curl -XGET http://192.168.1.112:9200/_cat/nodes?v
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
192.168.1.113           11          97   0    0.01    0.04     0.01 dilmrt    -      es-node03
192.168.1.111            7          97   0    0.00    0.04     0.03 dilmrt    *      es-node01
192.168.1.112           24          97   0    0.00    0.01     0.00 dilmrt    -      es-node02
```


## 五、Elasticsearch集群启用 es_xpack 认证

### 5.1 生成证书

```
# 登陆其中一个node节点执行命令，生成完证书复制到集群其他节点即可
docker exec -it  es-node01 bash
/usr/share/elasticsearch/bin/elasticsearch-certutil ca
    ...
Please enter the desired output file [elastic-stack-ca.p12]:
Enter password for elastic-stack-ca.p12 :

/usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
    ...
then the output will be be a zip file containing individual certificate/key files

Enter password for CA (elastic-stack-ca.p12) :
Please enter the desired output file [elastic-certificates.p12]:
Enter password for elastic-certificates.p12 :

Certificates written to /usr/share/elasticsearch/elastic-certificates.p12

This file should be properly secured as it contains the private key for
your instance.
```

```
# 复制证书到宿主机
ls elastic-*
elastic-certificates.p12  elastic-stack-ca.p12
mv elastic-* /usr/share/elasticsearch/data/
```

### 5.2 分发证书到其他主机

```
cp /data/elasticsearch/data/elastic-* /data/elasticsearch/config
chmod 644 /data/elasticsearch/config/elastic-*
scp /data/elasticsearch/config/elastic-* root@192.168.1.112:/data/elasticsearch/config
scp /data/elasticsearch/config/elastic-* root@192.168.1.113:/data/elasticsearch/config
```

### 5.3 增加 es 配置

```
##三台机器新增配置如下：
cat >> /data/elasticsearch/config/elasticsearch.yml << EOF
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/elastic-certificates.p12
EOF
```

### 5.4 挂在证书到容器

```
# 所有节点修改docker-compose.yml 增加ssl证书挂载
version: '3'
services:
  es-node1:
    image: elasticsearch:7.7.0
    container_name: es-node03
    environment:
      - TZ="Asia/Shanghai"
      - "ES_JAVA_OPTS=-Xms2048m -Xmx2048m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - /data/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
      ## 挂载 ssl 证书到容器中
      - /data/elasticsearch/config/elastic-certificates.p12:/usr/share/elasticsearch/config/elastic-certificates.p12:ro
      - /data/elasticsearch/config/elastic-stack-ca.p12:/usr/share/elasticsearch/config/elastic-stack-ca.p12:ro

      - /data/elasticsearch/data:/usr/share/elasticsearch/data:rw
      - /data/elasticsearch/logs:/usr/share/elasticsearch/logs:rw
    ......
```

### 5.5 创建账户，并为内置账号添加密码

```
# 这里密码我全部设置为dengyou@163
docker exec -it es-node01 bash
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,logstash_system,beats_system,remote_monitoring_user.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]y


Enter password for [elastic]:
Reenter password for [elastic]:
Enter password for [apm_system]:
Reenter password for [apm_system]:
Enter password for [kibana]:
Reenter password for [kibana]:
Enter password for [logstash_system]:
Reenter password for [logstash_system]:
Enter password for [beats_system]:
Reenter password for [beats_system]:
Enter password for [remote_monitoring_user]:
Reenter password for [remote_monitoring_user]:
Changed password for user [apm_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]
```

### 访问集群

```
curl -XGET -u elastic:dengyou@163 http://192.168.1.111:9200/_cat/health
1678511968 05:19:28 dev-es-cls green 3 3 2 1 0 0 0 0 - 100.0%

curl -XGET -u elastic:dengyou@163 http://192.168.1.111:9200/_cat/nodes?v
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
192.168.1.113           35          96   0    0.00    0.03     0.00 dilmrt    -      es-node03
192.168.1.111           38          94   1    0.00    0.02     0.00 dilmrt    *      es-node01
192.168.1.112           43          97   0    0.07    0.06     0.01 dilmrt    -      es-node02
```
