# install certbot
```bash
wget https://dl.eff.org/certbot-auto -O /usr/local/sbin/certbot-auto
chmod a+x /usr/local/sbin/certbot-auto

```

# create certificate
```bash
certbot-auto certonly --webroot -w /opt/programs/nginx_1.14.0/challenges/ -d sparkknow.com -d www.sparkknow.com -d reg.sparkknow.com

#certbot-auto certonly --manual
```

# update certificate
```bash
certbot-auto renew --quiet --renew-hook "/opt/programs/nginx_1.14.0/sbin/nginx -s reload"
```

# nginx config
```vim
    ssl on;
    ssl_certificate /etc/letsencrypt/live/sparkknow.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/sparkknow.com/privkey.pem;
```

# add --manual-auth-hook script
vim /etc/letsencrypt/renewal/sparkknow.com.sh
```vim
#!/bin/bash
echo $CERTBOT_VALIDATION > /opt/programs/nginx_1.14.0/challenges/$CERTBOT_TOKEN
```

添加执行权限
```bash
chmod +x /etc/letsencrypt/renewal/sparkknow.com.sh
```

测试是否成功
```bash
certbot-auto renew --dry-run --manual-auth-hook /etc/letsencrypt/renewal/sparkknow.com.sh
```

# add crontab
```vim
30 5 1 * * root /usr/local/sbin/certbot-auto renew --manual-auth-hook /etc/letsencrypt/renewal/sparkknow.com.sh --renew-hook "/opt/programs/nginx_1.14.0/sbin/nginx -s reload"
```

# other doc
* [certbot-auto教程](https://linuxstory.org/deploy-lets-encrypt-ssl-certificate-with-certbot/)
* [Let’s Encrypt 加密SSL证书并强制启用HTTPS访问]](https://blog.mimvp.com/article/20468.html)
