---
title: 修改Linux主机root登录用户名以减少暴力破解
description: ""
date: 2023-02-24 14:10:00 +0800
categories: [Linux]
tags: []
---

正常来说，Linux默认的最高管理员用户名都是root，服务器容易被暴力破解用户名，如果修改了root的用户名，则可以减少被暴力破解的几率，毕竟得先知道用户名才能登录（Linux尝试登录不存在的账户只会报密码错误）。

1. 首先用root用户登录，然后 `vim /etc/passwd` 编辑文件，修改第1行第1个root为新的用户名，`Esc` 后输入 `:x` 退出。
2. 接着 `vim /etc/shadow`，修改第1行第1个root为新的用户名，`Esc` 后输入 `:x!` 强制保存并退出。
3. 最后需要修改sudo的设置，`vim /etc/sudoers`，找到 `root  ALL=(ALL)  ALL`，在下面添加一行 `新用户名 ALL=(ALL) ALL`，`Esc` 后 `:x` 保存退出。

然后需要重启下系统，输入 `reboot`。
