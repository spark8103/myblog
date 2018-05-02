---
layout: post
title: Github Clone
date: 2018-03-09 16:22:12.000000000 +09:00
tags: 技术
---

# Github在centos6下无法克隆问题

## 背景
github的证书移除了TLSv1/TLSv1.1的支持，导致centos6无法克隆github的项目。[Weak cryptographic standards removed](https://blog.github.com/2018-02-23-weak-cryptographic-standards-removed/)

解决这个问题的方法是首先升级git版本(1.8之前的版本都要升级)，第二配置git的认证方式。

## 升级centos6的git版本。 
通过ius安装，如果是centos7就修改下6为7即可。
```shell
yum remove git
yum install epel-release
yum install https://centos6.iuscommunity.org/ius-release.rpm
yum install git2u
```

## 修改git的认证
```
git config --global http.sslversion tlsv1
```
