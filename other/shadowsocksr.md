# Get source code
```bash
cd /usr/local/
git clone -b manyuser https://github.com/shadowsocksr-backup/shadowsocksr.git
```

# server configure
init
```bash
cd /usr/local/shadowsocksr
bash initcfg.sh
```

create config
vim /usr/local/shadowsocksr/user-config.json
```vim
{
    "server": "0.0.0.0",
    "server_ipv6": "::",
    "server_port": 2111,  # change
    "local_address": "127.0.0.1",
    "local_port": 1080,

    "password": "password", # change
    "method": "aes-256-cfb",
    "protocol": "auth_aes128_md5",
    "protocol_param": "",
    "obfs": "http_simple",
    "obfs_param": "",  # add domain
    "speed_limit_per_con": 0,
    "speed_limit_per_user": 0,

    "additional_ports" : {}, // only works under multi-user mode
    "additional_ports_only" : false, // only works under multi-user mode
    "timeout": 120,
    "udp_timeout": 60,
    "dns_ipv6": false,
    "connect_verbose_info": 0,
    "redirect": "",
    "fast_open": false
}
```

vim /etc/systemd/system/shadowsocksr.service
```vim
[Unit]
Description=ShadowsocksR server
After=network.target
Wants=network.target

[Service]
Type=forking
PIDFile=/var/run/shadowsocksr.pid
ExecStart=/usr/bin/python /usr/local/shadowsocksr/shadowsocks/server.py --pid-file /var/run/shadowsocksr.pid -c /usr/local/shadowsocksr/user-config.json -d start
ExecStop=/usr/bin/python /usr/local/shadowsocksr/shadowsocks/server.py --pid-file /var/run/shadowsocksr.pid -c /usr/local/shadowsocksr/user-config.json -d stop
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=always

[Install]
WantedBy=multi-user.target
```

start service
```bash
systemctl enable shadowsocksr.service && systemctl start shadowsocksr.service
```
