---
layout: post
title: Vagrant 制作自己的 Box 虚拟机镜像
date: 2022-05-22
tags: VAGRANT
---

## 系统环境介绍

- Iso镜像: CentOS-7-x86_64-Minimal-2009.iso
- VirtualBox 版本: 6.1.34
- Vagrant 版本: 2.2.19

## VirtualBox 新建虚拟机

- 内存：2C/4096G
- 硬盘：
  - 系统盘：50G
  - 数据盘：100G
- 声音、USB： 禁用
- 网络： 网络地址转换（NAT)
- 端口转发 -> ssh 宿主机：127.0.0.1 2222 => 虚拟机：（空） 22

![](/images/pic/vagrant1.png)



## 配置账户

- 设置 sshd 远程登录

```
{
sed -i 's/#UseDNS no/UseDNS no/g' /etc/ssh/sshd_config
sed -i 's/#GSSAPIAuthentication no/GSSAPIAuthentication no/g' /etc/ssh/sshd_config
sed -i 's@PasswordAuthentication no@PasswordAuthentication yes@g' /etc/ssh/sshd_config
sed -i 's@#PermitRootLogin yes@PermitRootLogin yes@g' /etc/ssh/sshd_config
systemctl reload sshd
}
```

- vagrant sudo 免密登陆

```
{
useradd vagrant
echo "vagrant"|passwd --stdin vagrant
echo "vagrant  ALL=(ALL)       NOPASSWD: ALL" >> /etc/sudoers.d/vagrant
}
```

- 安装 Vagrant 密钥

```
{
su - vagrant
mkdir .ssh && chmod 0700 .ssh
wget --no-check-certificate \
    https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub \
    -O /home/vagrant/.ssh/authorized_keys

}
chmod 0600 .ssh/authorized_keys
```

## 配置Yum源

```
{
mkdir /etc/yum.repos.d/bak
mv /etc/yum.repos.d/*  /etc/yum.repos.d/bak
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
yum install epel-release
}
{
yum clean all
yum makecache
}
```

## 安装 VitualBox additions

- 安装编译工具

```
{
yum -y groupinstall Development tools
}
```

- 安装软件包

```
{
yum -y install xorg-x11-drivers xorg-x11-utils
yum install -y perl gcc dkms kernel-devel kernel-headers make bzip2
yum install -y vim wget lrzsz net-tools
reboot
}
```

- 加载 Guest Additions CD Image 镜像

![](/images/pic/vagrant2.png)

```
{
mkdir -p /tmp/cdrom && mount /dev/cdrom /tmp/cdrom
sh /tmp/cdrom/VBoxLinuxAdditions.run
}
```

## 清理缓存数据

```
yum  clean all
init 0
```

## 虚拟机 打包为 vagrant box

```
mkdir centos7-2009 && cd centos7-2009 
# centos7 为虚拟机的名字
vagrant package --output centos7-9.box --base centos7
==> centos7: Clearing any previously set forwarded ports...
==> centos7: Exporting VM...
==> centos7: Compressing package to: /Users/dengyou/Vagrant_Pro/centos7-2009/centos7-9.box
```

## 测试启动虚拟机啊

```
vagrant box add dengyouf/CentOS-7.9 centos7-9.box
```

```
# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure("2") do |config|
   (1..2).each do |i|
        config.vm.define "node#{i}" do |node|
            # 设置虚拟机的Box
            node.vm.box = "dengyouf/CentOS-7.9"

            # 设置虚拟机的主机名
            node.vm.hostname="centos-node#{i}"

            # 设置虚拟机的IP
            node.vm.network "private_network", ip: "192.168.56.#{100+i}"

            # 设置主机与虚拟机的共享目录
            node.vm.synced_folder ".", "/home/vagrant/share"

            # VirtaulBox相关配置
            node.vm.provider "virtualbox" do |v|
                # 设置虚拟机的名称
                v.name = "centos-node#{i}"
                # 设置虚拟机的内存大小
                v.memory = 4096
                # 设置虚拟机的CPU个数
                v.cpus = 2
            end
        end
        config.vm.provision "shell", inline: <<-SCRIPT
          {
          sudo mkdir -p /data
          sudo mkfs.ext4 -F /dev/sdb
          UUID=$(sudo blkid -s UUID /dev/sdb|awk -F":" '{print $2}'| tr -d '"')
          TYPE=$(sudo blkid  -s TYPE /dev/sdb|awk  -F"=" '{print $NF}'|tr -d '"')
          echo -e "$UUID\t/data\t$TYPE\tdefaults\t0 0"|sudo tee -a /etc/fstab
          sudo mount -a
          }
        SCRIPT
        # 执行初始化脚本
        #config.vm.provision "shell", path: "bootstrap.sh"
   end
end
```

```
vagrant status                                               
Current machine states:

node1                     running (virtualbox)
node2                     running (virtualbox)
```

```
vagrant ssh-config
Host node1
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /Users/dengyou/Vagrant_Pro/centos7-2009/.vagrant/machines/node1/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL

Host node2
  HostName 127.0.0.1
  User vagrant
  Port 2200
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /Users/dengyou/Vagrant_Pro/centos7-2009/.vagrant/machines/node2/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
```

