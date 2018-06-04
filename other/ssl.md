# install certbot
```bash
wget https://dl.eff.org/certbot-auto -O /usr/local/sbin/certbot-auto
chmod a+x /usr/local/sbin/certbot-auto

```

# create certificate
```bash
certbot-auto certonly --webroot -w /opt/programs/nginx_1.14.0/challenges/ -d sparkknow.com -d www.sparkknow.com
#certbot-auto certonly --manual
```

# update certificate
```bash
certbot-auto renew --quiet --renew-hook "/opt/programs/nginx_1.14.0/sbin/nginx -s reload"
```
