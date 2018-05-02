---
layout: post
title: Install Harbor
date: 2018-01-04 08:45:15.000000000 +09:00
tags: 技术
---

# Harbor简介
Harbor是一个企业级的docker私有仓库解决方案，提供更多的docker registry功能，是vmware公司开发的开源软件。这里主要介绍安装及使用。

# Harbor前置安装
Harbor安装依赖于docker和docker-compose，操作系统建议centos7，以下安装基于centos7。

## docker安装
### yum源安装
```shell
yum -y install yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
rpm --import https://download.docker.com/linux/centos/gpg
yum makecache

yum install docker-ce -y
```

### docker配置
vi /usr/lib/systemd/system/docker.service 
```shell
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS $DOCKER_OPTS $DOCKER_DNS_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

mkdir -p /usr/lib/systemd/system/docker.service.d && vi /usr/lib/systemd/system/docker.service.d/docker-options.conf
```shell
[Service]
Environment="DOCKER_OPTS=--insecure-registry=10.254.0.0/16 --graph=/opt/docker --registry-mirror=https://reg.sparkknow.com"
```

启动docker
```shell
systemctl daemon-reload
systemctl start docker
```

## docker-compose安装
注意查下版本信息， [官网安装链接](https://docs.docker.com/compose/install/#install-compose) 。
```shell
curl -L https://github.com/docker/compose/releases/download/1.18.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

# Harbor安装
## 下载解压Harbor
下载最新的Release版本，网速好可下载在线安装版本。
```shell
wget https://github.com/vmware/harbor/releases/download/v1.2.2/harbor-online-installer-v1.2.2.tgz
tar zxvf harbor-online-installer-v1.2.2.tgz
cd harbor
```

## 配置Harbor
vi harbor.cfg #可修改你的主机名，数据库账号密码，管理员账号密码等信息。
```shell
hostname = reg.sparkknow.com
ui_url_protocol = http
db_password = xxxxxxxx
clair_db_password = xxxxxxxx
harbor_admin_password = xxxxxxxx
self_registration = off
project_creation_restriction = adminonly
```

vi docker-compose.yml #可修改端口等信息，我这里使用nginx做了转发
```shell
  proxy:
    ports:
      - 8011:80
      - 1443:443
      - 4443:4443
```

修改模板配置，在common/templates/nginx/nginx.http.conf中,找到location /, location /v2/ and location /service/这3个配置块， 将这三个配置块中的proxy_set_header X-Forwarded-Proto $scheme;配置注释掉。

修改common/templates/registry/config.yml，修改auth.token.realm的地址：
```shell
auth:
  token:
    issuer: harbor-token-issuer
    realm: https://reg.sparkknow.com/service/token
    # realm: $ui_url/service/token
    rootcertbundle: /etc/registry/root.crt
    service: harbor-registry
```

执行安装
```shell
[root@test harbor]# ./install.sh
? ----Harbor has been installed and started successfully.----

Now you should be able to visit the admin portal at http://reg.sparkknow.com. 
For more details, please visit https://github.com/vmware/harbor .
```

vi reg.sparkknow.com.conf #nginx转发配置
```shell
server {
    server_name  reg.sparkknow.com;
    listen 443;

    ssl on;
    ssl_certificate /xxx/xxx/xxx/reg.sparkknow.com.pem;
    ssl_certificate_key /xxx/xxx/xx/reg.sparkknow.com.key;

    ssl_session_timeout 5m;
    ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
    ssl_prefer_server_ciphers on;

    proxy_set_header Host    $host;
    proxy_set_header X-Real-IP  $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    access_log   logs/reg.sparkknow.com-access.log main;
    error_log    logs/reg.sparkknow.com-error.log;

    location / {
        proxy_pass http://localhost:8011;
    }
}
```

# Harbor维护
### 停止Harbor
```shell
docker-compose stop
```

### 启动Harbor
```shell
docker-compose start
```

### 更新Harbor配置
```shell
docker-compose down -v
vim harbor.cfg
./prepare
docker-compose up -d
```

### 删除Harbor
```shell
docker-compose down -v
rm -r /data/database
rm -r /data/registry
```

### 网页登陆Harbor
https://reg.sparkknow.com/ 管理员登陆后可进行项目管理，用户管理。

### 关闭自注册
需要通过登陆页面界面配置管理里面关闭。

### push镜像
```shell
docker pull alpine:3.7
docker tag alpine:3.7 reg.sparkknow.com/k8s/alpine:3.7
docker login reg.sparkknow.com
docker push reg.sparkknow.com/k8s/alpine:3.7
```

### pull镜像
```shell
docker login reg.sparkknow.com
docker pull reg.sparkknow.com/k8s/alpine:3.7
```

### 退出登陆
```shell
docker logout reg.sparkknow.com
```
