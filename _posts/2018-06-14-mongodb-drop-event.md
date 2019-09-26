---
layout: post
title: MongoDB Drop 事件
subtitle: 'MongoDB, GitLab, Safety'
date: 2018-06-14T00:00:00.000Z
author: kylo
catalog: true
tags:
  - MongoDB
  - GitLab
  - Safety
---

# 起因

一天晚上突然发现生产系统使用的 MongoDB 库的一个 collection 被删掉了。问了团队成员，都未进行过删除操作，说明可能是三种情况：一、误删，连操作者自己也没意识到做了一个删除操作；二、bug，在某个程序中存在一个 bug；三、数据库帐号密码暴露了。

# MongoDB 系统日志

MongoDB 删除操作是有日志记录的，查看 MongoDB 系统日志（MongoDB 进程信息中可以看到配置文件的位置）：

```bash
[root@iZbp12sjfct392qm9maskkZ ~]# cat /data/mongodb/logs/mongod.log | grep drop
...
2018-06-12T20:44:33.307+0800 I COMMAND  [conn194051] CMD: drop report.####s
```

可以看到 conn194051 这个连接发起了删除操作，继续查看这个连接的记录：

```bash
2018-06-12T20:44:25.227+0800 I NETWORK  [conn194051] received client metadata from 115.195.50.134:53767 conn194051: { driver: { name: "nodejs", version: "3.0.2-2" }, os: { type: "Darwin", name: "darwin", architecture: "x64", version: "17.3.0" }, platform: "Node.js v8.2.1, LE, mongodb-core: 3.0.2", application: { name: "NoSQLBoosterV4" } }
2018-06-12T20:44:25.248+0800 I ACCESS   [conn194051] Successfully authenticated as principal front on report
2018-06-12T20:44:33.307+0800 I COMMAND  [conn194051] CMD: drop report.####
2018-06-12T20:50:17.725+0800 I -        [conn194051] end connection 115.195.50.134:53767 (92 connections now open)
```

发现了 IP 为 115.195.50.134 的某人，在 2018-06-12 20:44:25.227，通过 NoSQLBoosterV4 应用，使用 front 帐号登录了 report 数据库，并删除了 #### 这个 collection。这个 IP 并不是我们公司当前网络的 IP。

继续查看这个 IP 的所有日志，发现他分别用了 2 个方式连接了 MongoDB，NoSQLBoosterV4 和 MongoDB Shell。

