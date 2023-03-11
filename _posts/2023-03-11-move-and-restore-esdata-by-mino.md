---
layout: post
title: Elasticsearch集成MinIO实现数据迁移
date: 2023-03-11
tags: Elasticsearch
---
## 一、实验环境

本次实验采用 docker-compose 的方式部署 elasticsearch 实例和 MinIO 实例，通过备份数据到 MinIO 实现将 elasticsearch-A 实例的数据迁移至 elasticsearch-B 实例。

- OS：ubuntu20.04
- Docker: 20.10
- Docker-compose: 1.24.1
- Elasticsearch: 7.7.0
- MinIO: 

| 主机IP | 角色 |
| :--- | --- |
| 192.168.1.111 | elsticsearch-A |
| 192.168.1.250 | MinIO |
| 192.168.1.250 | elasticsearch-B |

## 二、安装服务

### 2.1 安装 MinIO

```
mkdir -pv /data/apps/minio/data 
cat > /data/apps/minio/docker-compose.yml <<EOF
version: '3'
services:
  minio:
    image: minio/minio
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - /data/apps/minio/data:/data
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: MinIO_Pass-2023
    command: server --console-address ":9001" /data

volumes:
  minio_storage: {}
EOF
```

```
cd /data/apps/minio/ && docker-compose up -d
```

### 2.2 安装 Elasticseach

```
mkdir -pv /data/apps/elasticsearch/{data,plugins,logs,config}

cat > /data/apps/elasticsearch/docker-compose.yml <<EOF
version: '3'
services:
  elasticsearch:
    image: elasticsearch:7.7.0  #镜像
    container_name: elasticsearch #定义容器名称
    restart: always  #开机启动，失败也会一直重启
    environment: {}
    volumes:
      - /data/apps/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - /data/apps/elasticsearch/config/jvm.options:/usr/share/elasticsearch/config/jvm.options
      - /data/apps/elasticsearch/logs:/usr/share/elasticsearch/logs
      - /data/apps/elasticsearch/plugins:/usr/share/elasticsearch/plugins
      - /data/apps/elasticsearch/data:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
      - 9300:9300
    extra_hosts:                        # 设置容器 hosts
      - "elasticsearch:192.168.1.250"
EOF

cat config/elasticsearch.yml
cluster.name: elasticsearch
node.name: elasticsearch
node.master: true
node.data: true

#network.host: 0.0.0.0
network.bind_host: 0.0.0.0
network.publish_host: 192.168.1.250
http.port: 9200
transport.tcp.port: 9300
http.cors.enabled: true
http.cors.allow-origin: "*"

discovery.type: single-node
bootstrap.memory_lock: true
action.destructive_requires_name: true
```

```
cd /data/apps/elasticsearch && docker-compose up -d 
```

## 三、集成MinIO

### 3.1 登陆 minio 创建AccessKey 和 bucket

- AccessKey & SecretKey: 
    - AccessKey: `DTNAzqBUZGAVisay`
    - SecretKey: `OyQzaSSXwXL3SohJ3hD7qgezOFqdywGg`
- Bucket: `dev-es-bucket`



### 3.1 在 es 节点安装 s3 插件并且重启 es 生效


```
# 安装插件，如果是集群需要在所有节点都安装
docker exec -it elasticsearch bash
elasticsearch-plugin install repository-s3 

# 配置AccessKey&SecretKey
/usr/share/elasticsearch/bin/elasticsearch-keystore add s3.client.default.access_key  【DTNAzqBUZGAVisay】
/usr/share/elasticsearch/bin/elasticsearch-keystore  add s3.client.default.secret_key 【OyQzaSSXwXL3SohJ3hD7qgezOFqdywGg】
```

```
# 重启 es
docker-compose down && docker-compose  up
curl 192.168.1.250:9200/_cat/plugins
elasticsearch repository-s3 7.7.0
```

## 四、备份 elasticsearch-A 实例的数据到 MinIO

### 4.1 注册repository快照仓库 `repository-minio`

```
curl  -XPUT http://192.168.1.111:9200/_snapshot/repository-minio?pretty \
-H 'Content-Type: application/json' \
-d @- <<'EOF'
{ "type":"s3", 
  "settings": {
     "bucket": "dev-es-bucket",
     "protocol": "http",
     "disable_chunked_encoding":"true",
     "endpoint":"192.168.1.250:9000"
   }
}
EOF
{
  "acknowledged" : true
}
```

