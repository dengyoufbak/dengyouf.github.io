---
layout: post
title: Ubuntu 20.04系统初始化
date: 2022-08-07
tags: Ubuntu
---

## 概述

Ubuntu20.04是Ubuntu的第 8 个 LTS 版本代号为"Focal Fossa"

## 常规初始化配置

#### 网络配置

```
# 1.设置静态网络
cat /etc/netplan/00-installer-config.yaml
network:
  ethernets:
    eth0:
      dhcp4: no
      addresses:
      - 192.168.1.122/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [223.5.5.5, 114.114.114.114]
  version: 2

# 2.应用网络配置
netplan apply
```

#### 初始化SSH配置文件

```
#1.允许Root登陆以及采用密码认证（prohibit-password:禁用密码）
sed -i "s|#PermitRootLogin prohibit-password|PermitRootLogin no#g" /etc/ssh/sshd_config  # 为了安全
sed -i "s|#PasswordAuthentication|PasswordAuthentication#g" /etc/ssh/sshd_config

#2.重启ssh服务
systemctl restart sshd
```

#### 关闭防火墙

```
# 1.放行端口
sudo ufw allow 3306/tcp

# 2.关闭防火墙
sudo ufw disable
```

#### 设置时区

```
# 1.查看当前时区
date -R

# 2.修改为北京时区
sudo cp /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime

# 3. 使用交互方式配置为Beijing时区
# 交互式地区选择亚洲 Asia，确认之后选择中国（China)，最后选择北京(Beijing)
sudo tzselect
```

#### 配置软件源

修改/etc/apt/sources.list替换默认的http://archive.ubuntu.com/为mirrors.aliyun.com，替换结果如下：

```
sudo sed -i 's@archive.ubuntu.com@mirrors.aliyun.com@g' /etc/apt/sources.list
```
```
deb https://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse

deb https://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse

deb https://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse

# deb https://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
# deb-src https://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse

deb https://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
```

```
sudo apt autoclean
sudo apt update
sudo apt -y install nano vim net-tools tree wget dos2unix unzip htop ncdu bash-completion
```