```bash
2018-06-12T20:44:24.039+0800 I NETWORK  [conn194050] received client metadata from 115.195.50.134:53760 conn194050: { driver: { name: "nodejs", version: "3.0.2-2" }, os: { type: "Darwin", name: "darwin", architecture: "x64", version: "17.3.0" }, platform: "Node.js v8.2.1, LE, mongodb-core: 3.0.2", application: { name: "NoSQLBoosterV4" } }
2018-06-12T20:44:24.080+0800 I ACCESS   [conn194050] Successfully authenticated as principal front on report
2018-06-12T20:44:24.249+0800 I ACCESS   [conn194050] Unauthorized: not authorized on admin to execute command { listCollections: true, filter: {}, cursor: {} }
2018-06-12T20:44:24.253+0800 I ACCESS   [conn194050] Unauthorized: not authorized on admin to execute command { listCollections: true, filter: {}, cursor: {} }
2018-06-12T20:44:24.267+0800 I ACCESS   [conn194050] Unauthorized: not authorized on admin to execute command { serverStatus: 1.0, asserts: 0.0, backgroundFlushing: 0.0, connections: 0.0, cursors: 0.0, dur: 0.0, extra_info: 0.0, globalLock: 0.0, locks: 0.0, network: 0.0, opcounters: 0.0, opcountersRepl: 0.0, writeBacksQueued: 0.0, mem: 0.0, metrics: 0.0, advisoryHostFQDNs: 0.0 }
2018-06-12T20:44:24.268+0800 I ACCESS   [conn194050] Unauthorized: not authorized on admin to execute command { listDatabases: 1.0 }
2018-06-12T20:44:24.756+0800 I ACCESS   [conn194050] Unauthorized: not authorized on admin to execute command { listCollections: true, filter: {}, cursor: {} }
2018-06-12T20:44:24.756+0800 I ACCESS   [conn194050] Unauthorized: not authorized on admin to execute command { listCollections: true, filter: {}, cursor: {} }
2018-06-12T20:44:24.818+0800 I ACCESS   [conn194050] Unauthorized: not authorized on admin to execute command { serverStatus: 1.0, asserts: 0.0, backgroundFlushing: 0.0, connections: 0.0, cursors: 0.0, dur: 0.0, extra_info: 0.0, globalLock: 0.0, locks: 0.0, network: 0.0, opcounters: 0.0, opcountersRepl: 0.0, writeBacksQueued: 0.0, mem: 0.0, metrics: 0.0, advisoryHostFQDNs: 0.0 }
2018-06-12T20:44:24.818+0800 I ACCESS   [conn194050] Unauthorized: not authorized on admin to execute command { listDatabases: 1.0 }
2018-06-12T20:44:24.926+0800 I ACCESS   [conn194050] Unauthorized: not authorized on report to execute command { usersInfo: 1.0 }
2018-06-12T20:44:25.005+0800 I ACCESS   [conn194050] Unauthorized: not authorized on report to execute command { aggregate: "errorbeers", pipeline: [ { $indexStats: {} }, { $skip: 0.0 } ], cursor: { batchSize: 1000.0 } }
2018-06-12T20:44:25.027+0800 I ACCESS   [conn194050] Unauthorized: not authorized on report to execute command { aggregate: "masters", pipeline: [ { $indexStats: {} }, { $skip: 0.0 } ], cursor: { batchSize: 1000.0 } }
2018-06-12T20:44:25.027+0800 I ACCESS   [conn194050] Unauthorized: not authorized on report to execute command { aggregate: "metas", pipeline: [ { $indexStats: {} }, { $skip: 0.0 } ], cursor: { batchSize: 1000.0 } }
2018-06-12T20:44:25.027+0800 I ACCESS   [conn194050] Unauthorized: not authorized on report to execute command { aggregate: "recorddaycounts", pipeline: [ { $indexStats: {} }, { $skip: 0.0 } ], cursor: { batchSize: 1000.0 } }
2018-06-12T20:44:25.028+0800 I ACCESS   [conn194050] Unauthorized: not authorized on report to execute command { aggregate: "records", pipeline: [ { $indexStats: {} }, { $skip: 0.0 } ], cursor: { batchSize: 1000.0 } }
2018-06-12T20:44:25.028+0800 I ACCESS   [conn194050] Unauthorized: not authorized on report to execute command { aggregate: "####", pipeline: [ { $indexStats: {} }, { $skip: 0.0 } ], cursor: { batchSize: 1000.0 } }
2018-06-12T20:44:25.036+0800 I ACCESS   [conn194050] Unauthorized: not authorized on report to execute command { aggregate: "users", pipeline: [ { $indexStats: {} }, { $skip: 0.0 } ], cursor: { batchSize: 1000.0 } }
2018-06-12T20:44:25.036+0800 I ACCESS   [conn194050] Unauthorized: not authorized on report to execute command { aggregate: "####", pipeline: [ { $indexStats: {} }, { $skip: 0.0 } ], cursor: { batchSize: 1000.0 } }
2018-06-12T20:50:17.727+0800 I -        [conn194050] end connection 115.195.50.134:53760 (90 connections now open)
```

这是通过 NoSQLBoosterV4 连接的另一个连接的日志，并没有做特殊操作，这些 command 都是 NoSQLBoosterV4 应用程序里发起的统计请求。

```bash
2018-06-12T22:01:27.393+0800 I NETWORK  [conn194081] received client metadata from 115.195.50.134:51860 conn194081: { application: { name: "MongoDB Shell" }, driver: { name: "MongoDB Internal Client", version: "3.6.4" }, os: { type: "Darwin", name: "Mac OS X", architecture: "x86_64", version: "17.3.0" } }
2018-06-12T22:01:27.512+0800 I ACCESS   [conn194081] Successfully authenticated as principal front on report
2018-06-12T22:01:27.518+0800 I ACCESS   [conn194081] Unauthorized: not authorized on admin to execute command { getLog: "startupWarnings" }
2018-06-12T22:01:27.546+0800 I ACCESS   [conn194081] Unauthorized: not authorized on admin to execute command { replSetGetStatus: 1.0, forShell: 1.0 }
2018-06-12T22:01:39.644+0800 I ACCESS   [conn194081] Unauthorized: not authorized on admin to execute command { getLog: "global" }
2018-06-12T22:03:13.027+0800 I ACCESS   [conn194081] Unauthorized: not authorized on admin to execute command { getLog: "global" }
2018-06-12T22:05:34.881+0800 I ACCESS   [conn194081] Unauthorized: not authorized on admin to execute command { listDatabases: 1.0 }
2018-06-12T22:10:16.194+0800 I -        [conn194081] end connection 115.195.50.134:51860 (91 connections now open)
```

