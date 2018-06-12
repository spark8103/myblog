---
layout: post
title: Etcd+Confd+Nginx构建服务发现
date: 2018-06-12 13:35:02.000000000 +09:00
tags: 技术
---

# Etcd+Confd+Nginx构建服务发现

## 背景
微服务的兴起，需要我们可以方便的自动生成nginx配置，可以通过API上线下线后端服务器，因此需要服务的注册和配置更新功能。

通过Etcd存储nginx的域名及upstream的信息，通过Confd进行配置的自动更新。

etcd api(update k/v) ==> etcd ==> confd ==> nginx 

以下操作在Ubuntu下面执行，其它系统类似。

## nginx安装
```shell
~# apt install nginx
~# whereis nginx
nginx: /usr/sbin/nginx /usr/lib/nginx /etc/nginx /usr/share/nginx /usr/share/man/man8/nginx.8.gz
~# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

我们将用模板将配置生成在/etc/nginx/conf.d/路径下面，然后检查nginx配置，进行nginx配置生效。

## Confd安装
```shell
~# wget https://github.com/kelseyhightower/confd/releases/download/v0.16.0/confd-0.16.0-linux-amd64
~# mv confd-0.16.0-linux-amd64 /usr/sbin/confd
~# chmod +x /usr/sbin/confd
```

## Etcd安装
```shell
~# apt install etcd-server etcd-client
```

修改配置 /etc/default/etcd 添加以下配置
```vim
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"
```

重启生效
```shell
~# service etcd restart
```

## Confd配置
创建配置目录
```shell
~# mkdir -p /etc/confd/{conf.d,templates}
```

创建toml配置文件 /etc/confd/conf.d/app1.toml
```vim
[template]
prefix = "/app1"
src = "subdomain-nginx.conf.tmpl"
dest = "/etc/nginx/conf.d/app1-subdomain-confd-nginx-auto.conf"
keys = [
  "/subdomain",
  "/upstream",
]
check_cmd = "/usr/sbin/nginx -t"
reload_cmd = "/usr/sbin/nginx -s reload"
```

创建 .tmpl 模板文件 /etc/confd/templates/subdomain.conf.tmpl 
```vim
[subdomain]
subdomain = {{getv "/subdomain"}}
upstream = {{getv "/upstream"}}
```

配置nginx模板 /etc/confd/templates/subdomain-nginx.conf.tmpl 
```vim
upstream {{getv "/subdomain"}} {
{{range getvs "/upstream/*"}}
    server {{.}};
{{end}}
}

server {
    server_name  {{getv "/subdomain"}}.example.com;
    location / {
        proxy_pass        http://{{getv "/subdomain"}};
        proxy_redirect    off;
        proxy_set_header  Host             $host;
        proxy_set_header  X-Real-IP        $remote_addr;
        proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
   }
}
```

## 注册服务
通过etcdctl命令，插入数据，也可以通过http api接口。
```shell
~# etcdctl set /app1/subdomain app1
~# etcdctl set /app1/upstream/instance1 "192.168.5.1:8000"
~# etcdctl set /app1/upstream/instance2 "192.168.5.2:8000"
```

使用confd将配置生效
```shell
# 只处理一次
~# confd -onetime -backend etcd -node http://127.0.0.1:2379
# 按时间轮询
~# nohup confd -interval=60 -backend etcd -node http://127.0.0.1:2379 &
```
