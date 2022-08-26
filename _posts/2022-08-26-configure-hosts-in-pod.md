---
layout: post
title: 使用hostAliases字段向Pod中添加hosts解析
date: 2020-08-26
tags: Kubernetes
---

默认情况下，pod的hosts文件中只包含IPV4和IPV6的基本配置，如果我们需要添加hosts解析，需要以来hostAliases字段实现

```
apiVersion: v1
kind: Pod
metadata: 
  name: hostaliases-pod
spec:
  restartPolicy: Never          
  hostAliases:
  - ip: "192.168.1.66"
    hostnames:
    - "harbor.linux.io"
    - "reg.linux.io"
  - ip: "192.168.1.250"
    hostnames:
    - "gitlab.liinux.io"
  containers:
  - name: busybox
    image: busybox:1.28
    command: ["sh", "-c", "sleep 3600"]

kubectl  apply -f host-aliases-pod.yaml
```

```
kubectl  exec -it hostaliases-pod -- cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
fe00::0 ip6-mcastprefix
fe00::1 ip6-allnodes
fe00::2 ip6-allrouters
172.20.17.202   hostaliases-pod

# Entries added by HostAliases.
192.168.1.66    harbor.linux.io reg.linux.io
192.168.1.250   gitlab.liinux.io
```