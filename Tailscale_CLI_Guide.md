# Tailscale 命令行玩法

## 局域网1设置部分

```bash
tailscale up --netfilter-mode=off --advertise-routes=局域网网段 --accept-routes
```

### 命令解析

这条命令用于配置Tailscale节点，特别是将该节点设置为一个子网路由器(Subnet Router)，下面是各参数的详细说明：

- `tailscale up`: 启动Tailscale服务并连接到Tailscale网络
- `--netfilter-mode=off`: 关闭Linux内核的netfilter模块交互，这在某些特定环境中可能需要（比如已有防火墙规则或需要特定网络设置时）
- `--advertise-routes=局域网网段`: 向Tailscale网络广播该设备可以路由的本地子网，这里的"局域网网段"应替换为实际的CIDR格式网段，例如`192.168.1.0/24`。这允许其他Tailscale节点通过此设备访问该子网中的设备
- `--accept-routes`: 允许该设备接受其他Tailscale节点广播的路由，使该设备可以访问其他子网路由器公布的网段

当这个命令执行后，需要在Tailscale的管理控制台上批准这个路由广播。批准后，其他Tailscale节点就可以通过此设备访问指定的局域网网段。

## 群晖设置部分

```bash
sudo -i

# 设置TUN模块
echo -e '#!/bin/sh -e \ninsmod /lib/modules/tun.ko' > /usr/local/etc/rc.d/tun.sh
chmod a+x /usr/local/etc/rc.d/tun.sh
/usr/local/etc/rc.d/tun.sh
ls /dev/net/tun
```

> 参考ZeroTier群晖的安装
> 资料：https://docs.zerotier.com/devices/synology

### Docker安装

```bash
docker run -d \
  --name=ts \
  --restart=always \
  -v /var/lib:/var/lib \
  -v /dev/net/tun:/dev/net/tun \
  --network=host \
  --cap-add=NET_ADMIN \
  --cap-add=NET_RAW \
  --env TS_STATE_DIR=/etc/ts \
  --env TS_SOCKET=/var/run/tailscale/tailscaled.sock \
  --env TS_USERSPACE=false \
  --env TS_ROUTES=局域网网段 \
  --env TS_EXTRA_ARGS="--accept-routes --advertise-exit-node --reset" \
  --env TS_AUTHKEY=API密钥 \
  tailscale/tailscale
```

### 防火墙设置

```bash
iptables -I FORWARD -i eth0 -j ACCEPT
iptables -I FORWARD -o eth0 -j ACCEPT
iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE
iptables -I FORWARD -i tailscale0 -j ACCEPT
iptables -I FORWARD -o tailscale0 -j ACCEPT
iptables -t nat -I POSTROUTING -o tailscale0 -j MASQUERADE

sleep 1m
```

## 云服务器DERP中转服务器搭建部分

### 系统准备

```bash
apt update && apt upgrade

apt install -y wget git openssl curl
```

### 安装Go

```bash
wget https://go.dev/dl/go1.20.5.linux-amd64.tar.gz

rm -rf /usr/local/go && tar -C /usr/local -xzf go1.20.5.linux-amd64.tar.gz

export PATH=$PATH:/usr/local/go/bin
go version

echo "export PATH=$PATH:/usr/local/go/bin" >> /etc/profile
source /etc/profile

go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```

### 安装DERP服务器

```bash
go install tailscale.com/cmd/derper@main

go build -o /etc/derp/derper

ls /etc/derp
```

### 创建SSL证书

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -keyout /etc/derp/derp.myself.com.key \
  -out /etc/derp/derp.myself.com.crt \
  -subj "/CN=derp.myself.com" \
  -addext "subjectAltName=DNS:derp.myself.com"
```

### 创建系统服务

```bash
cat > /etc/systemd/system/derp.service <<EOF
[Unit]
Description=TS Derper
After=network.target
Wants=network.target
[Service]
User=root
Restart=always
ExecStart=/etc/derp/derper -hostname derp.myself.com -a :33445 -http-port 33446 -certmode manual -certdir /etc/derp
RestartPreventExitStatus=1
[Install]
WantedBy=multi-user.target
EOF

