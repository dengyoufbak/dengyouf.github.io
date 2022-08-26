---
layout: post
title: Centos7升级ruby至v2.4
date: 2022-06-08
tags: Ruby
---

## 安装ruby

```
yum -y install ruby ruby-devel rubygems rpm-build
```

## 升级ruby至指定版本

```
yum install -y centos-release-scl-rh
yum install -y rh-ruby24 
scl enable rh-ruby24 bash
```

## 配置ruby国内源

```
gem sources  #列出默认源
gem sources --remove https://rubygems.org/  #移除默认源
gem sources -a https://mirrors.ustc.edu.cn/rubygems/  #添加科大源
```