这是通过 MongoDB Shell 连接的一个连接的日志，可以看到他企图通过 getLog 命令获取我们的日志信息，但是 front 帐号并没有这个权限。

此时，基本上可以肯定，这是有意为之的一次事件。

# GitLab 访问日志

经过上面对 MongoDB 系统日志的排查，我们怀疑我们的源码泄漏了。

我们的源码是在自建 GitLab 上管理的，所以我们希望能在 GitLab 访问日志上找到同一个 IP 的访问，但是并没有找到。

我们的 GitLab 是可以通过域名访问，所以 GitLab 访问日志上都是 Nginx 主机的 IP。但是，我们可以去 Nginx 访问日志上查看！

```bash
115.195.50.134 - - [12/Jun/2018:22:11:12 +0800] "GET / HTTP/1.1" 200 6046 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:25 +0800] "GET /####/#### HTTP/1.1" 200 10151 "http://####.####.###/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:27 +0800] "GET /####/####/autocomplete_sources?type_id=#### HTTP/1.1" 200 99287 "http://####.####.###/####/####" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:29 +0800] "GET /####/gpm HTTP/1.1" 200 9985 "http://####.####.###/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:29 +0800] "GET /####/gpm/uploads/79d025b0272ba2512bfea8cdc9cf1bad/5936ab56511fb.png HTTP/1.1" 200 153203 "http://####.####.###/####/gpm" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:30 +0800] "GET /####/gpm/autocomplete_sources?type_id=gpm HTTP/1.1" 200 97595 "http://####.####.###/####/gpm" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:35 +0800] "GET /####/gpm/tree/master HTTP/1.1" 200 9924 "http://####.####.###/####/gpm" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:35 +0800] "GET /####/gpm/refs/master/logs_tree/?_=1528812672783 HTTP/1.1" 200 2524 "http://####.####.###/####/gpm/tree/master" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:36 +0800] "GET /####/gpm/autocomplete_sources?type_id=master HTTP/1.1" 200 97595 "http://####.####.###/####/gpm/tree/master" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:38 +0800] "GET /####/gpm/tree/master/app HTTP/1.1" 200 8195 "http://####.####.###/####/gpm/tree/master" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:38 +0800] "GET /####/gpm/refs/master/logs_tree/app?_=1528812672784 HTTP/1.1" 200 1542 "http://####.####.###/####/gpm/tree/master/app" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:40 +0800] "GET /####/gpm/autocomplete_sources?type_id=master%2Fapp HTTP/1.1" 200 97595 "http://####.####.###/####/gpm/tree/master/app" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:41 +0800] "GET /####/gpm/tree/master/bin HTTP/1.1" 200 7681 "http://####.####.###/####/gpm/tree/master" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:41 +0800] "GET /####/gpm/refs/master/logs_tree/bin?_=1528812672785 HTTP/1.1" 200 506 "http://####.####.###/####/gpm/tree/master/bin" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:43 +0800] "GET /####/gpm/blob/master/bin/backup.sh HTTP/1.1" 200 8228 "http://####.####.###/####/gpm/tree/master/bin" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
115.195.50.134 - - [12/Jun/2018:22:12:43 +0800] "GET /####/gpm/autocomplete_sources?type_id=master%2Fbin HTTP/1.1" 200 97603 "http://####.####.###/####/gpm/tree/master/bin" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36"
...
```

我们找到了他，他主要查看了以下项目：

```bash
/####/####
/####/gpm
/####/trace
/####/facade
/####/vdetect
/####/door
/####/rn-alipay
/####/react-native-previewimage
/####/rn-components
/####/team
/####/annual-meeting
/####/market-crawler
/####/miaaa
```

针对这个访问时间，我们继续查看 GitLab 访问日志：

