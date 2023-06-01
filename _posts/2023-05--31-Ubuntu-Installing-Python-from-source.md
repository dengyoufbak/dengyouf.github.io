---
layout: post
title: Ubuntu 编译安装 Python
date: 2023-05-31
tags: Python
---

## Ubuntu 编译安装 Python 3.6.8

- 安装依赖

```
sudo apt-get update
sudo apt-get install build-essential zlib1g-dev libffi-dev libssl-dev libbz2-dev libreadline-dev libsqlite3-dev liblzma-dev
```

- 下载程序包
```
wget https://www.python.org/ftp/python/3.6.8/Python-3.6.8.tgz
```

- 编译安装

```
tar -xf Python-3.6.8.tgz
./configure --enable-optimizations
make -j 8
sudo make altinstall
```

- 验证

```
python3.6 --version
```