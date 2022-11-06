---
title: 记一次 Matrix Dendrite 的部署（SQLite 版）
tags:
  - Web3
  - Matrix
  - Dendrite
categories:
  - web3
  - service
  - deploying
date: 2022-11-02 11:12:51
---


[Dendrite](https://matrix-org.github.io/dendrite/) 是用 Go 写成的第二代 [Matrix homeserver](https://matrix.org/docs/guides/introduction#how-does-it-work)，今天就来踩个坑。

<!-- more -->

整个部署流程基于官方的 [Get started](https://github.com/matrix-org/dendrite#get-started)。

## 编译与生成密钥

编译和生成密钥的时候没有遇到啥问题。

我使用的环境是 AWS 的 1g1c 小鸡，系统是 Ubuntu 22.04-amd64，Go 版本是 1.18.1 linux/amd64。

## 配置文件

### 配置数据库

在编译和生成密钥完成后，运行 `dendrite-monolith-server` 会出现以下报错：

> PANI[2022-11-02T05:02:46.231645575Z] Failed to set up global database connections  error="failed to find maximum connections: dial tcp: lookup hostname: no such host"  
> panic: (*logrus.Entry) 0xc0002a5c70  
> ...

由于这个报错很模糊，也可能是因为我是新手，所以我并没有从网上找到有用的东西。阅读了它的源代码之后发现是在执行 config 的时候 panic 的，那就说明官方给的 `dendrite-sample.monolith.yaml` 是不能直接用的。

打开 YAML 一看，发现它直接在 `global.database` 里用了 PostgreSQL，但是我并不打算用 PostgreSQL 部署。研究了一下注释之后发现 `global.database` 是 PostgreSQL 专属的， SQLite 不能用 `global.database`（也就是 Dendrite 的 Global database connection pool），且只能手动给每个 component 指定 database，也就是在每个 component 下面加：

```yaml
database:
  connection_string: file:dendrite_<component>.db
  max_open_conns: 10
  max_idle_conns: 2
  conn_max_lifetime: -1
```

需要注意的是，每个 component 不能和其它 component 用同一个 .db，同时 `global.database` 也要注释掉。

### 添加缺失的 compenet 配置

再次运行 `dendrite-monolith-server` 又会得到下面的报错：

> INFO[2022-11-02T05:25:53.722317580Z] Dendrite version 0.10.5+8c7b274e  
> PANI[2022-11-02T05:25:53.756013624Z] failed to connect to room server db           error="sqlutil.Open: no database connections configured"  
> ...

然而在我的 YAML 里面并没有关于 room server 的配置，很显然 example 是漏了一些字段。于是我去网上找了一份也是用 SQLite 的 Dendrite [配置文件](https://codeberg.org/gerald/dendrite-on-flyio/src/branch/main/dendrite-example.yaml)，对比之后发现 example 少了 `room_server` 和 `key_server` 两个 component 的配置，还有 `user_api.account_database` 和 `user_api.device_database`。

解决方法是按照网上的配置文件补上即可，注意修改 `database.connection_string` 的路径。

再次运行 `dendrite-monolith-server`，server 已经顺利跑起来了。地址是 <http://localhost:8008>。

## 创建账号

按照 [Get started](https://github.com/matrix-org/dendrite#get-started) 的最后一步创建账号会出现以下错误：

>FATA[0000] Shared secret registration is not enabled, enable it by setting a shared secret in the config: 'client_api.registration_shared_secret'

这个报错就说得很清楚了，是因为没有在 YAML 里设置 `client_api.registration_shared_secret`。解决办法是写一段随机字符串作为 shared secret。

### 公开账号注册

如果你希望能在客户端注册账号而不是在服务器上，你需要公开账号注册。

公开账号注册有两种方法：

1. *（不安全）* 将 YAML 里的 `client_api.registration_disabled` 设为 `false`，然后在 `dendrite-monolith-server` 的启动参数添加 `-really-enable-open-registration`。
2. [Enable reCAPTCHA verification registration](https://matrix-org.github.io/dendrite/administration/registration)。

## 部署

在部署之前记得把 `global.server_name`、`global.well_known_server_name` 和 `global.well_known_client_name` 改成自己 server 的 domain。

使用 `-http(s)-bind-address` 可以更改 HTTP(S) 的端口，HTTP 端口默认是 8008，HTTPS 端口默认是 8448。

## Integration Manager

目前 Matrix 主流的 integration manager 是 [Dimension](https://github.com/turt2live/matrix-dimension)。

如果要部署 integration manager，需要用 Nginx 反向代理把 Dimension 的端口映射到 443 （或 80），具体方法可以看 [Installing Dimension](https://github.com/turt2live/matrix-dimension/blob/master/docs/installing.md)。

## 使用

测试使用的客户端分别是 Android 和 Desktop 版的 [Element](https://element.io/)。

测试发现 Android 端的电话图标和视频图标发起的分别是电话会议和视频会议，而 Desktop 端的是语言和视频通话。其中 Desktop 端发起的视频通话无法接通，会一直处于 connecting 的状态。

## EOF

[Evil version](https://lemonhx.moe/2022/11/04/fuckmatrix/)