```bash
Started GET "/" for ###.###.###.### at 2018-06-12 22:11:11 +0800
Processing by RootController#index as HTML
Read fragment views/groups/88-20171106011846019685000/projects/207-20180612135607942575000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Write fragment views/groups/88-20171106011846019685000/projects/207-20180612135607942575000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.6ms)
Read fragment views/groups/88-20171106011846019685000/projects/205-20180612130329312836000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.4ms)
Write fragment views/groups/88-20171106011846019685000/projects/205-20180612130329312836000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.4ms)
Read fragment views/groups/88-20171106011846019685000/projects/201-20180612112749249496000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.4ms)
Write fragment views/groups/88-20171106011846019685000/projects/201-20180612112749249496000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.5ms)
Read fragment views/groups/88-20171106011846019685000/projects/215-20180612112611014670000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.4ms)
Write fragment views/groups/88-20171106011846019685000/projects/215-20180612112611014670000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Read fragment views/groups/90-20170915100225974079000/projects/94-20180612103356453310000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Write fragment views/groups/90-20170915100225974079000/projects/94-20180612103356453310000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.5ms)
Read fragment views/groups/90-20170915100225974079000/projects/183-20180612100013077984000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Write fragment views/groups/90-20170915100225974079000/projects/183-20180612100013077984000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.5ms)
Read fragment views/groups/90-20170915100225974079000/projects/64-20180612095532863552000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Write fragment views/groups/90-20170915100225974079000/projects/64-20180612095532863552000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Read fragment views/groups/88-20171106011846019685000/projects/190-20180612091642136317000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Write fragment views/groups/88-20171106011846019685000/projects/190-20180612091642136317000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.4ms)
Read fragment views/groups/88-20171106011846019685000/projects/83-20180612084341867771000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.4ms)
Read fragment views/groups/87-20170920121332843406000/projects/160-20180612080435357057000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Read fragment views/groups/88-20171106011846019685000/projects/167-20180612022648572841000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.2ms)
Read fragment views/groups/88-20171106011846019685000/projects/168-20180608022600799766000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Read fragment views/groups/90-20170915100225974079000/projects/153-20180607035336637629000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Read fragment views/groups/88-20171106011846019685000/projects/170-20180606112209498284000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Read fragment views/groups/87-20170920121332843406000/projects/139-20180606102613104028000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Read fragment views/groups/88-20171106011846019685000/projects/135-20180606073721777839000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Read fragment views/groups/88-20171106011846019685000/projects/145-20180606072310755774000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.7ms)
Read fragment views/groups/87-20170920121332843406000/projects/105-20180605072113262048000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Read fragment views/groups/88-20171106011846019685000/projects/156-20180604123306261488000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Read fragment views/groups/87-20170920121332843406000/projects/188-20180601105239819353000/root/index/application_settings/5-20180530030607058181000/v2.3/81898b0083c38e04818f4d0c8130b34f (0.3ms)
Completed 200 OK in 352ms (Views: 254.6ms | ActiveRecord: 57.8ms)
Started GET "/####/####" for ###.###.###.### at 2018-06-12 22:12:25 +0800
Processing by ProjectsController#show as HTML
  Parameters: {"namespace_id"=>"####", "id"=>"####"}
Read fragment views/####/####-c415b756e350e973a1aa241092818f1dd458df6f-readme/6e1c334cb4104b0dd5ff60f7784a9511 (0.3ms)
Completed 200 OK in 212ms (Views: 127.8ms | ActiveRecord: 23.1ms)
```

我们在 GitLab 上找到了对应的日志，但是很遗憾，这个用户并没有进行登录操作，所以我们无法确定他到底是使用哪个帐号登录查看了 GitLab。这个账号一定是之前就在登录界面勾选了 Remember me！

# 其他异常

我们还发现了其他异常，虽然可能和本次事件无关，但是这里还是做下记录。

## GitLab 登录信息

GitLab 的数据库用户表里记录了每个帐户的当前登录时间和上次登录时间，我们最后对当前登录时间进行了倒序排查，发现了一个不应该登录的帐户在 2018-06-13 进行了登录：

