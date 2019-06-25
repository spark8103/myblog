## pre install
```bash
yum install gcc gcc-c++ libtool automake autoconf make unzip -y
yum install pcre pcre-devel zlib-devel openssl-devel -y
```

## install LuaJIT
```bash
mkdir -p /root/software && cd /root/software
wget http://luajit.org/download/LuaJIT-2.1.0-beta3.tar.gz
tar -xzvf LuaJIT-2.1.0-beta3.tar.gz && cd LuaJIT-2.1.0-beta3 make && make install && cd ..
```

add /etc/profile
```vim
export LUAJIT_LIB=/usr/local/lib
export LUAJIT_INC=/usr/local/include/luajit-2.1/
```

exec source and link
```baseh
source /etc/profile
ln -sf luajit-2.1.0-beta3 /usr/local/bin/luajit
ln -snf /usr/local/lib/libluajit-5.1.so.2.1.0 /lib64/libluajit-5.1.so.2
```

## download nginx
```bash
mkdir -p /root/software && cd /root/software
wget https://nginx.org/download/nginx-1.14.0.tar.gz
tar zxvf nginx-* && cd nginx-*

vi src/core/nginx.h
#define NGINX_VERSION      "1.14.0" -> #define NGINX_VERSION      "2.6.0"
#define NGINX_VER          "nginx/" NGINX_VERSION -> #define NGINX_VER          "apache/" NGINX_VERSION
```

## patch nginx_upstream_check_module 
```bash
wget https://github.com/yaoweibin/nginx_upstream_check_module/archive/master.zip
unzip master.zip && rm -rf master.zip

sed -i 's/nginx-1.12.1_orig\///g' nginx_upstream_check_module-master/check_1.12.1+.patch
sed -i 's/nginx-1.12.1\///g' nginx_upstream_check_module-master/check_1.12.1+.patch

patch -p0 < nginx_upstream_check_module-master/check_1.12.1+.patch
```

## download nginx_devel_kit
```bash
wget https://github.com/simpl/ngx_devel_kit/archive/v0.3.0.tar.gz
tar -xzvf v0.3.0.tar.gz && rm -rf v0.3.0.tar.gz
```

## download nginx_lua
```bash
wget https://github.com/openresty/lua-nginx-module/archive/v0.10.11.tar.gz
tar zxvf v0.10.11.tar.gz && rm -rf v0.10.11.tar.gz
```

## compile nginx
```bash
./configure --prefix=/opt/programs/nginx_1.14.0 \
    --with-http_realip_module \
    --with-http_sub_module \
    --with-http_gzip_static_module \
    --with-http_stub_status_module \
    --with-http_ssl_module \
    --with-threads \
    --with-http_v2_module \
    --with-http_addition_module \
    --with-http_dav_module \
    --with-file-aio \
    --with-http_gunzip_module \
    --add-module=nginx_upstream_check_module-master \
    --add-module=ngx_devel_kit-0.3.0 \
    --add-module=lua-nginx-module-0.10.11

make
make install
/opt/programs/nginx_1.14.0/sbin/nginx -V
```

## configure nginx
vi /opt/programs/nginx_1.14.0/conf/nginx.conf 
```vim
user  nobody;
worker_processes  1;
error_log  logs/error.log;

pid logs/nginx.pid;
worker_rlimit_nofile 100000;
thread_pool default threads=2  max_queue=65535;

events {
    use epoll;
    multi_accept on;
    worker_connections  65535;
}

http {
    server_tokens off;
    include       mime.types;
    default_type  application/octet-stream;
    charset utf-8;

    server_names_hash_bucket_size 256;
    client_header_buffer_size     32k;
    large_client_header_buffers   4 128k;

    client_max_body_size          256m;
    client_body_buffer_size       256k;
    proxy_connect_timeout         600;  
    proxy_read_timeout            600; 
    proxy_send_timeout            600;
    proxy_buffer_size             32k;
    proxy_buffers                 4 32k;
    proxy_busy_buffers_size       64k; 
    proxy_temp_file_write_size    1024m;
    proxy_ignore_client_abort     on;

    real_ip_header     x-forwarded-for;
    real_ip_recursive on;
    proxy_set_header X-Forwarded-For $remote_addr;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    log_format logstash_json '{ "timestamp": "$time_iso8601", '
                        '"host": "$server_addr", '
                        '"user": "$remote_user", '
                        '"request": "$request", '
                        '"clientip": "$remote_addr", '
                        '"size": $body_bytes_sent, '
                        '"responsetime": $request_time, '
                        '"upstreamtime": "$upstream_response_time", '
                        '"upstreamhost": "$upstream_addr", '
                        '"upstreamstatus": "$upstream_status", '
                        '"http_host": "$host", '
                        '"url": "$uri", '
                        '"referrer": "$http_referer", '
                        '"x_forwarded_for": "$http_x_forwarded_for", '
                        '"agent": "$http_user_agent", '
                        '"status": "$status"} ';

    access_log  logs/access.log  logstash_json;

    sendfile        on;
    tcp_nopush      on;
    keepalive_timeout  30;

    include upstream.d/*.conf;
    include conf.d/*.conf;
}
```

vi /etc/security/limits.d/20-nproc.conf
```vim
# Default limit for number of user's processes to prevent
# accidental fork bombs.
# See rhbz #432903 for reasoning.

*       hard    nproc     1000000
*       soft    nproc     1000000
*       hard    nofile    1000000
*       soft    nofile    1000000
root    soft    nproc     unlimited
```

