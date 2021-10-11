---
title: sl-im 基于 Swoft 微服务协程框架和 Layim 网页聊天系统 开发出来的聊天室
tags: []
id: '26'
categories:
  - - Swoole
date: 2020-04-13 20:22:59
---

## 简介

[sl-im](https://im.stitch.cn) 是基于 [Swoft](https://www.swoft.org) 微服务协程框架和 [Layim](https://www.layui.com/layim/) 网页聊天系统 所开发出来的聊天室。

## 体验地址

[sl-im](https://im.stitch.cn) https://im.stitch.cn

## 演示图
![sl-im](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9xaW5pdS5nYW9iaW56aGFuLmNvbS8yMDIwLzA0LzEzL2E5NmIwMzFjNjYwY2EuanBn?x-oss-process=image/format,png)

## 功能

- 登录注册（Http）
- 单点登录（Websocket）
- 私聊（Websocket）
- 群聊（Websocket）
- 在线人数（Websocket）
- 获取未读消息（Websocket）
- 好友在线状态（Websocket）
- 好友 查找 添加 同意 拒绝（Http+Websocket）
- 群 创建 查找 添加 同意 拒绝（Http+Websocket）
- 聊天记录存储
- 心跳检测
- 消息重发
- 断线重连

## Requirement

- [PHP 7.1+](https://github.com/php/php-src/releases)
- [Swoole 4.3.4+](https://github.com/swoole/swoole-src/releases)
- [Composer](https://getcomposer.org/)
- [Swoft >= 2.0.8](https://github.com/swoft-cloud/swoft/releases/tag/v2.0.8)



## 部署方式

### Composer

```bash
composer update
```
### bean

`app/bean.php`


```bash
'db' => [
        'class'    => Database::class,
        'dsn'      => 'mysql:dbname=im;host=127.0.0.1:3306',
        'username' => 'root',
        'password' => 'gaobinzhan',
        'charset'  => 'utf8mb4',
    ],
'db.pool' => [
        'class'     => \Swoft\Db\Pool::class,
        'database'  => bean('db'),
        'minActive' => 5, // 自己调下连接池大小
        'maxActive' => 10
    ],
```

### 数据表迁移

` php bin/swoft mig:up`

### env配置

`vim .env`

```bash
# basic
APP_DEBUG=0
SWOFT_DEBUG=0

# more ...
APP_HOST=https://im.stitch.cn/
WS_URL=ws://im.stitch.cn/im
# 是否开启静态处理 这里我关了 让nginx去处理
ENABLE_STATIC_HANDLER=false 
# swoole v4.4.0以下版本, 此处必须为绝对路径
DOCUMENT_ROOT=/data/wwwroot/IM/public
```
### nginx配置

```bash
server{
    listen 80;
    server_name im.stitch.cn;
    return 301 https://$server_name$request_uri;
}

server{
    listen 443 ssl;
    root /data/wwwroot/IM/public/;
    add_header Strict-Transport-Security "max-age=31536000";
    server_name im.stitch.cn;
    access_log /data/wwwlog/im-stitch.cn.access.log;
    error_log /data/wwwlog/im-stitch.cn.error.log;
    client_max_body_size 100m;
    ssl_certificate /etc/nginx/ssl/full_chain.pem;
    ssl_certificate_key /etc/nginx/ssl/private.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    location / {
        proxy_pass http://127.0.0.1:9091;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Real-PORT $remote_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    location /im {
        proxy_pass http://127.0.0.1:9091;
        proxy_http_version 1.1;
        proxy_read_timeout   3600s;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    location ~ .*\.(js|ico|css|ttf|woff|woff2|png|jpg|jpeg|svg|gif|htm)$ {
        root /data/wwwroot/IM/public;
    }
}
```

### Start

- 挂起

```bash
php bin/swoft ws:start
```

- 守护进程化

```bash
php bin/swoft ws:start -d
```

- 访问

怎么访问还用写吗？？？点个star吧 ✌️

## 联系方式

- QQ：975975398