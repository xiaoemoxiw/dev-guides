---
title: Tailscale DERP 中转服务器搭建指南
author: Tailscale 中文社区
date: 2025-06-01
---

<!-- 侧边栏目录 - GitHub风格 -->
<details>
<summary>目录</summary>

- [Tailscale DERP 中转服务器搭建指南](#tailscale-derp-中转服务器搭建指南)
  - [1. 什么是 DERP 中转服务器？](#1-什么是-derp-中转服务器)
    - [1.1 DERP 的作用](#11-derp-的作用)
    - [1.2 流量流向](#12-流量流向)
  - [2. 为什么需要自建 DERP 服务器？](#2-为什么需要自建-derp-服务器)
  - [3. 准备工作](#3-准备工作)
    - [3.1 所需资源](#31-所需资源)
    - [3.2 系统要求](#32-系统要求)
  - [4. 搭建流程](#4-搭建流程)
    - [4.1 安装 Go 环境](#41-安装-go-环境)
    - [4.2 安装 Tailscale](#42-安装-tailscale)
    - [4.3 安装 DERP 服务器](#43-安装-derp-服务器)
    - [4.4 配置域名与证书](#44-配置域名与证书)
    - [4.5 创建 DERP 服务启动脚本](#45-创建-derp-服务启动脚本)
    - [4.6 创建系统服务](#46-创建系统服务)
    - [4.7 验证 DERP 服务](#47-验证-derp-服务)
    - [4.8 配置 Tailscale 使用自建 DERP 服务器](#48-配置-tailscale-使用自建-derp-服务器)
  - [5. 连接流程详解](#5-连接流程详解)
  - [6. 工作原理简述](#6-工作原理简述)
  - [7. 故障排查](#7-故障排查)
  - [8. 维护事项](#8-维护事项)
    - [8.1 证书更新](#81-证书更新)
    - [8.2 服务监控](#82-服务监控)
  - [9. 性能优化建议](#9-性能优化建议)
  - [10. 常见问题解答](#10-常见问题解答)
</details>

# Tailscale DERP 中转服务器搭建指南

## 1. 什么是 DERP 中转服务器？

DERP（Designated Encrypted Relay for Packets）是 Tailscale 用于中转流量的服务器组件。它是一个简单但高效的中继服务，当两个 Tailscale 客户端无法直接建立连接时，它会负责转发加密数据包。

### 1.1 DERP 的作用

DERP 中转服务器在以下情况下会被使用：

1. **NAT 穿透失败**：当两台设备因为网络环境（如对称型 NAT）无法直接建立连接
2. **连接初期**：所有 Tailscale 连接首先通过 DERP 建立，以确保即时连接
3. **网络切换**：当网络环境变化（如从 WiFi 切换到移动网络）导致直连中断
4. **严格防火墙环境**：企业网络或仅允许 HTTP/HTTPS 出站流量的环境

重要的是，DERP 中转服务器**不同于**协调服务器（Control Server）：
- **协调服务器**：负责设备认证、密钥交换和配置分发
- **DERP 服务器**：仅负责在需要时中转加密数据包

### 1.2 流量流向

```
┌─────────┐    无法直连     ┌─────────┐
│ 设备 A  │<-------------->│ 设备 B  │
└────┬────┘                 └────┬────┘
     │                           │
     │     ┌──────────────┐     │
     └────>│ DERP 中转服务器 │<────┘
           └──────────────┘
           
数据始终是端到端加密的，DERP 服务器
无法看到流量内容
```

## 2. 为什么需要自建 DERP 服务器？

Tailscale 官方在全球多个区域部署了 DERP 服务器，但在中国国内使用时可能面临以下问题：

1. **高延迟**：官方 DERP 服务器大多位于国外，国内用户访问延迟高
2. **连接不稳定**：国际网络波动可能导致连接质量不稳定
3. **初始连接慢**：每次建立连接首先使用 DERP，国外服务器导致连接建立慢
4. **网络限制**：部分网络环境可能限制国际连接

自建 DERP 服务器的优势：
- 显著降低延迟
- 提高连接稳定性
- 加快初始连接建立速度
- 在特殊网络环境下提高成功率

## 3. 准备工作

### 3.1 所需资源

- 一台有公网 IP 的服务器（国内云服务商如阿里云、腾讯云等）
- 一个已备案的域名（用于 HTTPS 证书）
- 基本的 Linux 命令行操作知识

### 3.2 系统要求

- 操作系统：Ubuntu 20.04/22.04 或 CentOS 7/8（推荐 Ubuntu）
- 内存：至少 1GB
- CPU：1 核心（低负载情况足够）
- 网络：
  - 公网 IP（静态 IP 更佳）
  - 开放 443/TCP 端口（HTTPS 服务）
  - 开放 3478/UDP 端口（STUN 服务）

## 4. 搭建流程

### 4.1 安装 Go 环境

DERP 服务器是用 Go 语言编写的，需要安装 Go 环境：

```bash
# 下载并安装 Go
wget https://go.dev/dl/go1.20.6.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.20.6.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc

# 验证安装
go version
```

### 4.2 安装 Tailscale

在服务器上安装 Tailscale 客户端，这将允许 DERP 服务器验证客户端：

```bash
# Ubuntu/Debian
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/focal.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/focal.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list
sudo apt-get update
sudo apt-get install tailscale

# CentOS/RHEL
dnf config-manager --add-repo https://pkgs.tailscale.com/stable/centos/8/tailscale.repo
dnf install tailscale

# 启动并登录 Tailscale
sudo tailscale up
```

### 4.3 安装 DERP 服务器

使用 Go 工具安装 DERP 服务器：

```bash
# 安装 derper 工具
go install tailscale.com/cmd/derper@latest

# 创建证书目录
mkdir -p ~/derper/certs
```

### 4.4 配置域名与证书

#### 域名设置
1. 在域名提供商处添加 A 记录，指向服务器公网 IP
   例如：`derp.yourdomain.com -> 你的服务器IP`

#### 使用 Let's Encrypt 证书
使用 certbot 获取免费 SSL 证书：

```bash
# 安装 certbot
sudo apt install certbot

# 获取证书（请替换为你的域名）
sudo certbot certonly --standalone -d derp.yourdomain.com

# 复制证书到 DERP 证书目录
sudo cp /etc/letsencrypt/live/derp.yourdomain.com/fullchain.pem ~/derper/certs/
sudo cp /etc/letsencrypt/live/derp.yourdomain.com/privkey.pem ~/derper/certs/
sudo chown -R $USER:$USER ~/derper/certs/
```

### 4.5 创建 DERP 服务启动脚本

创建文件 `~/derper/run.sh`：

```bash
#!/bin/bash

HOSTNAME="derp.yourdomain.com"  # 替换为你的域名
CERTDIR="$HOME/derper/certs"
ADDR=":443"   # HTTPS 端口
STUN_PORT=3478  # STUN 服务端口

$HOME/go/bin/derper \
  --hostname=$HOSTNAME \
  --certdir=$CERTDIR \
  --certmode=manual \
  --a=$ADDR \
  --stun-port=$STUN_PORT \
  --verify-clients
```

添加执行权限：

```bash
chmod +x ~/derper/run.sh
```

### 4.6 创建系统服务

创建服务定义文件：

```bash
sudo nano /etc/systemd/system/derper.service
```

填入以下内容：

```
[Unit]
Description=Tailscale DERP Server
After=network.target

[Service]
Type=simple
User=YOUR_USERNAME  # 替换为你的用户名
ExecStart=/home/YOUR_USERNAME/derper/run.sh
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable derper
sudo systemctl start derper
```

### 4.7 验证 DERP 服务

确认服务正常运行：

```bash
# 检查服务状态
sudo systemctl status derper

# 测试 HTTPS 连接
curl -v https://derp.yourdomain.com

# 应该返回类似以下内容:
# <html><body><h1>DERP</h1><p>This is a Tailscale <a href="https://pkgs.go.dev/tailscale.com/derp">DERP</a> server.</p></body></html>
```

### 4.8 配置 Tailscale 使用自建 DERP 服务器

登录 Tailscale 管理页面（https://login.tailscale.com/admin/），修改 ACL 策略：

```json
{
  "acls": [
    // 你现有的 ACL 策略
  ],
  
  // 添加自定义 DERP 地图
  "derpMap": {
    "OmitDefaultRegions": false,  // 设置为 true 将只使用自定义 DERP
    "Regions": {
      "900": {
        "RegionID": 900,
        "RegionCode": "cn",
        "RegionName": "China Custom DERP",
        "Nodes": [
          {
            "Name": "cn1",
            "RegionID": 900,
            "HostName": "derp.yourdomain.com"  // 你的 DERP 服务器域名
          }
        ]
      }
    }
  }
}
```

> **注意**：RegionID 900 及以上被保留用于自定义 DERP 服务器。官方 DERP 服务器使用 1-899 的 ID。

保存配置后，Tailscale 会自动将这些配置推送到所有客户端。

## 5. 连接流程详解

Tailscale 设备连接流程图：

```
┌───────────────────────────────────────────────────────────────────┐
│                     Tailscale 连接流程                             │
└───────────────────────────────────────────────────────────────────┘

┌────────┐               ┌────────────────┐               ┌────────┐
│        │  1. 认证请求  │                │  1. 认证请求  │        │
│ 设备 A ├──────────────>│ 协调服务器     ├──────────────>│ 设备 B │
│        │               │(Control Server)│               │        │
└────┬───┘               └────────┬───────┘               └────┬───┘
     │                            │                            │
     │  2. 接收密钥和网络信息     │ 2. 接收密钥和网络信息      │
     │<───────────────────────────┴────────────────────────────┘
     │                                                          │
     │  3. 尝试直接连接 (NAT 穿透)                              │
     │<─────────────────────────────────────────────────────────┘
     │                                                          │
     │                                                          │
     │                  ┌───────────────┐                       │
     │  4a. 直连成功    │               │   4a. 直连成功        │
     │<─────────────────┤    互联网     ├───────────────────────┘
     │                  │               │                       │
     │                  └───────┬───────┘                       │
     │                          │                               │
     │                          │                               │
     │                  ┌───────┴───────┐                       │
     │  4b. 直连失败    │   自建 DERP   │   4b. 直连失败        │
     └─────────────────>│   中转服务器  │<──────────────────────┘
                        │ (已加密流量)  │
                        └───────────────┘
```

## 6. 工作原理简述

Tailscale 连接的工作原理可分为四个主要阶段：

1. **协调阶段**：
   - 设备连接到 Tailscale 协调服务器进行认证
   - 协调服务器分发网络配置、公钥信息和 DERP 服务器信息
   - 协调服务器告知设备其他设备的状态和地址信息

2. **连接建立阶段**：
   - 所有连接首先通过 DERP 服务器建立初始连接
   - 这确保连接能够立即建立，无需等待 NAT 穿透
   - 同时，设备开始尝试通过多种 NAT 穿透技术建立直连

3. **数据传输阶段**：
   - 如果直连成功，数据直接在设备间点对点传输
   - 如果直连失败，数据通过 DERP 服务器中继
   - 所有数据都使用 WireGuard 端到端加密，DERP 服务器无法解密

4. **连接维护阶段**：
   - DERP 连接始终保持活跃，作为备用通道
   - 系统会持续尝试建立更优的连接路径
   - 当网络环境变化时，可能会自动切换路径

## 7. 故障排查

当遇到 DERP 服务器问题时，可以按照以下步骤进行排查：

1. **服务无法启动**
   - 检查防火墙设置：`sudo ufw status` 确保端口开放
   - 检查日志：`sudo journalctl -u derper.service`
   - 确认证书路径正确：`ls -la ~/derper/certs/`

2. **客户端无法连接**
   - 验证域名解析：`nslookup derp.yourdomain.com`
   - 检查 ACL 策略是否正确配置
   - 验证证书是否有效：`openssl s_client -connect derp.yourdomain.com:443`
   - 确认 `--verify-clients` 选项与 Tailscale 登录状态一致

3. **延迟仍然很高**
   - 运行 `tailscale ping <对端IP>` 查看连接类型和延迟
   - 检查是否成功使用了自建 DERP：`tailscale netcheck`
   - 使用 `mtr` 查看网络路径：`mtr derp.yourdomain.com`

4. **常见错误消息**：
   - `connection refused`：检查服务是否运行并监听正确端口
   - `certificate expired`：更新 SSL 证书
   - `dial tcp: lookup ... no such host`：检查 DNS 解析

## 8. 维护事项

### 8.1 证书更新

Let's Encrypt 证书有效期为 90 天，需要定期更新。创建证书自动更新脚本 `~/derper/renew-cert.sh`：

```bash
#!/bin/bash
sudo certbot renew
sudo cp /etc/letsencrypt/live/derp.yourdomain.com/fullchain.pem ~/derper/certs/
sudo cp /etc/letsencrypt/live/derp.yourdomain.com/privkey.pem ~/derper/certs/
sudo chown -R $USER:$USER ~/derper/certs/
sudo systemctl restart derper
```

添加 crontab 任务，每月自动更新：

```bash
chmod +x ~/derper/renew-cert.sh
(crontab -l ; echo "0 0 1 * * ~/derper/renew-cert.sh") | crontab -
```

### 8.2 服务监控

为确保 DERP 服务持续稳定运行，建议：

1. 设置基本监控，监控服务状态
2. 定期检查日志：`sudo journalctl -u derper.service`
3. 配置系统资源监控，特别是内存和带宽使用情况
4. 创建简单的健康检查脚本，定期测试 DERP 服务可用性

## 9. 性能优化建议

1. **服务器选择**
   - 选择网络质量好的云服务商
   - 如有可能，选择带有 BGP 加速的服务器
   - 服务器地理位置尽量接近主要用户位置

2. **系统优化**
   - 调整系统 TCP 参数：
     ```bash
     # 编辑 /etc/sysctl.conf 添加：
     net.core.somaxconn = 4096
     net.ipv4.tcp_max_syn_backlog = 4096
     ```
   - 增加文件描述符限制：
     ```bash
     # 编辑 /etc/security/limits.conf 添加：
     * soft nofile 65536
     * hard nofile 65536
     ```

3. **多区域部署**
   - 如果用户分布在不同地区，考虑部署多个 DERP 服务器
   - 在 ACL 中配置多个区域，Tailscale 将自动选择最佳的服务器

## 10. 常见问题解答

**Q: DERP 服务器会影响 Tailscale 的安全性吗？**

A: 不会。DERP 服务器只能看到加密的数据包，无法解密内容。所有通信仍然使用 WireGuard 端到端加密。

**Q: 如何确认客户端正在使用我的自建 DERP 服务器？**

A: 运行 `tailscale netcheck` 命令，查看输出中是否包含你自建的 DERP 服务器地址和延迟信息。

**Q: 我需要付费 Tailscale 计划才能使用自建 DERP 吗？**

A: 不需要。自建 DERP 服务器功能在免费计划中也可以使用。

**Q: 自建 DERP 服务器的流量有限制吗？**

A: Tailscale 本身不对 DERP 中转的流量设限制，但你的服务器提供商可能有带宽限制。

**Q: 如何禁用官方 DERP 服务器？**

A: 在 ACL 配置中设置 `"OmitDefaultRegions": true`，客户端将仅使用自定义 DERP 服务器。

---

通过本指南，你已成功搭建了一个位于中国国内的 Tailscale DERP 中转服务器。这将显著提升 Tailscale 客户端之间的连接质量，尤其是当客户端无法建立直接连接时。 