### 4.2 创建快照 `snapshot-backup-202230311`

```
# 查看当前集群中文档总数
curl  -XGET http://192.168.1.111:9200/_cat/count?v
epoch      timestamp count
1678547297 15:08:17  4762
# 在repository-minio仓库仓库中创建完整快照
curl -uelastic:dengyou@163  -XPUT  http://192.168.1.111:9200/_snapshot/repository-minio/snapshot-backup-202230311?wait_for_completion=true 

# 查看repository中所有快照
curl -XGET  http://192.168.1.111:9200/_snapshot/repository-minio/_all?pretty
{
  "snapshots" : [
    {
      "snapshot" : "snapshot-backup-202230311",
      "uuid" : "FTTE0WTYTACn0o83eSZjwQ",
      "version_id" : 7070099,
      "version" : "7.7.0",
      "indices" : [
        ".kibana_1",
        ".apm-custom-link",
        ".security-7",
        "school1",
        "kibana_sample_data_ecommerce",
        ".kibana_task_manager_1",
        "school",
        ".apm-agent-configuration"
      ],
      "include_global_state" : true,
      "state" : "SUCCESS",
      "start_time" : "2023-03-11T15:10:41.636Z",
      "start_time_in_millis" : 1678547441636,
      "end_time" : "2023-03-11T15:10:42.236Z",
      "end_time_in_millis" : 1678547442236,
      "duration_in_millis" : 600,
      "failures" : [ ],
      "shards" : {
        "total" : 8,
        "failed" : 0,
        "successful" : 8
      }
    }
  ]
}
```

## 五、从 MinIO 中恢复数据到 elasticsearch-B 实例

### 5.1 关联 repository-minio 快照仓库到 elasticsearch-B 实例

```
# 关联本质是在elasticsearch-B实例中创建同名的repository
curl -XPUT http://192.168.1.250:9200/_snapshot/repository-minio -H 'Content-Type: application/json' -d @- <<'EOF'
{ "type":"s3",
  "settings": {
     "bucket": "dev-es-bucket",
     "protocol": "http",
     "disable_chunked_encoding":"true",
     "endpoint":"192.168.1.250:9000"
   }
}
EOF
{"acknowledged":true}
```

### 5.2 查看当前关联repository中所有快照

```
# 如果能查出来我们刚刚备份的快照 snapshot-backup-202230311 ，说明关联没问题了
curl -XGET  http://192.168.1.250:9200/_snapshot/repository-minio/_all?pretty
{
  "snapshots" : [
    {
      "snapshot" : "snapshot-backup-202230311",
      "uuid" : "FTTE0WTYTACn0o83eSZjwQ",
      "version_id" : 7070099,
      "version" : "7.7.0",
      "indices" : [
        ".kibana_1",
        ".apm-custom-link",
        ".security-7",
        "school1",
        "kibana_sample_data_ecommerce",
        ".kibana_task_manager_1",
        "school",
        ".apm-agent-configuration"
      ],
      "include_global_state" : true,
      "state" : "SUCCESS",
      "start_time" : "2023-03-11T15:10:41.636Z",
      "start_time_in_millis" : 1678547441636,
      "end_time" : "2023-03-11T15:10:42.236Z",
      "end_time_in_millis" : 1678547442236,
      "duration_in_millis" : 600,
      "failures" : [ ],
      "shards" : {
        "total" : 8,
        "failed" : 0,
        "successful" : 8
      }
    }
  ]
}
```

### 5.2 恢复数据

```
# 查看当前集群文档总数
 curl  -XGET http://192.168.1.250:9200/_cat/count?v
epoch      timestamp count
1678548444 15:27:24  0

# 恢复快照
curl -XPOST http://192.168.1.250:9200/_snapshot/repository-minio/snapshot-backup-202230311/_restore
{"accepted":true}


# 再次查看文档总数，与elasticsearch-A实例能对应上，说明没问题
curl  -XGET http://192.168.1.250:9200/_cat/count?v
epoch      timestamp count
1678548574 15:29:34  4762
```

