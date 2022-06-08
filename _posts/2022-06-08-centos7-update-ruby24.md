---
layout: post
title: Centos7升级ruby至v2.4
date: 2022-06-08
tags: JEKYLL
---

## 安装ruby

```
yum -y install ruby ruby-devel rubygems rpm-build
```

## 升级ruby

```
yum install -y centos-release-scl-rh
yum install -y rh-ruby24
scl enable rh-ruby24 bash
```