systemctl enable derp
systemctl start derp
```

### DERP配置

```json
"derpMap": {
    "OmitDefaultRegions": true,
    "Regions": {
        "901": {
            "RegionID":   901,
            "RegionCode": "Myself",
            "RegionName": "Myself Derper",
            "Nodes": [
                {
                    "Name":             "901a",
                    "RegionID":         901,
                    "DERPPort":         33445,
                    "IPv4":   "服务器IP",
                    "InsecureForTests": true,
                },
            ],
        },
    },
},
```

### 禁用默认区域

```json
"1":  null,
"2":  null,
"3":  null,
"4":  null,
"5":  null,
"6":  null,
"7":  null,
"8":  null,
"9":  null,
"10": null,
"11": null,
"12": null,
"13": null,
"14": null,
"15": null,
"16": null,
"17": null,
"18": null,
"19": null,
"20": null,
"21": null,
"22": null,
"23": null,
"24": null,    
"25": null,
```

### 常用命令

```bash
tailscale netcheck
tailscale status
tailscale ping 
ping6 240C::6666
tailscale down 
tailscale up
curl -fsSL https://tailscale.com/install.sh | sh
```

### 调整配置

```bash
nano /etc/systemd/system/derp.service

# 添加 --verify-clients 参数

systemctl daemon-reload
systemctl restart derp
```

## Headscale搭建部分

### 安装Headscale

```bash
wget --output-document=headscale.deb \
     https://github.com/juanfont/headscale/releases/download/v0.22.3/headscale_0.22.3_linux_amd64.deb

sudo dpkg --install headscale.deb

sudo systemctl enable headscale

nano /etc/headscale/config.yaml

apt install -y nginx
```

### Nginx配置

```nginx
map $http_upgrade $connection_upgrade {
    default      keep-alive;
    'websocket'  upgrade;
    ''           close;
}
server {
    listen 3355;
    listen [::]:3355;
    server_name 云服务器IP;
    location / {
    
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $server_name;
        proxy_buffering off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
        add_header Strict-Transport-Security "max-age=15552000; includeSubDomains" always;
    }
    
    location /web {
        index  index.html;
        alias  /var/www/web;
    }
}
```

### 安装Web UI

```bash
wget https://github.com/gurucomputing/headscale-ui/releases/download/2023.01.30-beta-1/headscale-ui.zip

unzip -d /var/www headscale-ui.zip

systemctl start headscale

systemctl restart nginx
```

### 创建API密钥并配置

```bash
headscale apikeys create --expiration 9999d

tailscale logout
tailscale up --login-server=http://云服务器IP:3355

touch /var/www/derp.json
```

### DERP配置JSON

```json
{
    "Regions": {
        "901": {
            "RegionID":   901,
            "RegionCode": "Myself",
            "RegionName": "Myself Derper",
            "Nodes": [
                {
                    "Name":             "901a",
                    "RegionID":         901,
                    "DERPPort":         33445,
                    "IPv4":   "IP地址",
                    "IPv6":    "IP地址",
                    "InsecureForTests": true
                }
            ]
        }
    }
}
```

### 静态文件服务

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name 127.0.0.1;

    root /var/www;
    index index.html index.htm index.nginx-debian.html;
    location /d {
        alias   /var/www;
        autoindex on;
    }
    location / {
        try_files $uri $uri/ =404;
    }
}
```

访问地址：`http://127.0.0.1/d/derp.json`

### 重启服务和配置

```bash
systemctl restart nginx
systemctl restart headscale

tailscale logout
tailscale up --login-server=http://你的云服务器ip:端口
```

## 改善GitHub下载速度慢的解决方案

下面代码可以放到`/etc/hosts`文件的末尾，然后重启云服务器就可以

```
20.205.243.166 github.com
159.24.3.173 gist.github.com
185.199.110.153 assets-cdn.github.com
185.199.110.153 raw.githubusercontent.com
185.199.110.153 gist.githubusercontent.com
185.199.110.153 cloud.githubusercontent.com
185.199.110.153 camo.githubusercontent.com
185.199.110.153 avatars0.githubusercontent.com
185.199.110.153 avatars1.githubusercontent.com
185.199.110.153 avatars2.githubusercontent.com
185.199.110.153 avatars3.githubusercontent.com
185.199.110.153 avatars4.githubusercontent.com
185.199.110.153 avatars5.githubusercontent.com
185.199.110.153 avatars6.githubusercontent.com
185.199.110.153 avatars7.githubusercontent.com
185.199.110.153 avatars8.githubusercontent.com
```

> 注意：IP可能之后需要更新，可以去站长工具，ping后面的域名来获取。 