add service
```bash
cat > /usr/lib/systemd/system/nginx.service << "EOF"
[Unit]
Description=nginx - high performance web server
Documentation=https://nginx.org/en/docs/
After=network.target remote-fs.target nss-lookup.target
 
[Service]
Type=forking
PIDFile=/opt/programs/nginx_1.14.0/logs/nginx.pid
ExecStartPre=/opt/programs/nginx_1.14.0/sbin/nginx -t -c /opt/programs/nginx_1.14.0/conf/nginx.conf
ExecStart=/opt/programs/nginx_1.14.0/sbin/nginx -c /opt/programs/nginx_1.14.0/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
EOF

mkdir /opt/programs/nginx_1.14.0/conf/upstream.d /opt/programs/nginx_1.14.0/conf/conf.d /opt/programs/nginx_1.14.0/conf/ssl
systemctl enable nginx.service
systemctl start nginx.service
```

## add nginx count
download file
```bash
wget https://github.com/GuyCheung/falcon-ngx_metric/archive/master.zip
unzip master.zip && rm -rf master.zip
mv falcon-ngx_metric-master/lua/ /opt/programs/nginx_1.14.0/modules

vi /opt/programs/nginx_1.14.0/conf/conf.d/ngx_metric.conf 
```vim
lua_package_path "/opt/programs/nginx_1.14.0/modules/?.lua;;";
lua_shared_dict result_dict 128M;
log_by_lua_file /opt/programs/nginx_1.14.0/modules/ngx_metric.lua;

server {
    listen          127.0.0.1:9091;
    server_name     localhost;

    location /monitor/basic_status {
        content_by_lua_file modules/ngx_metric_output.lua;
        access_log /dev/null;
        allow 127.0.0.1;
        deny all;
    }

    location /monitor/nginx_status {
        stub_status on;
        access_log /dev/null ;
        allow 127.0.0.1;
        deny all;
    }

    location /monitor/nstatus {
        check_status;
        access_log /dev/null;
        allow 127.0.0.1;
        deny all;
    }
}
```

## add domain configure
vi default.conf
```vim
server {
    server_name  _;
    listen       80 default_server;

    proxy_set_header Host    $host;
    proxy_set_header X-Real-IP  $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    access_log   logs/access.log main;
    error_log    logs/error.log;

    location / {
        rewrite ^/(.*)$ https://www.sparkknow.com/$1 permanent;
    }


    error_page    403 404 500 502 503 504  /404.html;
    location = /404.html {
        root html/;
    }
}
```

vi sparkknow.com.conf
```vim
server {
    server_name sparkknow.com www.sparkknow.com reg.sparkknow.com;
    listen 80;

    proxy_set_header Host    $host;
    proxy_set_header X-Real-IP  $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    access_log   logs/sparkknow.com-access.log main;
    error_log    logs/sparkknow.com-error.log;

    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        alias /opt/programs/nginx_1.14.0/challenges/;
        try_files $uri =404;
    }

    location / {
        rewrite ^/(.*)$ https://www.sparkknow.com/$1 permanent;
        #root html/;
    }


    error_page    403 404 500 502 503 504  /404.html;
    location = /404.html {
        root html/;
    }
}

server {
    server_name sparkknow.com;
    listen 443 ssl http2;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-Xss-Protection 1;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/sparkknow.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sparkknow.com/privkey.pem;

    ssl_session_timeout 30m;
    ssl_session_cache   shared:SSL:30m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    #ssl_ciphers TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+ECDSA+AES128:EECDH+aRSA+AES128:RSA+AES128:EECDH+ECDSA+AES256:EECDH+aRSA+AES256:RSA+AES256:EECDH+ECDSA+3DES:EECDH+aRSA+3DES:RSA+3DES:!MD5;
    ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4";
    ssl_prefer_server_ciphers on;

    proxy_set_header Host    $host;
    proxy_set_header X-Real-IP  $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    access_log   logs/sparkknow.com-access.log main;
    error_log    logs/sparkknow.com-error.log;


    location / {
        rewrite ^/(.*)$ https://www.sparkknow.com/$1 permanent;
    }

    error_page    403 404 500 502 503 504  /404.html;
    location = /404.html {
        root /opt/programs/nginx_1.14.0/html/;
    }
}

server {
    server_name www.sparkknow.com;
    listen 443 ssl http2;
   
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-Xss-Protection 1;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/sparkknow.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sparkknow.com/privkey.pem;

    ssl_session_timeout 30m;
    ssl_session_cache   shared:SSL:30m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    #ssl_ciphers TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+ECDSA+AES128:EECDH+aRSA+AES128:RSA+AES128:EECDH+ECDSA+AES256:EECDH+aRSA+AES256:RSA+AES256:EECDH+ECDSA+3DES:EECDH+aRSA+3DES:RSA+3DES:!MD5;
    ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4";
    ssl_prefer_server_ciphers on;

    proxy_set_header Host    $host;
    proxy_set_header X-Real-IP  $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    access_log   logs/www.sparkknow.com-access.log main;
    error_log    logs/www.sparkknow.com-error.log;


    location / {
        root /opt/programs/nginx_1.14.0/html/;
    }

    error_page    403 404 500 502 503 504  /404.html;
    location = /404.html {
        root /opt/programs/nginx_1.14.0/html/;
    }
}
```
