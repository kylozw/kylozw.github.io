---
layout: post
title: Gitlab 基本使用
subtitle: 'Gitlab'
author: kylo
catalog: true
tags:
  - Gitlab
---
# 注册用户

进入 <http://gitlab.yourdomain.com/>，填写相关信息，注册用户。Username 是登录用户名，不可更改。由于[腾讯邮箱把 ip 封了](http://service.mail.qq.com/cgi-bin/help?subtype=1&&id=20022&&no=1000725)，可以先使用 163 邮箱注册，确认验证通过后，在 Profile Settings 把邮箱设置为公司邮箱；另 gmail 由于被墙，无法发送。 ![](http://7xsf19.com1.z0.glb.clouddn.com/Sign%20in%20%C2%B7%20GitLab.png) ![](http://7xsf19.com1.z0.glb.clouddn.com/Profile%20Settings.png)

# 申请加入组

在侧边栏菜单选择 Groups，并切换到 Explore Groups，可以看到分组，点击分组名，进入分组详情，如要进入改组开发，点击 Request Access 申请。 由分组 Master 以上权限的角色通过；当然，也可以由 Master 直接添加组员。 ![](http://7xsf19.com1.z0.glb.clouddn.com/Groups%20%C2%B7%20Explore.png) ![](http://7xsf19.com1.z0.glb.clouddn.com/Request%20Access.png)

# 添加 SSH key

在 Profile Settings 切换到 SSH Keys 添加本地 SSH 公钥。 ![](http://7xsf19.com1.z0.glb.clouddn.com/SSH%20Keys.png)

# 修改 remote 源

在本地项目目录下，运行一下命令：

```shell
git remote set-url origin http://121.40.208.242/base-tech/moondragon.git
```

地址由项目详情页上复制，选择 http 的 pull/push 都需要用户名和密码。当然，你也可以在 Git 客户端，如 SmartGit 进行修改。
