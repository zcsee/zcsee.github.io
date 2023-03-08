---
layout: post
title: 'CentOS7安装ansible'
date: 2023-03-08
categories: CentOS7
author: Jason
tags: centos7 ansible python3
---

## 方式1 添加epel仓库后yum安装(无需指定ansible版本时，推荐这个方式)

### Ansible仓库默认不在yum仓库中，因此我们需要先安装epel仓库

```shell
yum install epel-release -y
```
### 通过yum安装

```shell
yum install ansible -y
```

### 查看ansible版本

```
ansible --version
```

## 方式2 安装python3后，通过pip3安装

```shell
# 通过下载rpm离线包方式安装
yum install gcc gcc-c++ ncurses ncurses-devel unzip zlib-devel zlib openssl-devel openssl libffi-devel epel-release  sshpass  --downloadonly  --downloaddir=/tmp/soft
cd /tmp/soft
yum localinstall  *.rpm -y

# 或下载python3的编译包，编译安装
wget https://www.python.org/ftp/python/3.7.9/Python-3.7.9.tgz
tar -xvf  Python-3.7.9.tgz
cd Python-3.7.9
./configure --prefix=/usr/local/python/
make
make install

# 验证安装python3
/usr/local/python/bin/python3 --version
# python3加入环境变量
ln -s /usr/local/python/bin/python3 /usr/local/bin/
python3 --version

# --------------------------------------------------------------------------------
# 在线安装ansible方式
# 升级pip源
/usr/local/python/bin/pip3 install --upgrade pip
# 安装ansible，并指定版本
/usr/local/python/bin/pip3 install ansible==2.9.13
# 若安装出现网络问题，可以指定国内源进行安装
/usr/local/python/bin/pip3 install -i  https://mirrors.aliyun.com/pypi/simple  ansible
# =========
# 离线安装方式
/usr/local/python/bin/pip3 --default-timeout=1000  download ansible   -d  pythonDir/   
/usr/local/python/bin/pip3 install  ansible --no-index --find-links=pythonDir/
# ===============================================================================
# 简化版安装及配置
/usr/local/python/bin/pip3 install ansible
/usr/local/python/bin/ansible --version
ln -s /usr/local/python/bin/ansible /usr/local/bin/
ansible --version
```

