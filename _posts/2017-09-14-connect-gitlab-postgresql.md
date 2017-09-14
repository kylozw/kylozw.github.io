---
layout: post
title: 访问 GitLab Omnibus 自带的 PostgreSQL
subtitle: 'GitLab, PostgreSQL'
author: kylo
catalog: true
tags:
  - GitLab
  - PostgreSQL
---
我们有时候需要使用 GitLab 的数据库数据，但是 GitLab Omnibus 安装版默认使用包内自带的 PostgreSQL，不开启对外的 TCP 端口访问。

# PostgreSQL 配置

## IP 限制

PostgreSQL 数据库默认只监听来自 localhost 的 TCP 访问，由 `postgresql.conf` 文件中的配置项 `listen_addresses = 'localhost'` 控制，将该值设置为 `*`，允许所有 ip 的访问。

## 认证控制

除了 IP 限制，还有认证机制控制。在 `pg_hba.conf` 文件下，配置了不同 cidr-address 的认证方式。

TYPE|DATABASE|USER|CIDR-ADDRESS|METHOD
--|---|---|---|--
host|all|all|127.0.0.1/32|trust
host|all|all|0.0.0.0/0|md5

`trust` 表示 `127.0.0.1/32` 下的访问，不需要认证；`md5` 表示需要密码验证。

## 设置用户密码

开启对外的 TCP 端口访问，出于安全考虑必然要设置密码。使用数据库的 superuser 用户，进入 PostgreSQL 的交互模式，可通过 2 种方式修改密码：
- \password username
    ```bash
    gitlabhq_production=# \password gitlab
    Enter new password:
    Enter it again:
    ```
- alter user username with password 'xxx'

# GitLab 配置

以上是普通 PostgreSQL 的配置，GitLab 内置的 PostgreSQL 这 2 个文件的目录在 `/var/opt/gitlab/postgresql/data`，打开这 2 个文件，你会发现这样的注释：
```bash
# This file is managed by gitlab-ctl. Manual changes will be
# erased! To change the contents below, edit /etc/gitlab/gitlab.rb
# and run `sudo gitlab-ctl reconfigure`.
```
大致意思是你应该修改 `/etc/gitlab/gitlab.rb` 文件的配置，因为直接手动修改这 2 个文件的内容并不会生效，因为它会被 `gitlab.rb` 里的配置重新覆盖。

打开 `gitlab.rb`，你会发现如下配置：
```bash
# postgresql['listen_address'] = nil
# postgresql['md5_auth_cidr_addresses'] = []
# postgresql['trust_auth_cidr_addresses'] = []
```
将配置改为：
```bash
postgresql['listen_address'] = "*"
postgresql['md5_auth_cidr_addresses'] = ['0.0.0.0/0']
postgresql['trust_auth_cidr_addresses'] = ["127.0.0.1/32"]
```
这样，内置的 PostgreSQL 就会监听所有 IP 的请求。本地访问不需要认证，所以 GitLab 不需要配置连接密码，当然设置也是可以的；外网访问需要密码认证，按照上面步骤讲的设置好密码就可以用密码访问了。

PS：输入命令 `gitlab-psql -d gitlabhq_production` 进入GitLab 内置的 PostgreSQL 的交互模式。

最后，执行 `sudo gitlab-ctl reconfigure`，生效配置。