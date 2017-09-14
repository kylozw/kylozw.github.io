---
layout: post
title: 在 Debian 平台或者 RedHat 平台安装 Java
subtitle: 'Java, Debian, RedHat'
date: '2016-08-23 15:02'
author: kylo
catalog: true
tags:
  - Java
  - Debian
  - RedHat
---

# Debian 平台

Debian 平台的安装非常简单，只需要添加 repository 源，更新源信息，然后安装即可：

```shell
$ sudo add-apt-repository ppa:webupd8team/java
$ sudo apt-get update
$ sudo apt-get install oracle-java8-installer
```

# Redhat 平台

Redhat 平台的安装需要到 [Oracle Java 下载页面](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)获取相应版本的 rpm 文件链接地址，使用 wget 命令下载（当然也可以直接下载），然后进行本地 yum 安装，以最新的 Java 8 为例：

```shell
$ wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u102-b14/jdk-8u102-linux-x64.rpm"
$ sudo yum localinstall jdk-8u102-linux-x64.rpm
```