![gitlab_登录记录.png | center | 747x341](https://cdn.nlark.com/yuque/0/2019/png/87959/1569498045168-1b8931c8-0e30-4613-b46f-517cba91fb3b.png)

## MongoDB 服务器 SSH 登录信息

在 2018-06-14 登录 MongoDB 所在服务器时，发现有一个 202.197.46.110 的 IP 试图登录服务器，但是失败了，查看 SSH 登录日志：

```bash
Jun 12 15:37:22 iZbp12sjfct392qm9maskkZ sshd[13413]: Did not receive identification string from 202.197.46.110
Jun 13 09:12:32 iZbp12sjfct392qm9maskkZ sshd[13982]: Invalid user omoto from 202.197.46.110
Jun 13 09:12:32 iZbp12sjfct392qm9maskkZ sshd[13982]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=202.197.46.110
Jun 13 09:12:34 iZbp12sjfct392qm9maskkZ sshd[13982]: Failed password for invalid user omoto from 202.197.46.110 port 35135 ssh2
Jun 13 09:12:34 iZbp12sjfct392qm9maskkZ sshd[13982]: Received disconnect from 202.197.46.110: 11: Bye Bye [preauth]
Jun 13 09:45:21 iZbp12sjfct392qm9maskkZ sshd[13985]: Invalid user omoto from 202.197.46.110
Jun 13 09:45:21 iZbp12sjfct392qm9maskkZ sshd[13985]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=202.197.46.110
Jun 13 09:45:23 iZbp12sjfct392qm9maskkZ sshd[13985]: Failed password for invalid user omoto from 202.197.46.110 port 51678 ssh2
Jun 13 09:45:23 iZbp12sjfct392qm9maskkZ sshd[13985]: Received disconnect from 202.197.46.110: 11: Bye Bye [preauth]
Jun 13 10:17:51 iZbp12sjfct392qm9maskkZ sshd[14086]: Invalid user tanaka from 202.197.46.110
Jun 13 10:17:51 iZbp12sjfct392qm9maskkZ sshd[14086]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=202.197.46.110
Jun 13 10:17:53 iZbp12sjfct392qm9maskkZ sshd[14086]: Failed password for invalid user tanaka from 202.197.46.110 port 46572 ssh2
Jun 13 10:17:53 iZbp12sjfct392qm9maskkZ sshd[14086]: Received disconnect from 202.197.46.110: 11: Bye Bye [preauth]
Jun 13 10:50:19 iZbp12sjfct392qm9maskkZ sshd[14214]: Invalid user hashimoto from 202.197.46.110
Jun 13 10:50:19 iZbp12sjfct392qm9maskkZ sshd[14214]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=202.197.46.110
Jun 13 10:50:20 iZbp12sjfct392qm9maskkZ sshd[14214]: Failed password for invalid user hashimoto from 202.197.46.110 port 50894 ssh2
Jun 13 10:50:20 iZbp12sjfct392qm9maskkZ sshd[14214]: Received disconnect from 202.197.46.110: 11: Bye Bye [preauth]
Jun 13 11:23:16 iZbp12sjfct392qm9maskkZ sshd[14255]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=202.197.46.110  user=root
Jun 13 11:23:18 iZbp12sjfct392qm9maskkZ sshd[14255]: Failed password for root from 202.197.46.110 port 43209 ssh2
Jun 13 11:23:18 iZbp12sjfct392qm9maskkZ sshd[14255]: Received disconnect from 202.197.46.110: 11: Bye Bye [preauth]
Jun 13 11:56:01 iZbp12sjfct392qm9maskkZ sshd[14279]: Invalid user ishikawa from 202.197.46.110
Jun 13 11:56:01 iZbp12sjfct392qm9maskkZ sshd[14279]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=202.197.46.110
Jun 13 11:56:03 iZbp12sjfct392qm9maskkZ sshd[14279]: Failed password for invalid user ishikawa from 202.197.46.110 port 48961 ssh2
Jun 13 11:56:03 iZbp12sjfct392qm9maskkZ sshd[14279]: Received disconnect from 202.197.46.110: 11: Bye Bye [preauth]
...
```

这个 IP 一直在使用各种用户尝试登录我们的服务器。

我们查找了这个 IP 的地理位置：

![mongo 服务器登陆失败 ip.png | center | 747x352](https://cdn.yuque.com/yuque/86/2018/png/87960/1528958805587-e2caaeea-b470-408a-bdbd-150c4e8687ba.png "")

# 总结

这次事件暴露了我们的好几处系统安全漏洞：

* MongoDB 访问权限
    * 由于我们的正式和测试数据库是同一个示例，所以没有设置 bind ip
    * 太多应用使用了同一个数据库用户
* GitLab 源码管理
    * 我们的仓库基本上都是设置 Internal 可见级别，就是已登录帐号可查看
    * 不在使用的帐号没有及时清理
    * 源码里保存了正式数据库的帐号密码

针对 MongoDB，我们将 MongoDB 的正式和测试数据库分离，并设置了正式数据库示例的 bind ip，并且每个应用配置单独的数据库用户；针对 GitLab，我们对帐号进行了清理。
