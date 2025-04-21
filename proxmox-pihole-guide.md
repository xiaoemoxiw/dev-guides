# Proxmox VE上使用LXC容器安装Pi-hole指南

> 本指南将帮助您在Proxmox VE(PVE)上使用LXC容器安装Pi-hole，Pi-hole是一个网络级的广告和互联网跟踪拦截应用，能够为您的整个网络提供广告拦截功能。

## 为什么选择LXC容器

LXC (Linux Containers) 相比完整的虚拟机更加轻量级，对系统资源消耗更少，非常适合运行Pi-hole这类服务。Pi-hole本身资源需求较低，使用LXC容器是理想选择。

## 安装步骤

### 1. 创建LXC容器

1. 登录到Proxmox VE的Web界面
2. 点击右上角的"Create CT"(创建容器)按钮
3. 配置基本设置（主机名、密码等）
4. 为容器分配资源（磁盘、CPU、内存）
5. 配置网络设置（重要！）

### 2. 配置网络

这一步非常重要，因为Pi-hole需要一个固定的IP地址。

1. 在网络配置页面，选择"Static"（静态）而非DHCP
2. 填写IP/CIDR和网关信息：

![网络配置](/images/pve-pihole/network-config.png)

3. 确保设置了正确的网关地址

### 3. DNS设置

保持DNS设置为默认的"use host settings"：

![DNS设置](/images/pve-pihole/dns-settings.png)

### 4. 完成容器创建

1. 确认所有设置无误后，创建并启动容器
2. 启动容器后通过控制台或SSH连接

### 5. 安装Pi-hole

1. 更新系统：
```bash
apt update && apt upgrade -y
```

2. 安装必要依赖：
```bash
apt install curl -y
```

3. 下载并运行Pi-hole安装脚本：
```bash
curl -sSL https://install.pi-hole.net > install.sh
chmod +x install.sh
./install.sh
```

### 6. Pi-hole配置

在安装过程中，您需要做以下选择：

1. 选择上游DNS提供商（在中国环境推荐选择Custom并设置为114.114.114.114和119.29.29.29）：

![上游DNS选择](/images/pve-pihole/upstream-dns.png)

2. 选择拦截列表（推荐使用默认的StevenBlack列表）：

![拦截列表选择](/images/pve-pihole/blocklists.png)

3. 设置隐私级别（建议保持默认的"Show everything"）：

![隐私设置](/images/pve-pihole/privacy-settings.png)

4. 完成剩余步骤，包括启用Web界面和日志功能

### 7. 后续配置

1. 安装完成后记下管理员密码
2. 通过浏览器访问Pi-hole Web界面：`http://您的Pi-hole_IP/admin`
3. 在路由器中将DNS服务器设置为Pi-hole的IP地址，或在各设备上手动配置DNS
4. 可选：设置自定义过滤列表和白名单

## 故障排除

### SSH连接问题

如果您无法通过SSH连接到容器，请尝试：

1. 通过Proxmox控制台连接容器
2. 确认SSH服务已安装并运行：
```bash
apt install openssh-server -y
systemctl start ssh
systemctl enable ssh
```

### 安装脚本下载问题

在中国网络环境下，可能需要使用以下方式下载并运行安装脚本：

```bash
curl -sSL https://install.pi-hole.net > install.sh
chmod +x install.sh
PIHOLE_SKIP_OS_CHECK=true ./install.sh
```

或指定国内DNS：

```bash
./install.sh --unattended --dns-listen-all=yes --interface=eth0 --dns1=114.114.114.114 --dns2=119.29.29.29
```

## 参考资源

- [Pi-hole官方文档](https://docs.pi-hole.net/)
- [Pi-hole GitHub仓库](https://github.com/pi-hole/pi-hole) 